---
layout: default
title: Architecture Decisions
---

# Architecture Decision Records

This page documents the key architectural decisions made throughout the ECIWise project — what was decided, why, the trade-offs accepted, and how each service evolved over time.

---

## ADR-001 — Microservice Architecture

**Status:** Accepted

### Context

ECIWise needed to support multiple independent domains: authentication, tutoring scheduling, academic materials, real-time chat, study practice, gamification, notifications, and AI predictions. A monolith would couple these domains, slow down team development, and make independent scaling impossible.

### Decision

Build each domain as an independent microservice with its **own database, own deployment, and own technology stack**. Services communicate synchronously via JWT-authenticated REST, and asynchronously via RabbitMQ for domain events.

```mermaid
graph TB
  subgraph Services["Domain Services — each owns its own database"]
    AUTH["wise_auth"]
    TUT["tutoring"]
    MAT["materials"]
    NOTIF["notifications"]
    STUDY["study"]
    TALK["talk"]
    TODO["todo"]
    GAMIF["gamification"]
    GAME["game"]
    COMM["community"]
    AI_D["AI dropout"]
    AI_P["AI performance"]
  end

  RMQ["RabbitMQ\nasync events"]
  FE["Angular Frontend"]

  FE -->|Bearer JWT| AUTH
  FE -->|Bearer JWT| TUT & MAT & NOTIF & STUDY
  FE -->|Bearer JWT| TALK & TODO & GAMIF & GAME & COMM

  AUTH -->|issues JWT| FE
  AUTH & TUT & MAT & COMM -->|domain events| RMQ
  RMQ --> NOTIF & GAMIF & AI_D & AI_P
```

### Consequences

- **Good:** Independent deployments, isolated failures, team autonomy per service, technology flexibility.
- **Accepted cost:** Operational complexity; distributed tracing and observability require explicit tooling. JWT-based JIT user provisioning avoids inter-service user-sync calls.

---

## ADR-002 — RabbitMQ as the Event Broker (Azure Service Bus kept as a contingency alternative)

**Status:** Accepted

### Context

The platform needs asynchronous event delivery for notifications, gamification triggers, and AI prediction requests. Two options were evaluated: **Azure Service Bus** (managed, pay-per-use, Azure-native) and **RabbitMQ** (open-source, self-hostable, protocol-level control via AMQP).

RabbitMQ gives the team direct control over exchanges, queues, routing keys, and dead-letter policies — topology that Azure Service Bus expresses differently and at higher cost. It also runs identically in Docker locally and on any VPS or cloud VM in production, with no per-message billing.

### Decision

**Use RabbitMQ as the chosen broker for both development and production.** Implement a strategy-based broker abstraction in consumer services (notifications, gamification) so that the active broker is selected at runtime via the `MESSAGING_BROKER` environment variable. This keeps Azure Service Bus as a zero-code-change contingency if the team ever migrates to a fully managed Azure hosting model.

```mermaid
graph LR
  subgraph Producers["Event Producers"]
    AUTH["wise_auth"]
    TUT["tutoring"]
    MAT["materials"]
    COMM["community"]
  end

  subgraph Broker["Broker — runtime ENV switch"]
    RMQ["RabbitMQ\n(chosen — dev + production)"]
    ASB["Azure Service Bus\n(contingency — not default)"]
  end

  subgraph Consumers["Event Consumers"]
    NOTIF["notifications"]
    GAMIF["gamification"]
    AI["AI workers\n(dropout · performance)"]
  end

  Producers -->|"MESSAGING_BROKER=rabbitmq ✓"| RMQ
  Producers -. "MESSAGING_BROKER=azure (contingency)" .-> ASB
  RMQ --> Consumers
  ASB -.-> Consumers
```

**RabbitMQ topology (notifications service):**

```mermaid
graph LR
  EX["Exchange: notifications\n(topic, durable)"]
  Q1["notification.individual"]
  Q2["notification.rol"]
  Q3["notification.masivo"]

  EX -->|"routing key: notification.individual"| Q1
  EX -->|"routing key: notification.rol"| Q2
  EX -->|"routing key: notification.masivo"| Q3
```

