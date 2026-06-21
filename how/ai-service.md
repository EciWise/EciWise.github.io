---
layout: default
title: AI Service
---

# AI Service

## Overview

`ECIWISE+ RAG Service` is the artificial intelligence microservice of the ECIWise platform. It provides students and tutors with intelligent academic support through Retrieval-Augmented Generation (RAG): document analysis, contextual academic chat, adaptive quiz generation and evaluation, and a Socratic tutoring assistant.

The service handles four core capabilities:

- **RAG Chat** — answers academic queries by retrieving semantically relevant documents from the indexed repository and generating a grounded response via an LLM.
- **Socratic Method** — guides students toward self-discovery using a chain of questions rather than direct answers, grounded in available documents.
- **Adaptive Quiz** — generates mixed multiple-choice and open-ended quizzes from a given topic, evaluates student answers using the LLM, and persists learning analytics per student.
- **Document Analysis Pipeline** — asynchronously classifies PDF documents uploaded by users (subject, topic, tags, summary), generates text-chunk embeddings for vector search, and publishes results back through Azure Service Bus.

The service supports multiple LLM backends (Google Gemini, Groq, Ollama) and multiple embedding providers, all switchable through environment variables without changing code.

---

## Use Case Diagram

<div class="mermaid">
graph LR
    subgraph Actors
        ST["🎓 Student / User"]
        AD["⚙️ Developer / Admin"]
    end

    subgraph "ECIWISE+ RAG Service"
        UC1["Ask question — Chat RAG"]
        UC2["Get recommendations"]
        UC3["Navigation chat"]
        UC4["Upload document (ingest)"]
        UC5["Socratic method dialogue"]
        UC6["Start / answer quiz"]
        UC7["View quiz analytics"]
        UC8["View API docs"]
        UC9["Simulate analysis / save (dev)"]
        UC10["Health check"]
    end

    ST --> UC1
    ST --> UC2
    ST --> UC3
    ST --> UC4
    ST --> UC5
    ST --> UC6
    ST --> UC7
    ST --> UC8

    AD --> UC9
    AD --> UC8
    AD --> UC10
</div>

---

## AI Module Flow

The service receives requests from the ECIWise frontend through a shared JWT contract and from the backend through Azure Service Bus for document processing.

### Chat RAG Flow

1. The student submits a question through the frontend chat interface.
2. `POST /api/chat` validates the JWT and the Zod request schema, then passes the query to `ChatUseCase`.
3. `searchRelevantDocuments` first attempts a vector search (pgvector cosine similarity on `DocumentChunk` embeddings); on failure it falls back to keyword ILIKE search on `Summary` fields.
4. Retrieved document snippets are injected into the LLM system prompt as context.
5. If a `chatId` is provided, the last 20 conversation turns are loaded from `ChatHistory` and passed as multi-turn messages.
6. The LLM generates a grounded response which is returned alongside the list of related documents.
7. The conversation turn is persisted to `ChatHistory` for continuity.

### Document Processing Flow

1. A user uploads a PDF through the ECIWise frontend, which stores it in Azure Blob Storage.
2. The frontend/backend publishes an `analysis` message to the `material.process` Azure Service Bus queue, containing the blob URL, file name, and a correlation ID.
3. `ServiceBusService` receives the message, downloads the PDF buffer (via direct HTTP or Azure SDK fallback), and extracts plain text with `pdf-parse` (capped at 30,000 characters).
4. The LLM classifies the document and returns a structured JSON: `{ valid, materia, tema, tags, summary }`.
5. If valid, a `Summary` record is created in PostgreSQL with `docId = PENDING_<correlationId>`.
6. Text is split into chunks by `textChunker` and embeddings are generated asynchronously. Each chunk is stored in `DocumentChunk` with its pgvector embedding.
7. The analysis result is published to the `material.responses` queue.
8. When the backend confirms with a `save` message, the `Summary.docId` is updated to the real document ID, making the document available for RAG search.

### Socratic Flow

1. The student sends a message and the optional prior conversation history to `POST /api/socratic`.
2. `SocraticUseCase` searches for relevant documents and injects them as context.
3. The LLM responds following strict Socratic rules: it answers every message with a guiding question, never gives direct answers, validates correct conclusions briefly, and uses everyday analogies.
4. Responses are limited to 2–3 sentences per turn.

### Quiz Flow

1. The student starts a quiz: `POST /api/quiz/start` with a topic, difficulty (`easy`, `medium`, `hard`), and number of questions (3–15).
2. `QuizUseCase` retrieves relevant documents, constructs a structured prompt with 60% multiple-choice and 40% open-ended questions, and calls the LLM.
3. A `Session` record is created in PostgreSQL with the full question set (with answers hidden from the student).
4. Questions are served one at a time via `POST /api/quiz/answer`. Multiple-choice answers are graded deterministically; open-ended answers are evaluated by the LLM using a rubric (score 0–10, feedback in ≤3 sentences).
5. When the last question is answered, the final score, percentage, and performance label are returned. The result is asynchronously persisted to `QuizResult` if the student is authenticated.
6. `GET /api/quiz/analytics/:studentId` returns topic-level learning analytics: average percentage per topic, weak topics (< 60%), strong topics (≥ 75%), and trend (improving / stable / declining).

---

## Architecture

