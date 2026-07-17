---
layout: default
title: Materials Service
---

# Materials Service

## Overview

`materials` (Wise Banco Material) is the collaborative academic repository microservice for ECIWise. Users upload, search, and rate PDF study materials organized by subject, semester, and topic. The service orchestrates AI-based content validation, cloud storage, and email notifications triggered by a message bus.

The service handles three core domains:

- **Material lifecycle**: upload PDFs with deduplication (SHA-256 hash check), AI validation, tag assignment, versioning, and soft deletion that also removes the cloud blob.
- **Discovery and statistics**: full-text search by name, multi-filter browsing (subject, semester, tags), and aggregated statistics (popular materials, downloads, view counts, rating averages).
- **Ratings**: 1–5 star ratings with comments; per-material and per-user aggregations.

---

## C4 — Level 1: System Context

```mermaid
graph TB
    subgraph Actors["Actors"]
        EST["Estudiante\nuploads · searches · rates\ndownloads PDFs"]
        TUT["Tutor\npublishes material"]
        ADM["Administrador\ntoggles AI validation\nexports statistics"]
    end

    MAT["Materials Service\nWise Banco Material\nNestJS 11 · Node"]

    subgraph Ecosystem["ECIWISE+ Ecosystem"]
        FE["ECIWISE+ Frontend\nAngular SPA"]
        AUTH["Auth Service\nissues the HS256 JWT"]
        AISVC["AI Validation Service\nclassifies · approves PDFs"]
        NOTIF["Notifications Service\nsends the emails"]
        GAMIF["Gamification Service\nawards points"]
    end

    subgraph External["External / Infrastructure"]
        STORE["Object Storage\nAzure Blob or AWS S3"]
        BUS["Message Bus\nAzure Service Bus or RabbitMQ"]
        PG[("PostgreSQL\nmateriales · tags · calificaciones")]
    end

    EST --> FE
    TUT --> FE
    ADM --> FE

    FE -->|"REST + Bearer JWT"| MAT
    AUTH -.->|"shared JWT_SECRET\nvalidated locally — no HTTP call"| MAT

    MAT -->|"upload · download · delete blobs"| STORE
    MAT -->|"Prisma"| PG
    MAT -->|"material.process (request)"| BUS
    BUS --> AISVC
    AISVC -->|"material.responses (reply)"| BUS
    BUS -->|"correlated reply"| MAT
    MAT -->|"mail.envio.individual"| BUS
    BUS --> NOTIF
    MAT -->|"eciwise.events / material.aprobado"| BUS
    BUS --> GAMIF
```

### Actors

| Actor | Interaction |
|---|---|
| Estudiante | Uploads PDFs, searches and filters, rates 1–5 with comments, downloads |
| Tutor | Publishes material for their subjects; same surface as a student |
| Administrador | Toggles AI validation at runtime, exports statistics to PDF |

### Neighbouring systems

