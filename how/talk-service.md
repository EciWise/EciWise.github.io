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

## C4 — Level 1: System Context

```mermaid
graph TB
    subgraph Actors["Actors"]
        EST["Estudiante\nchats · reacts · attaches files"]
        TUT["Tutor\nsame chat surface"]
        ADM["Administrador\nmanages the censored-word blocklist"]
    end

    TALK["Talk Service\nReal-time chat for ECIWISE+\nSpring Boot · Java 21"]

    subgraph Ecosystem["ECIWISE+ Ecosystem"]
        FE["ECIWISE+ Frontend\nAngular SPA · @stomp/stompjs"]
        AUTH["Auth Service\nissues the HS256 JWT"]
    end

    subgraph External["External / Infrastructure"]
        PG[("PostgreSQL\nconversations · messages\nreactions · reads · censored words")]
        REDIS[("Redis\ncensored-word cache\ntalk:censored_words · TTL 1 h")]
        S3["MinIO / AWS S3\nmessage attachments"]
    end

    EST --> FE
    TUT --> FE
    ADM --> FE

    FE -->|"REST + Bearer JWT"| TALK
    FE <-->|"WebSocket STOMP /ws/chat\nJWT on CONNECT"| TALK
    AUTH -.->|"shared JWT secret\nvalidated locally — no HTTP call"| TALK

    TALK -->|"JPA / Flyway"| PG
    TALK -->|"blocklist cache"| REDIS
    TALK -->|"upload · presigned GET"| S3
```

### Actors

| Actor | Interaction |
|---|---|
| Estudiante / Tutor | Creates conversations, sends/edits/pins messages, reacts, reads, attaches files |
| Administrador | Manages the censored-word blocklist via `/api/v1/censorship/words` |

### Neighbouring systems

| System | Relationship |
|---|---|
| Auth Service | Issues the JWT. Talk validates it locally with `jjwt` — no runtime call to Auth |
| Frontend | Talks REST for state and STOMP for live events; both authenticate with the same Bearer token |
| MinIO / AWS S3 | Holds attachment blobs; Talk stores only the `storageKey` and serves presigned URLs |
| Redis | Caches the active blocklist so censorship does not hit PostgreSQL per message |

Talk is **self-contained**: it publishes no events and calls no other microservice.

---

## C4 — Level 2: Containers

```mermaid
graph TB
    FE["ECIWISE+ Frontend\nAngular SPA"]

    subgraph TalkSystem["Talk Service — single Spring Boot process"]
        REST["REST API\nSpring Web · Java 21\nstateless · JwtAuthFilter\n/api/v1/**"]
        WS["WebSocket / STOMP\nspring-boot-starter-websocket\nendpoint /ws/chat\nin-memory simple broker"]
        JPA["Persistence\nSpring Data JPA · Hibernate\nFlyway migrations"]
        STORE["Attachment Storage\nAWS SDK v2 (S3 API)"]
    end

    subgraph Infra["Infrastructure"]
        PG[("PostgreSQL")]
        REDIS[("Redis")]
        S3["MinIO / AWS S3"]
    end

    FE -->|"HTTPS REST\nBearer JWT"| REST
    FE <-->|"ws:// STOMP\nBearer JWT on CONNECT"| WS
    REST --> JPA
    WS --> JPA
    REST --> STORE
    JPA --> PG
    REST -->|"blocklist cache"| REDIS
    STORE --> S3
```

| Container | Technology | Responsibility |
|---|---|---|
| REST API | Spring Web, Java 21 | Conversations, messages, reactions, read receipts, censorship admin |
| WebSocket / STOMP | Spring WebSocket | Live fan-out of messages, edits, reactions, typing and read events |
| Persistence | Spring Data JPA + Flyway | Entities and schema migrations |
| Attachment storage | AWS SDK v2 | Uploads, presigned GET URLs, deletion |
| PostgreSQL / Redis / S3 | — | Durable state, blocklist cache, attachment blobs |

The REST API and the STOMP broker are **logical containers inside one JVM** — they are not separate deployments. The broker is Spring's in-memory `SimpleBroker`, so horizontal scaling would require an external relay (RabbitMQ/ActiveMQ).

---

## C4 — Level 3: Components

```mermaid
graph TB
    subgraph Security["Security"]
        FILTER["JwtAuthFilter\nOncePerRequestFilter"]
        VALID["JwtValidator\njjwt · HS256"]
        PRINC["UserPrincipal\nid · email · rol"]
        SECCFG["SecurityConfig\nstateless · csrf off\n/actuator/health · /ws/** permitAll\nanyRequest authenticated"]
    end

    subgraph Controllers["Controllers"]
        CC["ConversationController\n/api/v1/conversations"]
        MC["MessageController\n/api/v1/conversations/{id}/messages"]
        CENSC["CensorshipController\n/api/v1/censorship/words"]
        WSC["ChatWsController\n@MessageMapping /app/*"]
    end

    subgraph Services["Services"]
        CONVS["ConversationService"]
        MSGS["MessageService\nsend · edit · delete · pin"]
        CENSS["CensorshipService\nleet-aware blocklist"]
        REACS["ReactionService"]
        ATTS["AttachmentStorageService\nvalidate · upload · presign"]
        NOTIF["WebSocketNotifier\nSimpMessagingTemplate"]
    end

    subgraph Repos["Repositories (Spring Data)"]
        R1["ConversationRepository\nParticipantRepository"]
        R2["MessageRepository\nMessageAttachmentRepository\nMessageReactionRepository\nMessageReadRepository"]
        R3["CensoredWordRepository"]
    end

    subgraph Config["Configuration"]
        WSCFG["WebSocketConfig\nSTOMP + CONNECT interceptor"]
        REDISCFG["RedisConfig"]
        S3CFG["S3Config · S3Properties"]
        JWTP["JwtProperties"]
        ERR["GlobalExceptionHandler\n@RestControllerAdvice"]
    end

    FILTER --> VALID
    VALID --> PRINC
    SECCFG --> FILTER
    FILTER --> Controllers
    WSCFG -->|"validates JWT on CONNECT"| WSC

    CC --> CONVS
    MC --> MSGS
    MC --> REACS
    CENSC --> CENSS
    WSC --> MSGS

    MSGS --> CENSS
    MSGS --> ATTS
    MSGS --> NOTIF
    CONVS --> NOTIF
    REACS --> NOTIF

    CONVS --> R1
    MSGS --> R2
    CENSS --> R3
    Controllers -.->|"errors"| ERR
```

