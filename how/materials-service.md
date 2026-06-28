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

## Hexagonal Architecture

```mermaid
graph TB
    subgraph Domain["Domain (core)"]
        MS["MaterialService\nbusiness logic"]
    end

    subgraph Ports["Ports (interfaces)"]
        SP["StoragePort\n«interface»"]
        MBP["MessageBusPort\n«interface»"]
    end

    subgraph StorageAdapters["Storage Adapters"]
        AB["AzureBlobAdapter\n@azure/storage-blob"]
        S3["S3Adapter\n@aws-sdk/client-s3"]
    end

    subgraph BusAdapters["Message Bus Adapters"]
        ASB["AzureServiceBusAdapter\n@azure/service-bus"]
        RMQ["RabbitMQAdapter\namqplib"]
    end

    subgraph Infra["Infrastructure"]
        CM["CommonModule\nfactory: env var → adapter"]
        DB["PostgreSQL\nvia Prisma"]
    end

    MS -->|uses| SP
    MS -->|uses| MBP
    SP -.->|implemented by| AB
    SP -.->|implemented by| S3
    MBP -.->|implemented by| ASB
    MBP -.->|implemented by| RMQ
    CM -->|provides| SP
    CM -->|provides| MBP
    MS -->|persists| DB
```

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

## System Architecture

```mermaid
graph TB
    subgraph Clients["Clients"]
        User["User (browser / app)"]
        AI["AI Validation Service\n(external)"]
    end

    subgraph MaterialsService["Materials Service — NestJS 11"]
        subgraph Controllers["Controllers"]
            MC["MaterialController\n22 REST endpoints"]
            PC["PdfExportController\nstats → PDF"]
        end
        JwtGuard2["JwtGuard\nHS256 validation"]
        Svc["MaterialService\nbusiness logic"]
        CommonMod["CommonModule\nadapter factory"]
        Prisma["PrismaService"]
    end

    subgraph External2["External Providers"]
        AzBlob["Azure Blob Storage\nor AWS S3"]
        AzBus["Azure Service Bus\nor RabbitMQ"]
        PG2[("PostgreSQL")]
    end

    User -->|"REST + Bearer JWT"| JwtGuard2
    JwtGuard2 --> Controllers
    Controllers --> Svc
    Svc --> CommonMod
    CommonMod -->|StoragePort| AzBlob
    CommonMod -->|MessageBusPort| AzBus
    AzBus <-->|"material.process / material.responses"| AI
    Svc --> Prisma
    Prisma --> PG2
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
