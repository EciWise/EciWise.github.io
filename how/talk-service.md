---
layout: default
title: Talk Service
---

# Talk Service

## Overview

`talk` is the real-time chat microservice for the ECIWise platform. It exposes a REST API for managing conversations and messages, and a WebSocket (STOMP) channel for live updates. The service integrates JWT authentication, PostgreSQL persistence, Redis for censored-word caching, Flyway for schema migrations, and MinIO / AWS S3 for file attachments.

The service handles six core domains:

- **Conversations**: create and manage individual and group chats; add or remove participants.
- **Messages**: send, edit, delete, pin, and reply to messages; attach images and documents.
- **Censorship**: automatically filter forbidden words before persistence; administrators manage the blocklist.
- **Reactions**: add and remove emoji reactions on any message.
- **Read receipts**: mark messages as read and broadcast the update over WebSocket.
- **Real-time**: push new messages, edits, reactions, and read events to all participants via STOMP.

---

## System Architecture

```mermaid
graph TB
    subgraph Clients["Clients"]
        Web["Browser / App"]
    end

    subgraph TalkService["Talk Service — Spring Boot / Java 21"]
        Security["Security Layer\nJwtAuthFilter · JwtValidator"]
        
        subgraph Controllers["Controllers"]
            CC["ConversationController\n/api/v1/conversations"]
            MC["MessageController\n/api/v1/conversations/{id}/messages"]
            CensC["CensorshipController\n/api/v1/censorship/words"]
            WsC["ChatWsController\nSTOMP /app/*"]
        end

        subgraph Services["Services"]
            ConvSvc["ConversationService"]
            MsgSvc["MessageService"]
            CensSvc["CensorshipService"]
            ReacSvc["ReactionService"]
            AttSvc["AttachmentStorageService"]
            Notifier["WebSocketNotifier\nSTOMP broker"]
        end
    end

    subgraph Storage["Storage & Cache"]
        PG[("PostgreSQL\nFlyway schema")]
        Redis[("Redis\ncensored words cache")]
        MinIO["MinIO / AWS S3\nfile attachments"]
    end

    Web -->|"REST — Bearer JWT"| Security
    Web -->|"WS — Bearer JWT (STOMP)"| Security
    Security --> Controllers
    Controllers --> Services
    Services --> PG
    CensSvc --> Redis
    AttSvc --> MinIO
    Services --> Notifier
    Notifier -->|"STOMP /topic/conversation/{id}"| Web
```

---

## Internal Component Diagram

```mermaid
graph LR
    subgraph Config["Configuration"]
        JwtProp["JwtProperties"]
        RedisCfg["RedisConfig"]
        S3Cfg["S3Config + S3Properties"]
        WsCfg["WebSocketConfig\nSTOMP endpoint"]
        SecCfg["SecurityConfig\nfilter chain"]
    end

    subgraph Domain["Domain (JPA entities)"]
        Conv["Conversation\n+ ConversationType\n+ Participant"]
        Msg["Message\n+ MessageAttachment\n+ MessageRead\n+ MessageReaction"]
        Word["CensoredWord"]
    end

    subgraph AppLayer["Application"]
        ConvSvc2["ConversationService"]
        MsgSvc2["MessageService"]
        CensSvc2["CensorshipService"]
        ReacSvc2["ReactionService"]
    end

    subgraph Infra["Infrastructure"]
        JwtFilt["JwtAuthFilter"]
        WsNotif["WebSocketNotifier"]
        AttStore["AttachmentStorageService\nMinIO / S3"]
        ErrHandler["GlobalExceptionHandler"]
    end

    Config --> AppLayer
    Domain --> AppLayer
    AppLayer --> Infra
```

---

## New Conversation Flow

```mermaid
sequenceDiagram
    participant U as User (Client)
    participant F as JwtAuthFilter
    participant C as ConversationController
    participant S as ConversationService
    participant P as PostgreSQL
    participant N as WebSocketNotifier
    participant O as Other Participant

    U->>F: POST /api/v1/conversations + Bearer JWT
    F->>F: validate token, build UserPrincipal
    F->>C: authenticated request
    C->>S: create(request, userPrincipal)
    S->>P: find existing conversation (idempotent for INDIVIDUAL)
    alt Already exists (INDIVIDUAL)
        P-->>S: existing conversation
    else New conversation
        S->>P: INSERT conversation
        S->>P: INSERT participants
    end
    S->>N: publish CONVERSATION_CREATED event
    N->>O: STOMP /topic/conversation/{id} — new conversation notification
    S-->>C: ConversationResponse
    C-->>U: 201 Created + ConversationResponse
```