**AI prediction pipeline via RabbitMQ:**

```mermaid
sequenceDiagram
  participant AUTH as wise_auth
  participant REQ as Exchange: ia.requests
  participant WD as AI worker (desercion)
  participant WP as AI worker (rendimiento)
  participant RES as Exchange: ia.results
  participant AUTH2 as wise_auth (result handler)

  AUTH->>REQ: PredictionRequest { usuarioId, studentName, features }
  REQ->>WD: routing key: desercion
  REQ->>WP: routing key: rendimiento
  WD-->>RES: PredictionResult { model: desercion, prediccionDesercion, confianzaDesercion }
  WP-->>RES: PredictionResult { model: rendimiento, prediccionRendimiento }
  RES-->>AUTH2: Persist prediction result per user
```

### Consequences

- **Good:** Full control over topology (exchanges, routing keys, DLX, TTL) with zero per-message cost. Runs identically locally and in production. Strategy abstraction means Azure SB can be activated at any time by changing one env var.
- **Accepted cost:** Self-hosting RabbitMQ in production requires cluster management, monitoring, and HA configuration. Not a managed service.

---

## ADR-003 — Architecture Pattern per Service: Hexagonal for Complex Domains, Layered for Simple Ones

**Status:** Accepted

### Context

Not every service in ECIWise has the same complexity or the same number of infrastructure dependencies. Applying hexagonal architecture everywhere would add unnecessary boilerplate to services whose domain logic is thin and whose infrastructure dependencies are stable. Applying layered architecture to services with rich domain rules or many swappable adapters would create framework coupling that makes testing and evolution harder.

### Decision

Choose the architecture pattern based on the domain complexity and number of swappable infrastructure dependencies for each service.

**Services using Hexagonal Architecture (Ports & Adapters):**

```mermaid
graph LR
  subgraph Driving["Inbound Adapters"]
    HTTP["REST Controller"]
    CONSUMER["Message Consumer\n(RabbitMQ)"]
  end

  subgraph Hexagon["Hexagon"]
    UC["Use Cases\n(Application Layer)"]
    PORTS["Outbound Ports\n(interfaces — no framework)"]
  end

  subgraph Driven["Outbound Adapters"]
    DB["Prisma / EF / JPA Repository"]
    EMAIL["Email Sender (SendGrid)"]
    TMPL["Template Renderer (Handlebars)"]
    MQ["Message Publisher (RabbitMQ)"]
    STORE["Storage (Azure Blob / S3)"]
  end

  HTTP & CONSUMER --> UC --> PORTS --> DB & EMAIL & TMPL & MQ & STORE
```

| Service | Why hexagonal | Key outbound ports |
|---------|--------------|-------------------|
| `wise_auth` | Many outbound dependencies (DB, cache, 2 RabbitMQ publishers); migrated from layered after god-class problems | `IUsuarioRepository`, `IDatosIaRepository`, `IPrediccionPublisher`, `INotificationPublisher`, `ICacheService` |
| `notifications` | Swappable broker (RabbitMQ / Azure SB), swappable email provider, swappable template engine | `NotificationRepositoryPort`, `EmailSenderPort`, `TemplateRendererPort` |
| `materials` | Swappable cloud storage (Azure Blob / S3), swappable message bus | `StoragePort`, `MessageBusPort`, `MaterialRepositoryPort` |
| `tutoring` | Rich domain (booking rules, concurrency, state machines), fully rewritten | Per-slice repository ports |
| `todo` | Port interfaces isolate Spring Data JPA from use cases | Input/output ports per use case |
| `gamification` | .NET Clean Architecture — RabbitMQ consumer adapter decoupled from domain | Repository ports, `Gamification.Messaging` adapter |

**Services using Classic Layered Architecture (Controller → Service → Repository):**