| System | Relationship |
|---|---|
| Auth Service | Issues the JWT; Materials validates it **offline** with the shared secret |
| AI Validation Service | Request–reply over the bus: approves or rejects each upload |
| Notifications Service | Receives `mail.envio.individual` and sends the confirmation email |
| Gamification Service | Subscribes to `material.aprobado` to award points to the author |
| Object Storage / Message Bus | Chosen at boot by env var — see [Provider Selection](#provider-selection) |

---

## C4 — Level 2: Containers

Materials is a **single NestJS process** whose storage and messaging are pluggable.

```mermaid
graph TB
    FE["ECIWISE+ Frontend\nAngular SPA"]

    subgraph MatSystem["Materials Service"]
        API["REST API\nNestJS 11 · Express\nhelmet · CORS · ThrottlerGuard\nglobal JwtAuthGuard · Swagger /api"]
        STOREAD["Storage Adapter\nAzureBlobAdapter or S3Adapter\nselected by STORAGE_PROVIDER"]
        BUSAD["Message Bus Adapter\nAzureServiceBusAdapter or RabbitMQAdapter\nselected by MESSAGE_BUS_PROVIDER"]
    end

    subgraph Infra["Infrastructure"]
        STORE["Object Storage\nAzure Blob · AWS S3"]
        BUS["Message Bus\nAzure Service Bus · RabbitMQ"]
        PG[("PostgreSQL\nPrisma 6 + @prisma/adapter-pg")]
    end

    AISVC["AI Validation Service"]
    NOTIF["Notifications Service"]
    GAMIF["Gamification Service"]

    FE -->|"HTTPS / REST\nBearer JWT"| API
    API -->|"StoragePort"| STOREAD
    API -->|"MessageBusPort"| BUSAD
    API -->|"reads / writes"| PG
    STOREAD --> STORE
    BUSAD --> BUS
    BUS <--> AISVC
    BUS --> NOTIF
    BUS --> GAMIF
```

| Container | Technology | Responsibility |
|---|---|---|
| REST API | NestJS 11, Express | 22 material endpoints + PDF export, guards, validation, Swagger |
| Storage adapter | `@azure/storage-blob` / `@aws-sdk/client-s3` | Upload, download, delete, exists |
| Message bus adapter | `@azure/service-bus` / `amqplib` | `send` (point-to-point) and `publish` (topic) |
| Database | PostgreSQL + Prisma | Materials, tags, ratings, mirrored users |
| Object storage | Azure Blob / S3 | The PDF blobs themselves — never stored in the database |

Rate limiting is applied globally by `ThrottlerGuard` in three tiers: **5 req/s**, **20 req/10 s**, and **60 req/min**.

---

## C4 — Level 3: Components

```mermaid
graph TB
    subgraph AppModule["AppModule"]
        THROTTLE["ThrottlerGuard\nAPP_GUARD · 3 tiers"]
        JWTG["JwtAuthGuard\nAPP_GUARD · global"]
    end

    subgraph AuthMod["AuthModule"]
        GUARD["JwtGuard\nHS256 · @nestjs/jwt"]
        CU["@CurrentUser()\ndecorator"]
    end

    subgraph MaterialMod["MaterialModule"]
        MC["MaterialController\n22 REST endpoints"]
        MS["MaterialService\nupload · dedup · search\nratings · statistics"]
        IAV["IaValidationService\nruntime on/off switch"]
    end

    subgraph PdfMod["PdfExportModule"]
        PC["PdfExportController"]
        PS["PdfExportService\npdfkit"]
    end

    subgraph CommonMod["CommonModule — adapter factory"]
        SPORT["STORAGE_PORT\n«interface»"]
        MPORT["MESSAGE_BUS_PORT\n«interface»"]
    end

    subgraph PrismaMod["PrismaModule"]
        PSVC["PrismaService"]
    end

    JWTG --> GUARD
    MC --> MS
    MC --> IAV
    MC --> CU
    MS --> SPORT
    MS --> MPORT
    MS --> PSVC
    MS -.->|"asks before sending to AI"| IAV
    PC --> PS
    PS --> PSVC
```

| Component | Role |
|---|---|
| `MaterialController` | 22 endpoints: upload, search, filter, rate, download, preview, statistics |
| `MaterialService` | The domain core — hashing/dedup, AI request–reply, persistence, events |
| `IaValidationService` | Hot switch for AI validation (`PATCH /material/ia-validation`), no restart |
| `JwtAuthGuard` | Registered as `APP_GUARD`, so every route is authenticated by default |
| `CommonModule` | Factory that reads `STORAGE_PROVIDER` / `MESSAGE_BUS_PROVIDER` and binds the ports |
| `PdfExportService` | Renders the statistics dashboards to PDF with `pdfkit` |

> The AI switch **only** gates the `material.process` request. Publishing `material.aprobado` to gamification is independent and is never affected by the flag.

---

## C4 — Level 4: Code

The hexagonal core: `MaterialService` depends on two interfaces and never imports a cloud SDK. `CommonModule` decides which implementation is bound at boot, so swapping Azure for AWS is an environment change, not a code change.

```mermaid
classDiagram
    class MaterialService {
        -StoragePort storage
        -MessageBusPort messageBus
        -PrismaService prisma
        -IaValidationService iaValidation
        -Map pendingRequests
        +onModuleInit() void
        +validateMaterial(file, dto, userId)
        +guardarMaterial(dto)
        +guardarTags(materialId, tags)
        +enviarNotificacion(response, userId, filename, template)
        +rateMaterial(materialId, userId, dto)
        +downloadMaterial(materialId)
        +searchMaterialsByName(name)
        -calculateHashAndExtension(file)
        -listenForResponses() void
        -waitForResponse(correlationId) Promise~RespuestaIADto~
        -emitirMaterialAprobado(userId, materialId)
    }

    class StoragePort {
        <<interface>>
        +upload(buffer, name, contentType) Promise~string~
        +download(fileUrl) Promise~DownloadResult~
        +delete(fileUrl) Promise~boolean~
        +exists(fileUrl) Promise~boolean~
    }

    class MessageBusPort {
        <<interface>>
        +send(queue, body, options) Promise~void~
        +publish(exchange, routingKey, body, options) Promise~void~
        +subscribe(queue, onMessage, onError) void
        +close() Promise~void~
    }

    class BaseBusService {
        <<abstract>>
        #Logger logger
    }

    class AzureBlobAdapter {
        +upload() Promise~string~
    }
    class S3Adapter {
        +upload() Promise~string~
    }
    class AzureServiceBusAdapter
    class RabbitMQAdapter {
        +onModuleInit() Promise~void~
        +onModuleDestroy() Promise~void~
    }

    class IaValidationService {
        -boolean enabled
        +isEnabled() boolean
        +setEnabled(value) boolean
    }

    MaterialService ..> StoragePort : STORAGE_PORT token
    MaterialService ..> MessageBusPort : MESSAGE_BUS_PORT token
    MaterialService --> IaValidationService

    StoragePort <|.. AzureBlobAdapter
    StoragePort <|.. S3Adapter
    MessageBusPort <|.. BaseBusService
    BaseBusService <|-- AzureServiceBusAdapter
    BaseBusService <|-- RabbitMQAdapter
```

### Ports and their adapters

| Port | Token | Adapters | Selected by |
|---|---|---|---|
| `StoragePort` | `STORAGE_PORT` | `AzureBlobAdapter`, `S3Adapter` | `STORAGE_PROVIDER` (`azure` \| `s3`) |
| `MessageBusPort` | `MESSAGE_BUS_PORT` | `AzureServiceBusAdapter`, `RabbitMQAdapter` | `MESSAGE_BUS_PROVIDER` (`azure` \| `rabbitmq`) |

### Message contracts

| Destination | Kind | Direction | Purpose |
|---|---|---|---|
| `material.process` | queue (`send`) | out | Ask the AI to validate an uploaded PDF (carries `correlationId`) |
| `material.responses` | queue (`subscribe`) | in | AI verdict, matched back by `correlationId` |
| `mail.envio.individual` | queue (`send`) | out | Confirmation email for the author |
| `eciwise.events` / `material.aprobado` | topic (`publish`) | out | Domain event so gamification awards points |

---

## Provider Selection

```mermaid
flowchart LR
    subgraph ENV["Environment Variables"]
        SP["STORAGE_PROVIDER\nazure | s3"]
        MP["MESSAGE_BUS_PROVIDER\nazure | rabbitmq"]
    end

    subgraph Storage["Storage"]
        SP -->|azure| AZ["Azure Blob Storage"]
        SP -->|s3| S3b["AWS S3"]
    end

    subgraph Bus["Messaging"]
        MP -->|azure| SB["Azure Service Bus"]
        MP -->|rabbitmq| RMQb["RabbitMQ"]
    end

    note["Switching providers requires\nonly .env change + restart.\nNo code change, no recompilation."]
```

---

## Material Upload Flow

```mermaid
sequenceDiagram
    actor U as User
    participant C as MaterialController
    participant S as MaterialService
    participant ST as StoragePort
    participant MB as MessageBusPort
    participant IA as AI Service (external)
    participant DB as PostgreSQL

    U->>C: POST /material (PDF buffer + metadata)
    C->>C: validate MIME type = application/pdf
    C->>S: validateMaterial(buffer, dto, filename)

    S->>S: SHA-256(buffer) → hash
    S->>DB: SELECT WHERE hash = ?
    DB-->>S: null (no duplicate)

    S->>ST: upload(buffer, filename, contentType)
    ST-->>S: fileUrl

    S->>MB: send("material.process", { fileUrl, filename }, correlationId)
    MB-->>IA: message enqueued

    IA-->>MB: response on "material.responses"
    MB-->>S: onMessage(body, correlationId)

    alt PDF valid
        S->>DB: INSERT Materiales + Tags + Resumen
        S->>MB: send("mail.envio.individual", { email, template })
        S-->>C: CreateMaterialResponseDto
        C-->>U: 201 Created
    else PDF invalid
        S->>ST: delete(fileUrl)
        S-->>C: UnprocessableEntityException
        C-->>U: 422 Unprocessable Entity
    end
```

---

## Deduplication Flow

```mermaid
flowchart TD
    Upload2["POST /material (PDF buffer)"]
    Hash["Compute SHA-256(buffer)"]
    Lookup["SELECT * FROM Materiales WHERE hash = ?"]
    Found{"Duplicate\nfound?"}
    Reject["409 Conflict\nDuplicate material"]
    Continue["Continue upload flow"]

    Upload2 --> Hash --> Lookup --> Found
    Found -->|yes| Reject
    Found -->|no| Continue
```

---

## Package Structure

```mermaid
graph TD
    AppModule2["AppModule"]
    MaterialModule["MaterialModule"]
    CommonModule2["CommonModule\nadapter factory"]
    PrismaModule2["PrismaModule"]
    PdfModule["PdfExportModule"]
    AuthMod["AuthModule\nJwtGuard"]
    Config2["config/env.ts\nJoi validation"]

    AppModule2 --> MaterialModule
    AppModule2 --> CommonModule2
    AppModule2 --> PrismaModule2
    AppModule2 --> PdfModule
    AppModule2 --> AuthMod
    AppModule2 --> Config2

    MaterialModule --> MC2["material.controller.ts\n22 endpoints"]
    MaterialModule --> MS2["material.service.ts\nbusiness logic"]
    MaterialModule --> DTO2["dto/\nrequest + response DTOs"]
    MaterialModule --> Entities["entities/\ndomain entities"]

    CommonModule2 --> Ports2["ports/\nStoragePort\nMessageBusPort"]
    CommonModule2 --> Adapters["adapters/\nazure-blob  s3\nazure-service-bus  rabbitmq"]
```

---

## Data Model

```mermaid
erDiagram
    usuarios {
        string id PK
        string email
        string nombre
        string apellido
        string telefono
        string biografia
        int semestre
        string google_id
        string avatar_url
        int rol_id
        int estado_id
        datetime ultimo_login
        datetime created_at
        datetime updated_at
    }

    Materiales {
        string id PK
        string nombre
        string userId FK
        string url
        string extension
        string descripcion
        int vistos
        int descargas
        string hash
        datetime createdAt
        datetime updatedAt
    }

    Tags {
        int id PK
        string tag
    }

    MaterialTags {
        int idTag FK
        string idMaterial FK
    }

    Calificaciones {
        int id PK
        string idMaterial FK
        string userId FK
        int calificacion
        string comentario
        datetime createdAt
    }

    Resumen {
        int id PK
        string idMaterial FK
        string resumen
    }

    usuarios ||--o{ Materiales : "uploads"
    usuarios ||--o{ Calificaciones : "rates"
    Materiales ||--o{ MaterialTags : "has"
    Tags ||--o{ MaterialTags : "groups"
    Materiales ||--o{ Calificaciones : "receives"
    Materiales ||--o{ Resumen : "generates"
```

---

## CI/CD Pipeline

```mermaid
flowchart TD
    Push["Push / PR"]
    Branch{"Branch"}
    Build["build.yml\nSonarQube + coverage"]
    Publish["publish.yml\nDocker → GHCR"]

    Push --> Branch
    Branch -->|develop or PR| Build
    Branch -->|main or develop| Publish

    Build --> B1["Checkout"]
    B1 --> B2["Setup Node 20"]
    B2 --> B3["npm ci"]
    B3 --> B4["npm run test:cov\n144 tests"]
    B4 --> B5["SonarQube Scan\nwith coverage/lcov.info"]

    Publish --> D1["Checkout"]
    D1 --> D2["Docker meta\nsha + branch tags"]
    D2 --> D3["Login GHCR"]
    D3 --> D4["Build & Push\nghcr.io/dosw2025/wise_banco_material"]
```

---

## Endpoints

### Materials (`/material`)

| Method | Path | Description |
|---|---|---|
| `POST` | `/material` | Upload a new PDF — AI validation, deduplication |
| `GET` | `/material` | List all materials (paginated) |
| `GET` | `/material/search` | Search by name |
| `GET` | `/material/filter` | Advanced filter (subject, semester, tags, sort) |
| `GET` | `/material/sorted/by-date` | List sorted by upload date |
| `GET` | `/material/stats/popular` | Top most-viewed materials |
| `GET` | `/material/stats/count` | Total material count |
| `GET` | `/material/stats/tags-percentage` | Global tag distribution |
| `GET` | `/material/:id` | Material detail |
| `GET` | `/material/:id/download` | Download PDF stream |
| `PUT` | `/material/:id` | Update metadata or replace file |
| `DELETE` | `/material/:id` | Delete material and remove cloud blob |
| `POST` | `/material/:id/ratings` | Rate a material (1–5 stars) |
| `GET` | `/material/:id/ratings` | Average rating and total count |
| `GET` | `/material/:id/ratings/list` | Full list of ratings with comments |
| `GET` | `/material/user/:userId` | All materials by a user with stats |
| `GET` | `/material/user/:userId/stats` | Aggregated upload statistics |
| `GET` | `/material/user/:userId/top-downloaded` | User's top 3 most downloaded |
| `GET` | `/material/user/:userId/top-viewed` | User's top 3 most viewed |
| `GET` | `/material/user/:userId/top` | All user materials sorted by popularity |
| `GET` | `/material/user/:userId/average-rating` | User's average rating |
| `GET` | `/material/user/:userId/tags-percentage` | User's tag distribution |

### PDF Export (`/pdf-export`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/pdf-export/:id/stats/export` | Export material statistics as a PDF |

---

## Deployment

### Environment Variables

```env
NODE_ENV=development
PORT=3000
SWAGGER_ENABLED=true

STORAGE_PROVIDER=azure          # azure | s3
BLOB_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;AccountName=...
BLOB_STORAGE_ACCOUNT_NAME=your-account-name

# AWS S3 (only if STORAGE_PROVIDER=s3)
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1
S3_BUCKET_NAME=your-bucket-name

MESSAGE_BUS_PROVIDER=azure      # azure | rabbitmq
SERVICE_BUS_CONNECTION_STRING=Endpoint=sb://your-namespace...

# RabbitMQ (only if MESSAGE_BUS_PROVIDER=rabbitmq)
RABBITMQ_URL=amqp://user:password@localhost:5672

DATABASE_URL=postgresql://user:pass@host:5432/db?schema=public
DIRECT_URL=postgresql://user:pass@host:5432/db?schema=public
```

### Local Execution

```bash
cp .env.template .env
npm install
npx prisma generate
npm run start:dev
```

### API Documentation

Swagger is available at `http://localhost:3000/api` when `SWAGGER_ENABLED=true`.

---

## Further Reading

- Source repository: [EciWise/materials](https://github.com/EciWise/materials)
- Port interfaces: `src/common/ports/`
- Prisma schema: `prisma/schema.prisma`
- Swagger: `/api` at runtime