<div class="mermaid">
graph TB
    subgraph "ECIWise Frontend"
        FE[Angular SPA]
    end

    subgraph "ECIWISE+ RAG Service (Node.js / Express)"
        direction TB
        MWARE["Middleware Layer<br/>auth · rate-limit · validate · topicGuard"]
        ROUTES["Routes<br/>/api/chat · /api/socratic · /api/quiz · /api/simulation"]
        UC["Use Cases<br/>ChatUseCase · SocraticUseCase · QuizUseCase"]
        SVC["Services (Adapters)<br/>GeminiService · EmbeddingService · AdvancedSearchService"]
        BUS["ServiceBusService"]
        BLOB["BlobStorageService"]
        DB["PrismaService"]
        SESSION["SessionService"]
    end

    subgraph "External Services"
        LLM["LLM Provider<br/>Gemini · Groq · Ollama"]
        SB["Azure Service Bus<br/>material.process / material.responses"]
        BLOBST["Azure Blob Storage"]
        PG[("PostgreSQL + pgvector")]
        AUTH["Auth Service<br/>(JWT issuer)"]
    end

    FE -->|Bearer JWT| MWARE
    MWARE --> ROUTES
    ROUTES --> UC
    UC --> SVC
    SVC --> LLM
    SVC --> DB
    DB --> PG
    BUS --> SB
    BUS --> BLOB
    BLOB --> BLOBST
    SESSION --> DB
    AUTH -->|HS256 JWT| FE
</div>

The service follows **Hexagonal Architecture (Ports & Adapters)**. Domain use cases depend on abstract input and output ports. Infrastructure adapters (Gemini, Prisma, Service Bus, Blob Storage) are wired through dependency injection at the route level. The domain layer does not import any infrastructure library.

### C4 — Level 3: Components

<div class="mermaid">
graph TB
    subgraph "Inbound Adapters"
        MW_AUTH["authMiddleware<br/>(JWT HS256 validation)"]
        MW_TOPIC["topicGuard<br/>(LLM academic classifier)"]
        MW_VAL["validate<br/>(Zod schema enforcement)"]
        MW_RATE["rate-limit<br/>(global 100/15min · LLM 20/min)"]
        R_CHAT["chat.routes<br/>POST /api/chat · /stream · /recommendations · /nav"]
        R_SOC["socratic.routes<br/>POST /api/socratic"]
        R_QUIZ["quiz.routes<br/>POST /start · /answer · GET /result · /analytics"]
        R_SIM["simulation.routes<br/>(dev only)"]
        SB_IN["ServiceBusService.startListening<br/>material.process queue"]
    end

    subgraph "Application Layer (Use Cases)"
        UC_CHAT["ChatUseCase<br/>chat · getRecommendations · navigate · stream"]
        UC_SOC["SocraticUseCase<br/>socratic"]
        UC_QUIZ["QuizUseCase<br/>startQuiz · answerQuestion · getResult · getAnalytics"]
    end

    subgraph "Output Ports"
        PORT_LLM["ILLMPort"]
        PORT_REPO["IDocumentRepositoryPort"]
        PORT_EMB["IEmbeddingPort"]
    end

    subgraph "Outbound Adapters"
        GEM["GeminiService<br/>Gemini / Groq / Ollama<br/>callLLM · callLLMStream · analyzeDocumentText<br/>generateQuiz · evaluateAnswer · socraticChat"]
        ADV["AdvancedSearchService<br/>vectorSearch · fullTextSearch<br/>fuzzySearch · semanticSearch · combineScores"]
        EMB["EmbeddingService<br/>Gemini / OpenAI / Ollama / Jina<br/>getEmbedding (768 dims)"]
        PRISMA["PrismaService<br/>Summary · DocumentChunk · Session<br/>ChatHistory · QuizResult · Synonym"]
        SESSION["SessionService<br/>create · get · update · delete · cleanup"]
        BLOB_SVC["BlobStorageService<br/>Azure Blob SDK fallback"]
        SB_OUT["ServiceBusService.sender<br/>material.responses queue"]
    end

    subgraph "External"
        LLM_EXT["LLM Provider<br/>Gemini API · Groq API · Ollama"]
        PG_EXT[("PostgreSQL + pgvector")]
        SB_EXT["Azure Service Bus"]
        BLOB_EXT["Azure Blob Storage"]
    end

    MW_AUTH --> R_CHAT & R_SOC & R_QUIZ
    MW_TOPIC --> R_CHAT & R_SOC
    MW_VAL --> R_CHAT & R_SOC & R_QUIZ
    MW_RATE --> R_CHAT & R_SOC & R_QUIZ
    R_CHAT --> UC_CHAT
    R_SOC --> UC_SOC
    R_QUIZ --> UC_QUIZ
    R_SIM --> GEM & PRISMA
    SB_IN --> GEM
    SB_IN --> BLOB_SVC
    SB_IN --> PRISMA
    SB_IN --> SB_OUT

    UC_CHAT --> PORT_LLM & PORT_REPO
    UC_SOC --> PORT_LLM
    UC_QUIZ --> PORT_LLM & PORT_REPO

    PORT_LLM --> GEM
    PORT_REPO --> PRISMA
    PORT_EMB --> EMB

    GEM --> ADV
    GEM --> EMB
    GEM --> LLM_EXT
    ADV --> EMB
    ADV --> PRISMA
    EMB --> LLM_EXT
    PRISMA --> PG_EXT
    SESSION --> PRISMA
    UC_QUIZ --> SESSION
    SB_OUT --> SB_EXT
    BLOB_SVC --> BLOB_EXT
</div>

### Runtime Environment