```mermaid
graph LR
  subgraph Layered["Layered (Controller → Service → Repository)"]
    CTRL["Controller\n(Spring MVC / Spring WS)"]
    SVC["Service\n(business logic)"]
    REPO["Repository\n(Spring Data JPA)"]
    DB[("PostgreSQL")]
  end

  CTRL --> SVC --> REPO --> DB
```

| Service | Technology | Reason for layered |
|---------|-----------|-------------------|
| `study` | Spring Boot · JPA | Thin domain (quiz sessions, flashcard review); stable infrastructure; no swappable adapters needed |
| `talk` | Spring Boot · WebSocket · Redis · MinIO | Chat logic is CRUD + real-time broadcast; Spring Data and MinIO are fixed infrastructure |

**`game` — event-driven goroutine model (Go):**

```mermaid
graph LR
  WS["WebSocket connections\n(Gorilla WS)"]
  ROOM["Room goroutine\nper active room"]
  LOOP["Game loop\ntick-based state update"]
  STATE["In-memory state\nplayers · food · scores"]

  WS -->|"player input"| ROOM
  ROOM --> LOOP --> STATE
  STATE -->|"broadcast state"| WS
```

`game` is a Go WebSocket server with a tick-based game loop and in-memory state. It has no database and no traditional architectural layers — the model that fits its problem is concurrent goroutines per room with a shared state machine, not ports and adapters.

### Consequences

- **Good:** Each service uses the pattern that matches its actual complexity. Simple services (`study`, `talk`) avoid hexagonal boilerplate. Complex services (`notifications`, `tutoring`, `wise_auth`) get full testability and adapter swappability.
- **Accepted cost:** The codebase is not architecturally uniform. New team members must read the README of each service to understand which pattern it follows.

---

## ADR-004 — wise_auth: Migration from Layered to Hexagonal

**Status:** Accepted

### Context

`wise_auth` started as a standard NestJS layered service (Controller → AuthService → PrismaService). As new capabilities were added — IA data management, prediction publishing, tutor assignments, notification publishing, role management — the single `AuthService` grew into a god class with direct Prisma calls and tight coupling to infrastructure.

### Decision

Refactor `wise_auth` to hexagonal architecture, introducing domain ports for all outbound dependencies and isolating each capability in its own module.

```mermaid
graph TB
  subgraph Before["Before — Layered"]
    C1["AuthController"]
    S1["AuthService\n(god class: auth + IA + users + predictions)"]
    P1["PrismaService\n(direct calls everywhere)"]
    C1 --> S1 --> P1
  end

  subgraph After["After — Hexagonal"]
    subgraph Controllers["Inbound"]
      AC["AuthController"]
      IC["IaController"]
    end
    subgraph Domain["Domain Ports (outbound)"]
      UR["IUsuarioRepository"]
      DIR["IDatosIaRepository"]
      ATR["IAsignacionTutorRepository"]
      PP["IPrediccionPublisher"]
      NP["INotificationPublisher"]
      CS["ICacheService"]
    end
    subgraph Adapters["Infrastructure Adapters"]
      PRISMA["Prisma Repositories"]
      RMQ["RabbitMQ Publishers"]
      REDIS["Redis Cache"]
    end
    Controllers --> Domain
    Domain --> Adapters
  end
```

**Key domain port contracts added:**

| Port | Purpose |
|------|---------|
| `IUsuarioRepository` | User CRUD, role and status management |
| `IDatosIaRepository` | AI profile data per student |
| `IAsignacionTutorRepository` | Tutor–student assignment management |
| `IPrediccionPublisher` | Publishes prediction requests to RabbitMQ (`ia.requests` exchange) |
| `INotificationPublisher` | Publishes notification events to RabbitMQ |
| `ICacheService` | Cache abstraction (Redis adapter) |

### Consequences

- **Good:** `wise_auth` can now be tested without a database. The IA module, auth module, and user management module are independently testable. Adding a new outbound dependency (e.g., a new cache provider) is an adapter swap.
- **Accepted cost:** Migration required significant refactoring effort mid-project. All existing tests had to be updated to use port fakes instead of Prisma mocks.

---

## ADR-005 — tutoring: Complete Rewrite with Hexagonal + DDD + Vertical Slicing