| Component | Role |
|---|---|
| `JwtAuthFilter` / `JwtValidator` | Validate the Bearer token per request and build a `UserPrincipal` |
| `WebSocketConfig` | Registers `/ws/chat` and intercepts STOMP `CONNECT` to authenticate the socket |
| `MessageService` | Send/edit/delete/pin; runs censorship, stores attachments, then notifies |
| `CensorshipService` | Leet-aware regex blocklist, cached in Redis for 1 h |
| `AttachmentStorageService` | Validates and uploads files, returns presigned GET URLs |
| `WebSocketNotifier` | The only component that touches `SimpMessagingTemplate` |

Authentication happens **twice by different paths**: `JwtAuthFilter` guards REST, while the STOMP `CONNECT` interceptor guards the socket. `/ws/**` is `permitAll` in the HTTP chain precisely because the handshake is authenticated at the STOMP layer instead.

---

## C4 — Level 4: Code

```mermaid
classDiagram
    class MessageService {
        -MessageRepository messageRepo
        -ConversationRepository conversationRepo
        -CensorshipService censorship
        -AttachmentStorageService attachments
        -WebSocketNotifier notifier
        +send(conversationId, req, sender) MessageResponse
        +send(conversationId, req, file, sender) MessageResponse
        +list(conversationId, user, pageable) Page~MessageResponse~
        +getPinned(conversationId, user) List~MessageResponse~
        +edit(conversationId, messageId, req, user) MessageResponse
    }

    class CensorshipService {
        -CensoredWordRepository wordRepo
        -RedisTemplate redisTemplate
        +censor(content) CensorResult
        +listAll() List~CensoredWordResponse~
        +addWord(word, admin) CensoredWordResponse
        +deactivateWord(wordId) void
        -loadActiveWords() Set~String~
        -invalidateCache() void
        -buildLeetPattern(word) Pattern
    }

    class AttachmentStorageService {
        +upload(conversationId, file) StoredFile
        +presignedGetUrl(storageKey) String
        +delete(storageKey) void
        +isImage(mimeType) boolean
        -validate(file) void
        -sanitizeFileName(raw) String
    }

    class WebSocketNotifier {
        -SimpMessagingTemplate template
        +broadcastMessage(conversationId, event) void
        +broadcastTyping(conversationId, event) void
        +notifyUser(userId, payload) void
        +notifyNewConversation(targetUserId, conversation) void
    }

    class JwtValidator {
        -JwtProperties jwtProperties
        +validate(token) UserPrincipal
        -signingKey() SecretKey
    }

    class UserPrincipal {
        <<record>>
        +String id
        +String email
        +String rol
    }

    class Conversation {
        <<entity>>
        +UUID id
        +ConversationType type
        +String name
        +List~Participant~ participants
    }

    class Message {
        <<entity>>
        +UUID id
        +UUID conversationId
        +String senderId
        +String content
        +boolean pinned
        +boolean deleted
        +Message replyTo
    }

    class MessageAttachment {
        <<entity>>
        +String storageKey
        +String mimeType
        +long sizeBytes
    }

    class CensoredWord {
        <<entity>>
        +UUID id
        +String word
        +boolean active
    }

    MessageService --> CensorshipService
    MessageService --> AttachmentStorageService
    MessageService --> WebSocketNotifier
    MessageService ..> Message
    JwtValidator ..> UserPrincipal
    Conversation "1" --> "*" Message
    Message "1" --> "*" MessageAttachment
    CensorshipService ..> CensoredWord
```

### Censorship — why the regex is unusual

`CensorshipService` does not do a plain `contains`. Each blocked word is compiled into a **leet-speak-tolerant pattern**, so `tonto` also matches `T0nt0` or `t0NT@`:

| Rule | Detail |
|---|---|
| Character classes | `a→[a4@]`, `e→[e3]`, `i→[i1!\|]`, `o→[o0]`, `s→[s5$]`, `t→[t7]`, … |
| Length-preserving | Every substitution is 1↔1, so match offsets survive `replaceAll()` |
| Custom boundaries | `(?<![\p{L}\d])` / `(?![\p{L}\d])` instead of `\b` — a native `\b` breaks on the letter↔digit transition inside `t0nt0` |
| Case | Handled by `CASE_INSENSITIVE`, not by the classes |
| Cache | Active words in Redis under `talk:censored_words`, TTL 1 h, invalidated on write |

Matches are replaced with `*****` **before persistence** — the raw text never reaches the database.

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