---

## Message Send Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as MessageController
    participant CS as CensorshipService
    participant Redis as Redis Cache
    participant AS as AttachmentStorageService
    participant MinIO as MinIO / S3
    participant MS as MessageService
    participant P as PostgreSQL
    participant N as WebSocketNotifier
    participant L as All Participants

    U->>C: POST /messages (text) or /messages/with-attachment (multipart)
    C->>MS: send(conversationId, request, [file], userPrincipal)
    MS->>CS: applyCensorship(content)
    CS->>Redis: load active word list
    Redis-->>CS: censored words
    CS-->>MS: censored content + isCensored flag

    alt Has file attachment
        MS->>AS: upload(buffer, filename, contentType)
        AS->>MinIO: PUT object
        MinIO-->>AS: fileUrl
        AS-->>MS: fileUrl
    end

    MS->>P: INSERT message + attachment (if any)
    MS->>N: publish NEW_MESSAGE event
    N->>L: STOMP /topic/conversation/{id}  type=NEW_MESSAGE
    MS-->>C: MessageResponse
    C-->>U: 201 Created + MessageResponse
```

---

## Read Receipt & Reaction Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as Controller
    participant S as Service
    participant P as PostgreSQL
    participant N as WebSocketNotifier
    participant L as Participants

    U->>C: POST /messages/read  { messageIds: [...] }
    C->>S: markAsRead(conversationId, messageIds, userPrincipal)
    S->>P: UPSERT message_reads (idempotent)
    S->>N: emit MESSAGE_READ
    N->>L: STOMP /topic/conversation/{id}  type=MESSAGE_READ

    U->>C: POST /messages/{id}/reactions  { emoji: "👍" }
    C->>S: reactionService.add(…)
    S->>P: INSERT message_reactions
    S->>N: emit REACTION_ADDED
    N->>L: STOMP /topic/conversation/{id}  type=REACTION_ADDED
```

---

## Censorship Flow

```mermaid
flowchart TD
    Upload["Message sent by user"]
    Check["CensorshipService\ncheck content against word list"]
    Redis2["Redis\n(active words, loaded at startup\ninvalidated on admin update)"]
    Match{"Forbidden word\nfound?"}
    Replace["Replace matched words with ***\nset isCensored = true"]
    Persist["Persist message to PostgreSQL"]
    Broadcast["Broadcast via WebSocketNotifier"]
    AdminCensor["PATCH /messages/{id}/censor\n(admin manual override)"]
    SetFlag["Set isCensored = true\nbroadcast MESSAGE_UPDATED"]

    Upload --> Check
    Check <--> Redis2
    Check --> Match
    Match -->|yes| Replace
    Match -->|no| Persist
    Replace --> Persist
    Persist --> Broadcast
    AdminCensor --> SetFlag
```

---

## WebSocket Event Map

```mermaid
graph LR
    subgraph ClientToServer["Client → Server (STOMP /app/*)"]
        TypingIn["/app/typing\n{conversationId}"]
        SendIn["/app/send\n{conversationId, content}"]
    end

    subgraph ServerToClient["Server → Client (/topic/conversation/{id})"]
        NewMsg["NEW_MESSAGE"]
        Updated["MESSAGE_UPDATED"]
        Deleted["MESSAGE_DELETED"]
        Read["MESSAGE_READ"]
        ReactionAdd["REACTION_ADDED"]
        ReactionRem["REACTION_REMOVED"]
        Typing["TYPING"]
        ParticipantAdd["PARTICIPANT_ADDED"]
        ParticipantRem["PARTICIPANT_REMOVED"]
    end

    TypingIn -->|broadcast| Typing
    SendIn -->|persist + broadcast| NewMsg
```

---

## Data Model