**Status:** Accepted

### Context

The original tutoring service was a proof-of-concept with a flat layered structure and mock in-memory persistence. As business requirements matured — recurring availability templates, slot materialization, concurrency-safe booking, cancellation rules, rescheduling — the original codebase could not support them without a full redesign. The domain was too rich for a simple CRUD service.

### Decision

**Rewrite from scratch** using Hexagonal Architecture + Domain-Driven Design + Vertical Slicing. Each business capability is an independent vertical slice with its own domain model, use cases, and infrastructure. Dependencies flow strictly inward.

```mermaid
graph LR
  subgraph Slices["Vertical Slices (left to right = dependency direction)"]
    ID["identidad\nuser mirror (JWT JIT)"]
    CAT["catalogos\nsubjects · rooms · time slots\ntutor-subject assignments"]
    DISP["disponibilidad\nrecurring templates\nmaterialization job"]
    TUT["tutorias\nslot search & query"]
    RES["reservas\nbooking · cancel\nreschedule · evaluate"]
  end

  ID -.->|read| CAT & DISP & TUT & RES
  CAT --> DISP --> TUT --> RES
```

**Business rules encoded in the domain layer:**

| Rule | Enforcement layer |
|------|-----------------|
| No overlapping bookings (RN-01) | Domain — `Reserva` aggregate |
| Slot capacity control (RN-09) | Domain — atomic counter in `Tutoria` |
| Cancellation before session | Domain — `Reserva` state machine |
| Rescheduling constraints | Domain — `Reserva` aggregate |
| Slot materialization idempotency | Application — `DisponibilidadService` cron job |

```mermaid
stateDiagram-v2
  [*] --> Pendiente : student books slot
  Pendiente --> Confirmada : tutor confirms (or auto)
  Pendiente --> Cancelada : student or tutor cancels
  Confirmada --> Completada : session completed
  Confirmada --> Cancelada : cancelled before session
  Completada --> [*]
  Cancelada --> [*]
```

**Technology:**

| Component | Technology |
|-----------|------------|
| Framework | NestJS 11 · TypeScript (strict) |
| Architecture | Hexagonal + DDD + Vertical Slicing |
| ORM / DB | Prisma 7 · PostgreSQL (Neon) |
| Scheduling | `@nestjs/schedule` (materialization cron) |
| Events | `@nestjs/event-emitter` (in-memory, prepared for RabbitMQ) |
| Auth | Passport-JWT HS256 |
| Tests | Jest — domain unit tests with pure fakes (no DB) |

### Consequences

- **Good:** Business rules are explicit, testable, and co-located with the domain. Replacing the persistence layer requires only new adapters. The cron-based materialization decouples scheduling from the booking API.
- **Accepted cost:** Full rewrite cost in sprint time. DDD overhead is only justified by domain complexity — for simpler CRUD services it would be over-engineering.

---

## ADR-006 — gamification: .NET 10 / C# with Hexagonal Architecture

**Status:** Accepted

### Context

The gamification service manages points, levels, achievements, and leaderboards driven by user actions across the platform. The team member leading this service had deep expertise in .NET/C#. Additionally, .NET's strong typing, LINQ, and Entity Framework ecosystem were well-suited to the query-heavy nature of leaderboards and achievement evaluation.

### Decision

Build the gamification service in **.NET 10 / C#** following Clean/Hexagonal Architecture (Ports & Adapters). The service consumes domain events from RabbitMQ via a dedicated `Gamification.Messaging` project and exposes a REST API via `Gamification.Api`.