| Component | Technology |
|---|---|
| Runtime | Node.js 22.x (ESM native modules) |
| Framework | Express.js 4.x |
| LLM (default) | Google Gemini 2.0 Flash / Flash-Lite |
| ORM | Prisma 6.x |
| Database | PostgreSQL 13+ with `pgvector` extension |
| Messaging | Azure Service Bus 7.x |
| Storage | Azure Blob Storage 12.x |
| Embeddings | Gemini `text-embedding-004` (768 dimensions) |
| Testing | Jest 30.x |
| API Docs | Swagger UI Express |

### Package Structure

```text
src/
├── application/
│   └── use-cases/
│       ├── ChatUseCase.js        # RAG chat, recommendations, navigation, streaming
│       ├── SocraticUseCase.js    # Socratic method chat
│       └── QuizUseCase.js        # Quiz generation, evaluation, analytics
├── domain/
│   └── ports/
│       ├── input/
│       │   ├── IChatPort.js
│       │   ├── ISocraticPort.js
│       │   └── IQuizPort.js
│       └── output/
│           ├── ILLMPort.js
│           ├── IDocumentRepositoryPort.js
│           └── IEmbeddingPort.js
├── middleware/
│   ├── auth.middleware.js        # JWT validation (HS256)
│   ├── topicGuard.middleware.js  # Classifies query as academic / off-topic
│   ├── validate.middleware.js    # Zod schema enforcement
│   ├── upload.middleware.js      # Multipart file handling
│   ├── searchMetrics.middleware.js
│   └── error.middleware.js       # Global error and 404 handlers
├── routes/
│   ├── chat.routes.js            # /api/chat (+ /nav, /recommendations, /stream)
│   ├── socratic.routes.js        # /api/socratic
│   ├── quiz.routes.js            # /api/quiz
│   └── simulation.routes.js      # /api/simulation (dev/test only)
├── schemas/
│   ├── chat.schemas.js           # Zod: chatSchema, recommendationsSchema, navSchema
│   ├── socratic.schemas.js
│   ├── quiz.schemas.js
│   └── simulation.schemas.js
├── services/
│   ├── gemini.service.js         # ILLMPort + IEmbeddingPort implementation (multi-provider)
│   ├── embedding.service.js      # Embedding generation (Gemini / OpenAI / Ollama / Jina)
│   ├── advancedSearch.service.js # Hybrid search: vector + FTS + fuzzy + semantic
│   ├── serviceBus.service.js     # Azure Service Bus message handler
│   ├── blobStorage.service.js    # Azure Blob Storage operations
│   ├── session.service.js        # Quiz session lifecycle
│   └── prisma.service.js         # Prisma client singleton
└── utils/
    ├── sysInstructions.js        # LLM system prompts for all modes
    ├── textChunker.js            # PDF text splitting for embeddings
    ├── logger.js                 # Winston structured logger
    └── retry.js                  # Exponential-backoff retry for LLM calls

prisma/
├── schema.prisma                 # Relational + vector schema
└── migrations/
    └── advanced_search_migration.sql  # pgvector extension + tsvector index
```

### Runtime Dependencies

| Dependency | Purpose |
|---|---|
| Express.js 4.x | HTTP application and routing |
| Prisma 6.x | ORM and generated database client |
| `@azure/service-bus` 7.x | Azure Service Bus SDK |
| `@azure/storage-blob` 12.x | Azure Blob Storage SDK |
| `pdf-parse` | PDF text extraction |
| `fuse.js` | Client-side fuzzy search |
| `zod` 3.x | Runtime request schema validation |
| `jsonwebtoken` | JWT verification |
| `helmet` | HTTP security headers |
| `express-rate-limit` | Rate limiting |
| `swagger-ui-express` | Swagger UI |
| `winston` | Structured logging |
| `nodemon` | Dev hot-reload |
| `jest` 30.x | Unit testing |

---

## JWT-based Identity

All `/api/chat`, `/api/socratic`, and `/api/quiz` routes require a Bearer JWT.

`authMiddleware` validates the token using the shared `JWT_SECRET` with algorithm `HS256`. On success it sets `req.user = { id, role, email }` from the following claims:

| Claim | Aliases | Purpose |
|---|---|---|
| `sub` | `id`, `userId` | User identifier (mapped to `req.user.id`) |
| `rol` | `role` | User role (student, tutor, admin) |
| `email` | — | User email |

In `NODE_ENV=test` the middleware is bypassed entirely so the existing test suite is unaffected.

The service does not maintain a local user projection. User identity is read exclusively from the JWT on each request. The Auth service that issues the tokens is external; the AI service only validates the shared HMAC signature.

---

## Data Model