```mermaid
erDiagram
    conversations {
        uuid id PK
        string type
        string name
        timestamp created_at
    }

    participants {
        uuid conversation_id FK
        uuid user_id
        timestamp joined_at
    }

    messages {
        uuid id PK
        uuid conversation_id FK
        uuid sender_id
        text content
        uuid reply_to_id FK
        boolean is_pinned
        boolean is_censored
        boolean is_deleted
        timestamp created_at
        timestamp updated_at
    }

    message_attachments {
        uuid id PK
        uuid message_id FK
        string url
        string content_type
        string filename
    }

    message_reads {
        uuid message_id FK
        uuid user_id
        timestamp read_at
    }

    message_reactions {
        uuid id PK
        uuid message_id FK
        uuid user_id
        string emoji
    }

    censored_words {
        uuid id PK
        string word
        uuid added_by
        boolean active
    }

    conversations ||--o{ participants : "has"
    conversations ||--o{ messages : "contains"
    messages ||--o{ message_attachments : "has"
    messages ||--o{ message_reads : "tracked by"
    messages ||--o{ message_reactions : "receives"
    messages }o--o| messages : "reply_to"
```

---

## Endpoints

### Conversations (`/api/v1/conversations`)

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/conversations` | JWT | Create an individual or group conversation |
| `GET` | `/api/v1/conversations` | JWT | List conversations for the authenticated user |
| `GET` | `/api/v1/conversations/{id}` | JWT | Get conversation details |
| `PUT` | `/api/v1/conversations/{id}` | JWT | Update conversation metadata |
| `DELETE` | `/api/v1/conversations/{id}` | JWT | Delete a conversation |
| `POST` | `/api/v1/conversations/{id}/participants` | JWT | Add a participant |
| `DELETE` | `/api/v1/conversations/{id}/participants/{userId}` | JWT | Remove a participant |

### Messages (`/api/v1/conversations/{conversationId}/messages`)

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/messages` | JWT | List messages paginated (50 per page, asc by `createdAt`) |
| `POST` | `/messages` | JWT | Send a text message |
| `POST` | `/messages/with-attachment` | JWT | Send a message with file attachment (multipart) |
| `PUT` | `/messages/{messageId}` | JWT | Edit message text |
| `DELETE` | `/messages/{messageId}` | JWT | Soft-delete a message |
| `PATCH` | `/messages/{messageId}/censor` | JWT (admin) | Manually censor a message |
| `POST` | `/messages/read` | JWT | Mark a list of message IDs as read |
| `POST` | `/messages/{messageId}/reactions` | JWT | Add an emoji reaction |
| `DELETE` | `/messages/{messageId}/reactions/{emoji}` | JWT | Remove an emoji reaction |
| `PATCH` | `/messages/{messageId}/pin` | JWT | Toggle pin state |
| `GET` | `/messages/pinned` | JWT | List pinned messages |

### Censorship (`/api/v1/censorship/words`)

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/v1/censorship/words` | JWT (admin) | List all censored words |
| `POST` | `/api/v1/censorship/words` | JWT (admin) | Add a forbidden word |
| `DELETE` | `/api/v1/censorship/words/{id}` | JWT (admin) | Deactivate a forbidden word |

---

## Deployment

### Configuration

| Variable | Purpose |
|---|---|
| `SERVER_PORT` | HTTP port (default `3003`) |
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC URL |
| `SPRING_DATASOURCE_USERNAME` | Database user |
| `SPRING_DATASOURCE_PASSWORD` | Database password |
| `SPRING_REDIS_HOST` | Redis host |
| `SPRING_REDIS_PORT` | Redis port (default `6379`) |
| `JWT_SECRET` | HS256 shared secret |
| `MINIO_ENDPOINT` | MinIO or S3-compatible endpoint |
| `MINIO_ACCESS_KEY` | Object storage access key |
| `MINIO_SECRET_KEY` | Object storage secret key |
| `MINIO_BUCKET` | Target bucket name |

### Local Execution

```bash
docker compose up -d   # start PostgreSQL, Redis, MinIO
mvn spring-boot:run    # Flyway migrations run automatically on startup
```

---

## Further Reading

- Source repository: [EciWise/talk](https://github.com/EciWise/talk)
- C4 diagrams: `talk/docs/diagramas/`
- Database schema: Flyway migrations in `src/main/resources/db/migration/`