```mermaid
graph TB
  subgraph GamificationService[".NET 10 Solution — Gamification"]
    subgraph Domain["Gamification.Domain\n(no external dependencies)"]
      ENT["Entities\nUsuario · Recompensa · Nivel\nLogro · Mision"]
      VO["Value Objects\nPoints · ReputationScore\nSubjectCode · AchievementContext"]
      RULES["Business Rules\npoint assignment · level-up\nachievement unlock"]
    end
    subgraph App["Gamification.Application\n(use cases + port interfaces)"]
      CMD["Commands\nAssignPointsCommand\nUnlockAchievementCommand"]
      QRY["Queries\nRankingQueries"]
      HANDLERS["Handlers\n(orchestrate domain + ports)"]
    end
    subgraph Infra["Gamification.Infrastructure\n(adapters)"]
      REPO["EF Core Repositories"]
      DB[("PostgreSQL")]
    end
    subgraph Messaging["Gamification.Messaging\n(RabbitMQ consumer adapter)"]
      CONSUMER["RabbitMQ Consumer\n(domain event listener)"]
    end
    subgraph Api["Gamification.Api\n(HTTP entry point)"]
      CTRL["REST Controllers\n/gamification/*"]
      DI["DI Composition Root"]
    end
  end

  RMQ["RabbitMQ\ndomain events from all services"]

  RMQ -->|"gamification events"| CONSUMER
  CONSUMER --> HANDLERS
  CTRL --> HANDLERS
  HANDLERS --> Domain
  HANDLERS --> REPO
  REPO --> DB
```

**Technology:**

| Component | Technology |
|-----------|------------|
| Platform | .NET 10 · C# |
| Architecture | Hexagonal / Clean Architecture |
| ORM | Entity Framework Core |
| Messaging | RabbitMQ (`Gamification.Messaging`) |
| Tests | xUnit · Moq |

### Consequences

- **Good:** Polyglot microservice architecture — the best tool for the job per team expertise. .NET's LINQ and EF Core are excellent for leaderboard and ranking queries. The service is fully decoupled from all other services via RabbitMQ events.
- **Accepted cost:** Adds a second runtime to the stack (.NET alongside Node.js and JVM). Docker images are slightly larger.

---

## ADR-007 — Two AI Models: Dropout Prediction and Performance Prediction

**Status:** Accepted

### Context

The platform aims to reduce student dropout and improve academic outcomes. Two distinct prediction needs were identified:
- **Dropout risk**: predict whether a student is at risk of dropping out based on socioeconomic, academic, and enrollment factors.
- **Academic performance**: predict a student's likely academic performance based on study habits, attendance, tutoring usage, and extracurricular factors.

These are different ML models with different feature sets and different intervention strategies. Merging them into one service would couple unrelated models and complicate independent retraining.

### Decision

Deploy **two independent AI worker services** (Python), each consuming from a dedicated RabbitMQ queue and publishing results back to `wise_auth` via the `ia.results` exchange. `wise_auth` orchestrates the prediction request lifecycle and stores results per student.

```mermaid
graph LR
  subgraph AuthService["wise_auth"]
    IA_SVC["IaService\n(stores data, publishes requests,\nreceives results)"]
    PUB["IPrediccionPublisher\n(port → RabbitMQ adapter)"]
  end

  subgraph Broker["RabbitMQ"]
    REQ_EX["Exchange: ia.requests"]
    RES_EX["Exchange: ia.results"]
    Q_PERF["Queue: ia.rendimiento.requests"]
    Q_DROP["Queue: ia.desercion.requests"]
    Q_RES["Queue: ia.results"]
  end

  subgraph AIWorkers["AI Workers — Python"]
    PERF["Performance Worker\n(rendimiento)\nrouting key: rendimiento\n11 features: study time,\nabsences, tutoring, sports…"]
    DROP["Dropout Worker\n(desercion)\nrouting key: desercion\n22 features: enrollment,\ngrades, socioeconomic…"]
  end

  IA_SVC -->|request| PUB
  PUB --> REQ_EX
  REQ_EX -->|rendimiento| Q_PERF --> PERF
  REQ_EX -->|desercion| Q_DROP --> DROP
  PERF -->|result| RES_EX --> Q_RES
  DROP -->|result| RES_EX --> Q_RES
  Q_RES -->|PredictionResultMessage| IA_SVC
```

**Feature sets:**