<div class="mermaid">
erDiagram
    Summary {
        String id PK
        String correlationId UK
        String docId
        String fileName
        String materia
        String tema
        Text summary
        String tags
        DateTime createdAt
        DateTime updatedAt
    }

    DocumentChunk {
        String id PK
        String summaryId FK
        Int chunkIndex
        Text content
        vector_768 embedding
        DateTime createdAt
    }

    Session {
        String id PK
        String type
        Text questions
        Int currentIndex
        Text answers
        Int score
        String topic
        String difficulty
        String studentId
        DateTime createdAt
        DateTime expiresAt
    }

    ChatHistory {
        String id PK
        String chatId UK
        Text messages
        DateTime updatedAt
        DateTime createdAt
    }

    QuizResult {
        String id PK
        String studentId
        String topic
        String difficulty
        Int score
        Int totalQuestions
        Int percentage
        Text answers
        DateTime createdAt
    }

    Synonym {
        UUID id PK
        String term UK
        String[] synonyms
        String language
        DateTime createdAt
        DateTime updatedAt
    }

    Summary ||--o{ DocumentChunk : "has chunks"
</div>

### Summary

Stores the result of the LLM document classification step. Created when a PDF passes analysis; `docId` starts as `PENDING_<correlationId>` until the backend confirms the real document ID.

| Field | Notes |
|---|---|
| `id` | UUID primary key |
| `correlationId` | Unique identifier from the Service Bus message |
| `docId` | Real document ID after confirmation; `PENDING_<correlationId>` before |
| `materia` | Subject extracted by LLM |
| `tema` | Specific topic extracted by LLM |
| `tags` | JSON string array of 3–5 keywords |
| `summary` | LLM-generated academic summary |

### DocumentChunk

Stores overlapping text segments of each document together with their vector embedding for similarity search.

| Field | Notes |
|---|---|
| `summaryId` | FK → `Summary.id` (CASCADE DELETE) |
| `chunkIndex` | Position in the original document |
| `content` | Raw text of this chunk |
| `embedding` | `vector(768)` — Gemini `text-embedding-004` values |

### Session

Transient quiz session persisted to PostgreSQL to survive process restarts. Sessions expire after 2 hours; a cleanup job runs at startup.

| Field | Notes |
|---|---|
| `type` | Currently `quiz` |
| `questions` | JSON array of generated questions (with answers included) |
| `currentIndex` | Next question index |
| `answers` | JSON array of submitted student answers with feedback |
| `score` | Correct answer count so far |
| `expiresAt` | TTL timestamp (2 hours from creation) |

### ChatHistory

Persists multi-turn conversation context by `chatId`. Stores up to the last 20 turns as a JSON array of `{ role, content }` objects. Used by `/api/chat` to provide conversation continuity across requests.

### QuizResult

Permanent record of completed quizzes, used by the analytics endpoint. Each entry captures the topic, difficulty, score, percentage, and the full per-question answer log.

### Synonym

Static synonym dictionary for semantic search expansion. Each term maps to an array of related terms; used by `AdvancedSearchService.semanticSearch` to expand the user's query before embedding.

---

## Business Rules

| Rule | Status | Enforcement |
|---|---|---|
| BR-01 — Only academic queries are answered | Implemented | `topicGuard` middleware calls the LLM classifier; off-topic requests are rejected |
| BR-02 — JWT required on all LLM routes | Implemented | `authMiddleware` applied to `/api/chat`, `/api/socratic`, `/api/quiz` |
| BR-03 — Rate limit on LLM endpoints | Implemented | 20 req/min per IP on LLM routes; 100 req/15min global |
| BR-04 — Documents must be valid and academic | Implemented | LLM analysis sets `valid: false` for off-topic or inappropriate content; rejected docs are not persisted |
| BR-05 — Socratic mode never gives direct answers | Implemented | LLM system instruction with absolute rules; validated by prompt design |
| BR-06 — Quiz sessions expire after 2 hours | Implemented | `expiresAt` checked on every `get`; cleanup on startup |
| BR-07 — LLM calls time out after 30 seconds | Implemented | `AbortSignal.timeout(30_000)` on every fetch to the LLM API |
| BR-08 — LLM calls are retried on transient failure | Implemented | `withRetry` with exponential backoff (3 attempts) |
| BR-09 — Simulation routes disabled in production | Implemented | `if (nodeEnv !== 'production')` guard in `server.js` |
| BR-10 — Multiple-choice grading is deterministic | Implemented | String comparison; LLM is only called for open-ended answers |
| BR-11 — Document chunk embeddings are generated asynchronously | Implemented | `createDocumentChunks` is fire-and-forget; errors are logged but do not fail the analysis response |

---

## Endpoints

The live interactive contract is available at `/api-docs` while the service is running.

### Chat

| Method | Route | Auth | Description |
|---|---|---|---|
| `POST` | `/api/chat` | Yes | Academic RAG chat — answers using document context |
| `POST` | `/api/chat/stream` | Yes | Same as above with SSE token streaming |
| `POST` | `/api/chat/recommendations` | Yes | Returns personalized document recommendations |
| `POST` | `/api/chat/nav` | Yes | Platform navigation assistant |

**`POST /api/chat`** — required field: `message` (max 2000 chars). Optional: `chatId` string to enable persistent multi-turn context.

Response:
```json
{
  "success": true,
  "answer": "Based on the indexed documents...",
  "relatedDocuments": [
    { "docId": "uuid", "fileName": "algebra.pdf", "materia": "Matemáticas", "tema": "Álgebra Lineal" }
  ]
}
```

**`POST /api/chat/stream`** — same request body; response is `text/event-stream`. Each SSE event carries `{ token, done }`. The final event includes `{ done: true, relatedDocuments: [...] }`.

**`POST /api/chat/recommendations`** — optional fields: `descripcion` (max 500 chars), `materias[]` (max 10), `temas[]` (max 10). Returns up to 5 matching documents with a friendly LLM-generated recommendation message.

**`POST /api/chat/nav`** — required: `message` (max 1000 chars). Returns a guided answer about platform features, navigation, or functionality.

### Socratic

| Method | Route | Auth | Description |
|---|---|---|---|
| `POST` | `/api/socratic` | Yes | Socratic method chat — guides with questions, never answers directly |

**`POST /api/socratic`** — required: `message` (max 2000 chars). Optional: `history[]` (max 20 turns of `{ role: 'user'|'assistant', content }` pairs).

Response:
```json
{
  "success": true,
  "answer": "¿Qué crees que pasaría si...?",
  "relatedDocuments": [...]
}
```

### Quiz

| Method | Route | Auth | Description |
|---|---|---|---|
| `POST` | `/api/quiz/start` | Yes | Start an adaptive quiz on any academic topic |
| `POST` | `/api/quiz/answer` | Yes | Submit an answer to the current question |
| `GET` | `/api/quiz/result/:sessionId` | Yes | Get full quiz result with per-question breakdown |
| `GET` | `/api/quiz/analytics/:studentId` | Yes | Learning analytics: topic trends, weak/strong areas |

**`POST /api/quiz/start`** — required: `topic`. Optional: `numQuestions` (default 5, range 3–15), `difficulty` (`easy` | `medium` | `hard`, default `medium`).

Response (first question):
```json
{
  "success": true,
  "sessionId": "uuid",
  "topic": "Cálculo diferencial",
  "totalQuestions": 5,
  "currentQuestion": 1,
  "question": {
    "id": 1,
    "type": "multiple_choice",
    "question": "¿Qué representa la derivada de una función en un punto?",
    "options": ["La pendiente de la tangente", "El área bajo la curva", "El límite al infinito", "El valor de la función"]
  }
}
```

**`POST /api/quiz/answer`** — required: `sessionId` (UUID), `answer` (max 2000 chars).

Response (mid-quiz):
```json
{
  "success": true,
  "correct": true,
  "score": 10,
  "feedback": null,
  "correctAnswer": "La pendiente de la tangente",
  "explanation": "La derivada en un punto x₀ es...",
  "accumulated": "1/1",
  "isComplete": false,
  "currentQuestion": 2,
  "question": { ... }
}
```

Response (final question):
```json
{
  "success": true,
  "correct": false,
  "isComplete": true,
  "finalScore": 3,
  "totalQuestions": 5,
  "percentage": 60,
  "performance": "Bueno",
  "sessionId": "uuid"
}
```

Performance labels: `Excelente` (≥ 90%), `Muy bueno` (≥ 75%), `Bueno` (≥ 60%), `Regular` (≥ 40%), `Necesita refuerzo` (< 40%).

### Health & Meta

| Method | Route | Auth | Description |
|---|---|---|---|
| `GET` | `/health` | No | Deep health check — database, LLM, storage |
| `GET` | `/version` | No | Returns `{ version }` from `package.json` |
| `GET` | `/` | No | API info and available endpoint map |
| `GET` | `/api-docs` | No | Interactive Swagger UI |
| `GET` | `/swagger.json` | No | Raw OpenAPI 3.0 JSON spec |

**`GET /health`** response:

| Status | HTTP | Condition |
|---|---|---|
| `healthy` | 200 | Database reachable and LLM API key valid |
| `degraded` | 207 | Database OK but LLM or storage has an issue |
| `unhealthy` | 503 | Database unreachable |

### Simulation Endpoints (dev/test only)

Disabled when `NODE_ENV=production`.

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/simulation/analysis` | Simulate receiving an `analysis` message for a PDF URL |
| `POST` | `/api/simulation/save` | Simulate the `save` confirmation with a real docId |

These endpoints exist to enable end-to-end testing of the document pipeline without a live Service Bus connection.

### Error Mapping

| HTTP | Meaning |
|---|---|
| 400 | Invalid request — Zod validation failure |
| 401 | Missing, invalid, or expired JWT |
| 404 | Session not found or expired |
| 422 | Business rule violation (e.g., quiz generation failed) |
| 429 | Rate limit exceeded (global 100/15min or LLM 20/min) |
| 503 | Service unavailable (database unreachable) |
| 504 | LLM did not respond within 30 seconds |

All error responses follow the same envelope:
```json
{
  "success": false,
  "error": "Error type",
  "message": "Human-readable description"
}
```

---

## Search Strategy

Document retrieval uses a four-method hybrid approach in `AdvancedSearchService`, combining scores with configurable weights:

<div class="mermaid">
flowchart LR
    Q["User Query"] --> V["Vector Search<br/>40% weight<br/>pgvector cosine similarity<br/>on DocumentChunk embeddings"]
    Q --> F["Full-Text Search<br/>30% weight<br/>PostgreSQL to_tsquery (Spanish)<br/>ILIKE fallback"]
    Q --> FZ["Fuzzy Search<br/>20% weight<br/>Fuse.js on materia / tema / tags<br/>threshold 0.4"]
    Q --> S["Semantic Search<br/>10% weight<br/>Synonym expansion → re-embedding<br/>→ vector cosine search"]
    V --> C["Score Combiner"]
    F --> C
    FZ --> C
    S --> C
    C --> R["Ranked Results<br/>(score ≥ 0.1 threshold)"]
</div>

**Vector search** converts the query to a 768-dimensional embedding using Gemini `text-embedding-004` and finds the closest chunk for each document via pgvector (`<=>` cosine distance operator). Falls back to keyword search if the `pgvector` extension is not available.

**Full-text search** uses PostgreSQL `to_tsquery('spanish', ...)` on an indexed `search_text` column. Falls back to ILIKE on `materia`, `tema`, `tags`, and `summary` if the column does not exist.

**Fuzzy search** loads up to 1000 documents and uses Fuse.js to find approximate matches by `materia`, `tema`, `tags`, and `fileName` — tolerant of typos and partial matches.

**Semantic search** expands the query with synonyms from the built-in `synonymMap` (e.g., `ia` → `inteligencia artificial, machine learning, ai`), generates a new embedding from the expanded query, and runs a second vector search. Synonyms can also be extended at runtime via the `Synonym` model.

When `AdvancedSearchService` is not called explicitly (e.g., inside `GeminiService.searchRelevantDocuments`), a simplified version runs: vector search first, with keyword ILIKE as fallback.

---

## Service Communication

### Authentication Contract

The ECIWise Auth service issues HS256 JWTs. The AI service validates them locally with the shared `JWT_SECRET`. There is no runtime HTTP call to the Auth service for each request.

### Azure Service Bus Integration

<div class="mermaid">
sequenceDiagram
    participant FE as Frontend/Backend
    participant SB as Azure Service Bus
    participant AI as AI Service
    participant LLM as LLM Provider
    participant DB as PostgreSQL

    FE->>SB: Publish message {action: analysis, fileUrl, fileName, correlationId}
    SB->>AI: Deliver message (material.process queue)
    AI->>AI: downloadFile(fileUrl)
    AI->>AI: pdf-parse → extract text
    AI->>LLM: analyzeDocumentText(text, fileName)
    LLM-->>AI: {valid, materia, tema, tags, summary}
    AI->>DB: Summary.create({correlationId, docId: PENDING_..., ...})
    AI->>AI: createDocumentChunks (async, fire-and-forget)
    AI->>SB: Publish response {valid, materia, tema, tags, summary} to material.responses
    SB-->>FE: Deliver analysis result
    FE->>SB: Publish {action: save, correlationId, docId}
    SB->>AI: Deliver save message
    AI->>DB: Summary.update({docId: realId})
</div>

Messages that fail processing after 3 attempts with exponential backoff are moved to the dead-letter queue with a `MaxRetriesExceeded` reason.

The Service Bus integration is optional: if `SERVICE_BUS_CONNECTION_STRING` is not set, the listener is disabled and the service starts normally (degraded mode for document processing only).

### LLM Provider Abstraction

`GeminiService` implements all LLM operations and supports three backends transparently:

| Provider | Protocol | Notes |
|---|---|---|
| `gemini` (default) | Gemini REST API (v1beta) | Supports structured JSON output via `responseSchema` |
| `groq` | OpenAI-compatible `/chat/completions` | Fast inference, no native embedding API |
| `ollama` | Ollama `/api/generate` and `/api/chat` | Local deployment, no cloud cost |

Each `callLLM` invocation uses `AbortSignal.timeout(30_000)` and is wrapped in a `withRetry` helper with exponential backoff.

### Database Communication

Prisma repositories are the only persistence adapters. Raw SQL is used for two specific operations:
1. `pgvector` cosine distance queries (`<=>` operator) which are not supported by Prisma's query builder.
2. Bulk chunk insertion with embedding vectors (`$executeRaw`).

---

## C4 — Level 1: System Context

<div class="mermaid">
graph TB
    subgraph Actors
        ST["🎓 Student<br/>chat · quiz · socratic"]
        TU["📚 Tutor<br/>uploads PDFs"]
        AD["⚙️ Administrator<br/>monitors /health"]
    end

    subgraph "External Systems"
        AUTH["Auth Service<br/>(JWT issuer — HS256)"]
        FE["ECIWise Frontend<br/>(Angular SPA)"]
        BE["ECIWise Backend"]
        SB["Azure Service Bus<br/>material.process / material.responses"]
        BLOB["Azure Blob Storage<br/>(PDF repository)"]
        LLM["LLM Provider<br/>Gemini · Groq · Ollama"]
        PG[("PostgreSQL + pgvector<br/>summaries · embeddings<br/>sessions · quiz results")]
    end

    subgraph "ECIWISE+ AI Service"
        AI["ECIWISE+ RAG Service<br/>Node.js 22 · Express.js 4"]
    end

    ST -->|uses| FE
    TU -->|uploads PDFs| FE
    AD -->|monitors| AI
    FE -->|"Bearer JWT"| AI
    FE -->|login| AUTH
    AUTH -.->|"shared JWT_SECRET"| AI
    BE -->|document events| SB
    AI -->|receive / publish| SB
    AI -->|download PDFs| BLOB
    AI -->|LLM calls| LLM
    AI -->|Prisma reads/writes| PG
</div>

### Actors

| Actor | Interaction |
|---|---|
| Student | Submits academic queries, participates in Socratic dialogue, takes quizzes |
| Tutor | Uploads PDF materials that are analyzed by the AI pipeline |
| Administrator | Monitors service health via `/health`; uses simulation endpoints in dev |

### Systems

| System | Responsibility |
|---|---|
| AI Service | RAG, Socratic method, quiz, document analysis |
| Auth Service | Issues HS256 JWTs; shares `JWT_SECRET` with AI Service |
| Azure Service Bus | Decouples document upload events from analysis processing |
| Azure Blob Storage | Stores the raw PDF files referenced in Service Bus messages |
| LLM Provider | Generates document classification, chat responses, quiz questions |
| PostgreSQL + pgvector | Stores summaries, chunk embeddings, sessions, chat history, quiz results |

---

## C4 — Level 2: Containers

<div class="mermaid">
graph TB
    USER["👤 Student / Tutor"]
    FE["ECIWise Frontend<br/>Angular SPA"]

    subgraph "ECIWISE+ AI Service (single Node.js process)"
        API["REST API<br/>Node.js 22 · Express.js 4<br/>JWT auth · rate-limit · Zod validation · Swagger"]
        LLMSVC["LLM Adapter<br/>GeminiService (multi-provider)<br/>chat · socratic · quiz · analysis · embeddings · streaming"]
        SEARCH["Search Engine<br/>AdvancedSearchService + EmbeddingService<br/>vector · FTS · fuzzy · semantic"]
        BUS["Service Bus Listener<br/>ServiceBusService<br/>receive analysis · publish responses"]
        PRISMA["Persistence Adapter<br/>Prisma 6 + pgvector<br/>Summary · DocumentChunk · Session · ChatHistory · QuizResult"]
    end

    PG[("PostgreSQL 13+<br/>pgvector extension")]
    SB["Azure Service Bus<br/>material.process / material.responses"]
    BLOB["Azure Blob Storage<br/>PDF files"]
    LLM["LLM API<br/>Gemini / Groq / Ollama"]

    USER -->|HTTPS| FE
    FE -->|"JSON/HTTPS + Bearer JWT"| API
    API --> LLMSVC
    API --> SEARCH
    API --> PRISMA
    LLMSVC -->|"HTTPS (30s timeout)"| LLM
    SEARCH --> PRISMA
    BUS -->|AMQP| SB
    BUS -->|HTTPS| BLOB
    BUS --> LLMSVC
    BUS --> PRISMA
    PRISMA -->|"Prisma Client + raw SQL"| PG
</div>

### Containers

| Container | Technology | Responsibility |
|---|---|---|
| REST API | Express.js 4 | Auth, validation, routing, rate limiting, Swagger docs |
| LLM Adapter | GeminiService (ESM) | Multi-provider LLM calls, streaming, embeddings |
| Search Engine | AdvancedSearchService | Hybrid document retrieval for RAG context |
| Service Bus Listener | ServiceBusService | Async document processing pipeline |
| Persistence Adapter | Prisma 6 | All database reads and writes |

The REST API, LLM Adapter, Search Engine, Service Bus Listener, and Persistence Adapter all run in the same Node.js process. They are logical containers, not independent deployments.

---

## Design Patterns & Best Practices

### 1. Hexagonal Architecture (Ports & Adapters)

Input ports (`IChatPort`, `ISocraticPort`, `IQuizPort`) define what use cases expose. Output ports (`ILLMPort`, `IDocumentRepositoryPort`, `IEmbeddingPort`) define what adapters must implement. Use cases depend only on abstractions; `GeminiService` and `PrismaService` are adapters wired in at the route level.

### 2. Multi-provider LLM Abstraction

`GeminiService` implements a unified `callLLM({ model, prompt, messages, systemPrompt, responseMimeType, responseSchema })` interface that dispatches to the correct provider based on `LLM_PROVIDER`. Switching from Gemini to Groq or Ollama requires only an environment variable change. The same applies to streaming via `callLLMStream`.

### 3. RAG (Retrieval-Augmented Generation)

The service never answers academic questions from LLM parametric memory alone. Every chat, socratic, and quiz request retrieves relevant document chunks first and injects them as context into the system prompt. This grounds responses in the institution's indexed material.

### 4. Hybrid Search

Four retrieval methods with configurable weights prevent any single method from being a single point of failure. If pgvector is unavailable, full-text and fuzzy search continue to work. If FTS indexes are missing, ILIKE fallback activates automatically.

### 5. Fire-and-Forget Async Embeddings

Chunk embedding generation is intentionally decoupled from the document analysis response. The Service Bus message is completed and the result returned to the backend immediately after the `Summary` is created. Embedding generation runs asynchronously and its failure does not block the analysis pipeline.

### 6. Retry with Exponential Backoff

`withRetry` wraps every LLM call with up to 3 attempts. The delay doubles between attempts. This handles transient network errors without user impact.

### 7. Topic Guard

`topicGuard` middleware calls the LLM to classify incoming queries before routing them to the main use case. Off-topic queries (recipes, sports, entertainment) are blocked at the middleware layer with a fixed refusal message. This is applied to `/api/chat` and `/api/socratic`.

### 8. Zod Schema Validation at the Boundary

All incoming JSON bodies are validated by Zod schemas before any business logic runs. Invalid requests fail fast at the `validate` middleware with a structured error listing field-level issues.

### 9. Session-based Quiz State

Quiz state is persisted to PostgreSQL `Session` rows rather than held in-process memory. This allows the service to be restarted or scaled horizontally without losing quiz progress. Sessions expire automatically after 2 hours; a cleanup routine removes expired rows at startup.

### 10. Dead-Letter Queue Handling

Service Bus messages that fail processing after the configured retry count are moved to the dead-letter queue with a `deadLetterReason` of `MaxRetriesExceeded`. This prevents poison messages from blocking the queue and makes failures observable.

---

## Deployment

### Environment Variables

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `PORT` | No | `3000` | HTTP port |
| `NODE_ENV` | No | `development` | Runtime mode (`production` enables strict auth and disables simulation routes) |
| `DATABASE_URL` | Yes | — | PostgreSQL connection string (`postgresql://`) |
| `JWT_SECRET` | Yes | — | Shared HMAC secret (HS256) with the Auth service |
| `LLM_PROVIDER` | No | `gemini` | LLM backend: `gemini`, `groq`, or `ollama` |
| `GOOGLE_GEMINI_API_KEY` | If `gemini` | — | Gemini API key |
| `LLM_API_KEY` | If `groq` | — | Groq API key |
| `LLM_BASE_URL` | If `ollama` | `http://localhost:11434` | Ollama base URL |
| `LLM_CHAT_MODEL` | No | `gemini-2.0-flash` | Model for chat, socratic, recommendations |
| `LLM_ANALYSIS_MODEL` | No | `gemini-2.0-flash-lite` | Model for document analysis and quiz evaluation |
| `EMBEDDING_PROVIDER` | No | Same as `LLM_PROVIDER` | Embedding backend (`gemini`, `openai`, `ollama`, `jina`) |
| `EMBEDDING_MODEL` | No | `text-embedding-004` | Embedding model name |
| `EMBEDDING_DIMENSIONS` | No | `768` | Output vector dimensions |
| `SERVICE_BUS_CONNECTION_STRING` | No | — | Azure Service Bus — omit to disable queue listener |
| `SERVICE_BUS_INPUT_QUEUE` | No | `material.process` | Input queue name |
| `SERVICE_BUS_OUTPUT_QUEUE` | No | `material.responses` | Output queue name |
| `SERVICE_BUS_MAX_RETRIES` | No | `3` | Max attempts before dead-lettering |
| `AZURE_STORAGE_CONNECTION_STRING` | No | — | Azure Blob Storage — omit to disable SDK fallback |
| `AZURE_STORAGE_CONTAINER_NAME` | No | `academic-documents` | Blob container |
| `MAX_FILE_SIZE_MB` | No | `10` | Max upload size |
| `CORS_ALLOWED_ORIGINS` | No | Two Azure App Service origins | Comma-separated production CORS allowlist |

### Local Execution

```bash
# Install dependencies
npm install

# Generate Prisma client
npx prisma generate

# Apply database migrations
npx prisma migrate deploy

# Apply pgvector extension and advanced search indexes
npx prisma db execute \
  --file prisma/migrations/advanced_search_migration.sql \
  --schema prisma/schema.prisma

# Start with hot-reload
npm run dev

# Start in production mode
npm start
```

### Verification Commands

```bash
# Run full test suite with coverage
npm test

# Run specific test file
npm test -- gemini.service.test.js

# Smoke test LLM connectivity
npm run smoke:gemini

# Open Prisma Studio (database GUI)
npx prisma studio
```

### CI/CD (GitHub Actions)

The workflow `.github/workflows/main_iamodule.yml` triggers on every push to `main`:

1. **Build** — installs dependencies, runs tests in `NODE_ENV=test`.
2. **Deploy** — applies Prisma migrations and the `advanced_search_migration.sql` against the production `DATABASE_URL` secret.
3. **Azure login** — uses OIDC federated identity (no long-lived credentials).
4. **Web app deploy** — deploys the artifact to Azure App Service slot `Production` under app name `iamodule`.

### GitHub Actions Secrets

| Secret | Purpose |
|---|---|
| `DATABASE_URL` | Production PostgreSQL connection string |
| `AZUREAPPSERVICE_CLIENTID_*` | OIDC federated identity client ID |
| `AZUREAPPSERVICE_TENANTID_*` | Azure tenant ID |
| `AZUREAPPSERVICE_SUBSCRIPTIONID_*` | Azure subscription ID |

### Azure App Service Configuration

Configure in Azure Portal → App Service → Configuration → Application Settings:

```
NODE_ENV=production
PORT=8080
DATABASE_URL=postgresql://...?sslmode=require
JWT_SECRET=<shared secret>
LLM_PROVIDER=gemini
GOOGLE_GEMINI_API_KEY=AIzaSy...
SERVICE_BUS_CONNECTION_STRING=Endpoint=sb://...
AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;...
SCM_DO_BUILD_DURING_DEPLOYMENT=true
```

### Common Issues

**Gemini unreachable from Azure App Service VNet**

If the App Service is VNet-integrated with `WEBSITE_VNET_ROUTE_ALL=1`, outbound HTTPS to `generativelanguage.googleapis.com` is blocked. Solutions:

```bash
# Option 1 — disable route-all
az webapp config appsettings set \
  --name iamodule \
  --settings WEBSITE_VNET_ROUTE_ALL="0"

# Option 2 — switch to Groq
az webapp config appsettings set \
  --name iamodule \
  --settings LLM_PROVIDER="groq" LLM_API_KEY="gsk_..."
```

**Prisma P1012 — invalid protocol**

Prisma 6.x requires `postgresql://` (not `postgres://`):

```
DATABASE_URL=postgresql://user:pass@host:5432/db?sslmode=require
```

**`/health` returns 503**

The database is not reachable. Verify `DATABASE_URL`, PostgreSQL server status, and firewall rules allowing the App Service outbound IP.

**Quiz sessions lost after restart**

Sessions are persisted in PostgreSQL and survive restarts. If sessions appear missing, check that `DATABASE_URL` points to the same database and that migrations have been applied.

---

## Further Reading

- Source repository: [EciWise/ai](https://github.com/EciWise/ai)
- Runtime API contract: `/api-docs`
- Database schema: `prisma/schema.prisma`
- LLM system prompts: `src/utils/sysInstructions.js`
- Search strategy: `src/services/advancedSearch.service.js`
- Document pipeline: `src/services/serviceBus.service.js`

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral' });
</script>