| Model | Routing Key | Key Features |
|-------|-------------|-------------|
| Performance | `rendimiento` | `studyTimeWeekly`, `absences`, `tutoring`, `extracurricular`, `sports`, `music`, `volunteering`, `parentalSupport`, `gender`, `ethnicity`, `parentalEducation` |
| Dropout | `desercion` | `curricularUnits1stSem*` (credited, enrolled, evaluated, approved), `ageAtEnrollment`, `scholarshipHolder`, `debtor`, `tuitionFeesUpToDate`, `course`, `previousQualification`, `maritalStatus`, `applicationMode` + others |

**Result message contract:**

```json
{
  "usuarioId": "uuid",
  "model": "rendimiento | desercion",
  "prediccionRendimiento": "High | Medium | Low",
  "prediccionDesercion": "At Risk | Not At Risk",
  "confianzaDesercion": 0.87
}
```

**Role-based access to predictions:**

| Endpoint | Student | Tutor | Admin |
|----------|:-------:|:-----:|:-----:|
| `GET /ia/me` — own IA data | x | | |
| `PUT /ia/me` — update own features | x | | |
| `PUT /ia/me/prediccion` — save own prediction | x | | |
| `GET /ia/estudiantes` — list all students | | x | x |
| `GET /ia/estudiantes/:id` — student detail | | x | x |
| `GET /ia/metricas` — dashboard metrics | | x | x |
| `GET /ia/estadisticas` — platform-wide stats | | | x |
| `POST /ia/asignaciones` — tutor-student link | | | x |

### Consequences

- **Good:** Models are independently retrainable and deployable. Failure in one worker does not affect the other. Feature sets are cleanly separated. `wise_auth` acts as a thin orchestrator, not an ML service.
- **Accepted cost:** Two additional services to deploy and monitor. The async request/result pattern introduces latency; prediction results are not immediate. Students need to fill in their IA profile data before predictions can be generated.

---

## ADR-008 — Database per Service

**Status:** Accepted

### Context

Multiple microservices sharing a single database creates hidden coupling: schema migrations in one service can break another, a slow query in one domain can starve others, and it becomes impossible to evolve the data model of one service independently.

### Decision

Each microservice **owns its own database instance**. No service reads or writes directly to another service's database. Cross-service data needs are satisfied via:
- **JWT claims** for identity data (name, role, email) — no service calls `wise_auth` to look up a user.
- **JIT user provisioning** — services that need local user records upsert them on first authenticated request from JWT claims.
- **Async events** — for data that changes over time (e.g., a tutoring completion event triggers a gamification point award via RabbitMQ, not a direct DB read).

```mermaid
graph TB
  subgraph Databases["Each service — own PostgreSQL schema / instance"]
    DB_AUTH[("wise_auth DB\nusuarios · roles\nTokens · ia_datos\nasignaciones_tutor")]
    DB_TUT[("tutoring DB\nidentidad · catalogos\ndisponibilidad\ntutorias · reservas")]
    DB_MAT[("materials DB\nmaterials · ratings\ntags · users mirror")]
    DB_NOTIF[("notifications DB\nnotifications\nusuarios · roles\n(JIT mirror)")]
    DB_TALK[("talk DB\nconversations\nmessages · participants")]
    DB_GAMIF[("gamification DB\nusuarios · puntos\nlogros · niveles")]
    DB_STUDY[("study DB\ncollections · flashcards\nreviews · sessions")]
    DB_TODO[("todo DB\ntasks · categories\nachievements")]
  end

  JWT["JWT claims\n{ sub, email, nombre, apellido, rol }\nshared across all services — no DB call"]
  RMQ["RabbitMQ\nasync events for cross-service state"]

  JWT -.->|identity without DB call| DB_TUT & DB_MAT & DB_NOTIF & DB_GAMIF
  RMQ -.->|state propagation| DB_NOTIF & DB_GAMIF
```

### Consequences

- **Good:** Services are fully independent. Schema migrations, performance tuning, and technology choices (ORM, indexing) are local to each service.
- **Accepted cost:** No cross-service JOINs. Reporting that needs data from multiple services requires aggregation at the application level or a dedicated analytics pipeline.
