---
layout: default
title: Notifications Service
---

# Notifications Service

## Overview

The `notifications` service is the asynchronous messaging and email delivery backbone of the ECIWISE platform. It follows a **hexagonal (ports and adapters)** architecture: the domain core defines port interfaces, the application layer orchestrates use cases, and the infrastructure layer wires concrete adapters — none of which the domain or application layers know about directly.

- **Email Delivery**: Sends transactional emails (individual, role-based, and bulk) via the SendGrid API with configurable sender identity and template-driven content.
- **Template Engine**: Maintains 17 Handlebars HTML templates covering authentication events, tutoring lifecycle, materials and forum activity — with optional internationalization (es, en, de, pt, fr).
- **Message Queue Consumer**: Consumes from Azure Service Bus or RabbitMQ (selectable at runtime via `MESSAGING_BROKER`) with two-layer message validation.
- **REST API**: Exposes notification management endpoints. All user identity is derived from the JWT `sub` claim — never from the URL — preventing IDOR attacks.

---

## Architecture

The service is structured as three layers with dependencies always pointing inward toward the domain core. The `NotificationsModule` is the **composition root** that wires each port (DI token) to its concrete adapter.

```mermaid
graph LR
  subgraph Driving["Inbound Adapters (Driving)"]
    HTTP["NotificationController\n[REST /notificacion]"]
    ORCH["MessagingOrchestratorService\n[selects broker via ENV]"]
    AZ["AzureMessagingConsumer\n[mail.envio.*]"]
    RMQ["RabbitMQMessagingConsumer\n[notification.*]"]
  end

  subgraph Hexagon["Hexagon — Notifications"]
    subgraph App["application/ — use cases"]
      MNS["ManageNotificationsService"]
      DNS["DispatchNotificationService"]
    end
    subgraph Domain["domain/ — pure core, no framework"]
      PIN["Inbound Ports\nManageNotificationsUseCase\nDispatchNotificationUseCase"]
      POUT["Outbound Ports\nNotificationRepositoryPort · UserRepositoryPort\nEmailSenderPort · TemplateRendererPort"]
    end
  end

  subgraph Driven["Outbound Adapters (Driven)"]
    PR_N["PrismaNotificationRepository\n[PostgreSQL]"]
    PR_U["PrismaUserRepository\n[PostgreSQL]"]
    SG["SendgridEmailSender\n[SendGrid API]"]
    HBS["HandlebarsTemplateRenderer\n[*.hbs]"]
  end

  HTTP -->|"ManageNotificationsUseCase"| MNS
  ORCH -->|"MESSAGING_BROKER=azure"| AZ
  ORCH -->|"MESSAGING_BROKER=rabbitmq"| RMQ
  AZ -->|"DispatchNotificationUseCase"| DNS
  RMQ -->|"DispatchNotificationUseCase"| DNS

  MNS --> POUT
  DNS --> POUT
  POUT --> PR_N
  POUT --> PR_U
  POUT --> SG
  POUT --> HBS
```

### Ports and Adapters

```mermaid
classDiagram
  direction LR

  class ManageNotificationsUseCase {
    <<interface — inbound port>>
    +findByUser(userId) Notification[]
    +countUnread(userId) number
    +countUnreadChat(userId) number
    +markAllRead(userId) number
    +markRead(userId, id) Notification
    +deleteById(userId, id) void
  }
  class DispatchNotificationUseCase {
    <<interface — inbound port>>
    +userExists(email) boolean
    +sendIndividual(data, locale?) void
    +saveIndividual(data) void
    +sendByRole(data, locale?) void
    +saveByRole(data) void
    +sendBulk(emails, data, locale?) void
  }
  class NotificationRepositoryPort {
    <<interface — outbound port>>
    +findByUser(userId) Notification[]
    +countUnread(userId) number
    +countUnreadChat(userId) number
    +markAllRead(userId) number
    +markRead(userId, id) Notification
    +deleteById(userId, id) void
    +create(notification) void
  }
  class UserRepositoryPort {
    <<interface — outbound port>>
    +findByEmail(email) NotificationUser
    +findByRole(role) NotificationUser[]
  }
  class EmailSenderPort {
    <<interface — outbound port>>
    +sendToRecipient(to, subject, html) void
    +sendBulk(recipients, subject, html) void
  }
  class TemplateRendererPort {
    <<interface — outbound port>>
    +render(template, context, locale?) string
  }

  class ManageNotificationsService
  class DispatchNotificationService
  class NotificationController
  class AzureMessagingConsumer
  class RabbitMQMessagingConsumer
  class PrismaNotificationRepository
  class PrismaUserRepository
  class SendgridEmailSender
  class HandlebarsTemplateRenderer

  ManageNotificationsService ..|> ManageNotificationsUseCase : implements
  DispatchNotificationService ..|> DispatchNotificationUseCase : implements
  PrismaNotificationRepository ..|> NotificationRepositoryPort : implements
  PrismaUserRepository ..|> UserRepositoryPort : implements
  SendgridEmailSender ..|> EmailSenderPort : implements
  HandlebarsTemplateRenderer ..|> TemplateRendererPort : implements

  NotificationController --> ManageNotificationsUseCase : uses
  AzureMessagingConsumer --> DispatchNotificationUseCase : uses
  RabbitMQMessagingConsumer --> DispatchNotificationUseCase : uses

  ManageNotificationsService --> NotificationRepositoryPort : uses
  DispatchNotificationService --> NotificationRepositoryPort : uses
  DispatchNotificationService --> UserRepositoryPort : uses
  DispatchNotificationService --> EmailSenderPort : uses
  DispatchNotificationService --> TemplateRendererPort : uses
```

### Package Structure

```
src/
├── notifications/                        # Hexagon context (composition root: notifications.module.ts)
│   ├── domain/                           # Pure core — no framework dependencies
│   │   ├── model/
│   │   │   ├── notification.entity.ts    # Notification, NewNotification interfaces
│   │   │   ├── notification-user.ts      # NotificationUser interface
│   │   │   ├── notification-type.enum.ts # TypeEnum (info, success, warning, error…)
│   │   │   ├── template.ts              # TemplateEnum (17 keys)
│   │   │   └── locale.ts                # SupportedLocale type
│   │   └── ports/
│   │       ├── inbound/
│   │       │   ├── manage-notifications.use-case.ts    # ManageNotificationsUseCase
│   │       │   └── dispatch-notification.use-case.ts   # DispatchNotificationUseCase
│   │       └── outbound/
│   │           ├── notification-repository.port.ts     # NotificationRepositoryPort
│   │           ├── user-repository.port.ts             # UserRepositoryPort
│   │           ├── email-sender.port.ts                # EmailSenderPort
│   │           └── template-renderer.port.ts           # TemplateRendererPort
│   ├── application/                      # Use cases — depend only on port interfaces
│   │   ├── manage-notifications.service.ts
│   │   └── dispatch-notification.service.ts
│   ├── infrastructure/
│   │   ├── inbound/                      # Driving adapters
│   │   │   ├── http/
│   │   │   │   ├── notification.controller.ts
│   │   │   │   └── dto/notificacion.dto.ts
│   │   │   └── messaging/
│   │   │       ├── consumers/
│   │   │       │   ├── base-messaging.consumer.ts
│   │   │       │   ├── azure-messaging.consumer.ts
│   │   │       │   └── rabbitmq-messaging.consumer.ts
│   │   │       ├── contracts/            # Zod schemas: envelope, individual, rol, masivo
│   │   │       ├── messaging.orchestrator.ts
│   │   │       └── messaging-provider.port.ts
│   │   └── outbound/                     # Driven adapters
│   │       ├── persistence/
│   │       │   ├── prisma-notification.repository.ts
│   │       │   └── prisma-user.repository.ts
│   │       ├── email/
│   │       │   └── sendgrid-email-sender.adapter.ts
│   │       └── templating/
│   │           └── handlebars-template-renderer.adapter.ts
│   └── notifications.module.ts           # Composition root: ports → adapters via DI
├── auth/                                 # Shared infra: JWT strategy, guard, JIT provisioning
├── config/env.ts                         # Joi-validated environment variables
├── prisma/prisma.service.ts              # Shared Prisma singleton
├── templates/                            # 17 Handlebars email templates
├── app.module.ts
└── main.ts
```

### Runtime Environment

| Component | Technology |
|-----------|------------|
| Framework | NestJS 11 + TypeScript |
| ORM | Prisma 7 + `@prisma/adapter-pg` |
| Messaging | Azure Service Bus (`@azure/service-bus` 7.x) and RabbitMQ (`amqplib`) |
| Email | SendGrid (`@sendgrid/mail` 8.x) |
| Templates | Handlebars (`hbs`) with i18n (es, en, de, pt, fr) |
| Auth | Passport-JWT HS256 with shared `JWT_SECRET` |
| Validation | `class-validator`, `class-transformer`, Zod (contract schemas) |

---

## Notification Flow

```mermaid
sequenceDiagram
  participant P as Producer Service
  participant B as Broker (Azure SB / RabbitMQ)
  participant C as Consumer
  participant V as Validators
  participant D as DispatchNotificationService
  participant R as HandlebarsTemplateRenderer
  participant E as SendGrid
  participant DB as PostgreSQL

  P->>B: NotificationEnvelope {eventType, notificationType, language, data}
  B->>C: Deliver message
  C->>V: validateEnvelope()
  alt Invalid envelope
    V-->>C: errors[]
    C-->>B: ACK (discarded, no retry)
  else Valid envelope
    V-->>C: ok
    C->>V: validateData(individual | rol | masivo)
    alt Invalid data
      V-->>C: errors[]
      C-->>B: ACK (discarded)
    else Valid data
      C->>D: sendIndividual / sendByRole / sendBulk
      D->>R: render(template, context, locale?)
      R-->>D: HTML
      D->>E: sendToRecipient / sendBulk
      E-->>D: 202 Accepted
      opt save = true
        D->>DB: notifications.create(...)
      end
      C-->>B: ACK (processed)
    end
  end
```

### Notification Lifecycle

```mermaid
stateDiagram-v2
  [*] --> Unread : saveIndividual / saveByRole (visto = false)
  Unread --> Read : markRead(id) / markAllRead()
  Unread --> [*] : deleteById(id)
  Read --> [*] : deleteById(id)
```

---

## Data Model

```mermaid
erDiagram
  roles {
    Int    id     PK
    String nombre UK
  }
  usuarios {
    String   id          PK
    String   email       UK
    String   nombre
    String   apellido
    Int      rol_id      FK
    DateTime created_at
    DateTime updated_at
  }
  notifications {
    Int      id            PK
    String   userId        FK
    DateTime fechaCreacion
    String   asunto
    String   resumen
    String   type
    Boolean  visto
  }

  roles ||--o{ usuarios : "groups"
  usuarios ||--o{ notifications : "owns"
```

| Table | Notes |
|-------|-------|
| `roles` | Shared with `auth/`. Created on-demand by `UsuariosSyncService` during JIT provisioning. |
| `usuarios` | Mirror of the central auth registry. Populated JIT from JWT claims on each authenticated request. |
| `notifications` | Owned exclusively by this service. `type` maps to `TypeEnum` (info, success, warning, error, achievement, denied). |

---

## Endpoints

All endpoints are under the `/notificacion` prefix and require `Authorization: Bearer <jwt>`. The `userId` is derived from the token `sub` claim — never from the URL. Operations scoped to `:id` return `404` if the notification does not exist or does not belong to the authenticated user.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/notificacion` | Get authenticated user's notifications (sorted by date desc) |
| `GET` | `/notificacion/unread-count` | Count unread notifications |
| `GET` | `/notificacion/unread-chat-count` | Count unread notifications of type `chat` |
| `PATCH` | `/notificacion/read-all` | Mark all notifications as read |
| `PATCH` | `/notificacion/read/:id` | Mark a single notification as read (owner-scoped) |
| `DELETE` | `/notificacion/:id` | Delete a notification (owner-scoped) |

**NotificacionDto** (response): `id: number`, `asunto: string`, `resumen: string`, `visto: boolean`, `fechaCreacion: Date`, `type: string`

---

## Message Broker Integration

```mermaid
graph LR
  subgraph Producers
    AUTH["auth/"]
    COM["community/"]
    MAT["materials/"]
    TUT["tutorships/"]
  end

  subgraph AzureSB["Azure Service Bus"]
    Q1["mail.envio.individual"]
    Q2["mail.envio.rol"]
    Q3["mail.envio.masivo"]
  end

  subgraph RabbitMQ["RabbitMQ — exchange: notifications (topic)"]
    RK1["notification.individual"]
    RK2["notification.rol"]
    RK3["notification.masivo"]
  end

  subgraph SVC["Notifications Service"]
    AZC["AzureMessagingConsumer"]
    RMQC["RabbitMQMessagingConsumer"]
  end

  AUTH & COM & MAT & TUT -->|Azure SB| Q1 & Q2 & Q3
  AUTH & COM & MAT & TUT -->|RabbitMQ| RK1 & RK2 & RK3

  Q1 & Q2 & Q3 --> AZC
  RK1 & RK2 & RK3 --> RMQC
```

`MessagingOrchestratorService` activates exactly one consumer at startup based on `MESSAGING_BROKER`. Only one broker is active per runtime instance.

### Message Envelope

Every message must conform to the `NotificationEnvelope` format:

```json
{
  "eventType": "notification",
  "notificationType": "individual | rol | masivo",
  "language": "es | en | de | pt | fr",
  "data": { ... }
}
```

### Notification Types

| Type | Who receives | Key `data` fields |
|------|-------------|-------------------|
| `individual` | Single recipient by email | `email`, `template`, `subject` |
| `rol` | All users with a given role | `rol`, `template`, `subject` |
| `masivo` | Batch list (single SendGrid API call) | `emails[]`, `template`, `subject` |

### Validation Pipeline

Messages go through a two-layer validation process:

1. **Envelope layer** — `NotificationEnvelopeValidator` checks `eventType`, `notificationType`, `language`, and `data` presence.
2. **Data layer** — Type-specific Zod schemas check required fields per type.

Failed validation results in the message being ACK'd (not retried) to prevent poison-pill scenarios.

---

## JWT-based Identity

All routes are protected by `JwtAuthGuard` (global `APP_GUARD`) unless marked with `@Public()`. The service validates HS256 tokens using a shared `JWT_SECRET` — no HTTP call to the `auth` service.

| Claim | Purpose |
|-------|---------|
| `sub` | User identifier (UUID) — used as `userId` for all data operations |
| `email` | User email address |
| `nombre` | First name |
| `apellido` | Last name |
| `rol` | Role name (e.g., `admin`, `tutor`, `estudiante`) |

`UsuariosSyncService` performs **Just-in-Time User Provisioning**: on each authenticated request it upserts the JWT user into the local `usuarios` table, keeping the local registry in sync with the central `auth` service without inter-service calls.

---

## Email Templates

The service maintains **17 Handlebars HTML templates** in `src/templates/`.

| Template Key | Email Subject | Category |
|-------------|---------------|----------|
| `cambioDeRol` | Su rol ha sido actualizado | Auth |
| `cuentaEliminada` | Su cuenta ha sido eliminada | Auth |
| `nuevoUsuario` | Nuevo usuario registrado | Auth |
| `SolicitudTutoriaEstudiante` | Ha creado una nueva solicitud de tutoria | Tutoring |
| `SolicitudTutoriaTutor` | Ha recibido una nueva solicitud de tutoria | Tutoring |
| `ConfirmacionTutoriaEstudiante` | Su tutoria ha sido confirmada | Tutoring |
| `ConfirmacionTutoriaTutor` | Se ha confirmado una nueva tutoria | Tutoring |
| `RechazoTutoriaEstudiante` | Su solicitud de tutoria ha sido rechazada | Tutoring |
| `RechazoTutoriaTutor` | Se ha rechazado una solicitud de tutoria | Tutoring |
| `CancelacionTutoriaEstudiante` | Su tutoria ha sido cancelada | Tutoring |
| `CancelacionTutoriaTutor` | Ha sido cancelada una tutoria | Tutoring |
| `CompletacionTutoriaEstudiante` | Su tutoria ha sido completada | Tutoring |
| `CompletacionTutoriaTutor` | Se ha completado una tutoria | Tutoring |
| `nuevoMaterialSubido` | Se ha subido un nuevo material | Materials |
| `nuevoThreadEnForo` | Se ha creado un nuevo hilo en el foro | Forum |
| `mencionThread` | Has sido mencionado en un hilo del foro | Forum |
| `mencionRespuesta` | Has sido mencionado en una respuesta del foro | Forum |

### i18n Resolution

Template resolution is controlled by `TRANSLATED_TEMPLATES_ENABLED` (default `false`). Resolution order:

1. `{templateName}.{locale}.hbs` — if locale enabled and file exists
2. `{templateName}.hbs` — fallback

Fallback locale is configurable via `TRANSLATED_TEMPLATES_FALLBACK` (default `'es'`).

---

## C4 — Level 2: Containers

```mermaid
graph TB
  subgraph Clients["Clients"]
    FE["Frontend / REST Client"]
    PROD["Producer Services\nauth · community · materials · tutorships"]
  end

  subgraph NotifService["Notifications Service — NestJS 11"]
    CTRL["NotificationController\n[REST /notificacion]"]
    ORCH["MessagingOrchestratorService\n[broker selection]"]
    AZ_C["AzureMessagingConsumer"]
    RMQ_C["RabbitMQMessagingConsumer"]
    MNS["ManageNotificationsService\n[inbound use case]"]
    DNS["DispatchNotificationService\n[inbound use case]"]
    SYNC["UsuariosSyncService\n[JIT provisioning]"]
    JWT["JwtStrategy\n[HS256 validation]"]
    HBS_R["HandlebarsTemplateRenderer\n[outbound adapter]"]
    PRISMA["PrismaService"]
  end

  subgraph External["External Systems"]
    ASB["Azure Service Bus\n[3 queues]"]
    RABBITMQ["RabbitMQ\n[exchange + 3 queues]"]
    SENDGRID["SendGrid API"]
    PG[("PostgreSQL\nnotifications · usuarios · roles")]
  end

  FE -->|"Bearer JWT"| CTRL
  PROD -->|AMQP| ASB
  PROD -->|AMQP| RABBITMQ

  CTRL --> MNS
  ORCH -->|"MESSAGING_BROKER=azure"| AZ_C
  ORCH -->|"MESSAGING_BROKER=rabbitmq"| RMQ_C
  AZ_C --> ASB
  RMQ_C --> RABBITMQ
  AZ_C & RMQ_C --> DNS
  DNS --> HBS_R
  DNS --> SENDGRID
  MNS --> PRISMA
  DNS --> PRISMA
  SYNC --> PRISMA
  JWT --> SYNC
  PRISMA --> PG
```

---

## Deployment

```mermaid
graph TB
  subgraph CICD["CI/CD — GitHub Actions"]
    GHA["main_eciwise-alert.yml\nnpm install → build → test → deploy"]
  end

  subgraph Azure["Azure Cloud"]
    WEBAPP["Azure Web App\neciwise-alert · Node.js 20"]
    ASB["Azure Service Bus\n[3 queues]"]
  end

  subgraph LocalDev["docker compose (local dev)"]
    SVC["notifications :3000\n(NestJS + Prisma)"]
    PG["postgres:16 :5432"]
    RMQ["rabbitmq:3 :5672 / management :15672"]
  end

  SENDGRID["SendGrid API\n(external)"]

  GHA -->|"Azure OIDC"| WEBAPP
  WEBAPP -->|AMQP| ASB
  WEBAPP -->|HTTPS| SENDGRID
  SVC --> PG
  SVC --> RMQ
  SVC -->|HTTPS| SENDGRID
```

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | yes | — | HTTP server port |
| `MAIL_FROM` | yes | — | Sender email address |
| `SENDGRID_API_KEY` | yes | — | SendGrid API key |
| `JWT_SECRET` | yes (min 16 chars) | — | HMAC secret for JWT verification |
| `MESSAGING_BROKER` | no | `'azure'` | `'azure'` or `'rabbitmq'` |
| `SERVICE_BUS_CONNECTION_STRING` | if azure | — | Azure Service Bus connection string |
| `RABBITMQ_URL` | if rabbitmq | `amqp://guest:guest@localhost:5672` | RabbitMQ connection URL |
| `DATABASE_URL` | yes | — | PostgreSQL session-mode pooler URL |
| `DIRECT_URL` | no | — | PostgreSQL transaction-mode pooler URL |
| `SWAGGER_ENABLED` | no | `true` | Enable Swagger at `/api` |
| `TRANSLATED_TEMPLATES_ENABLED` | no | `false` | Enable i18n template resolution |
| `TRANSLATED_TEMPLATES_FALLBACK` | no | `'es'` | Fallback locale |
| `THROTTLE_TTL` | no | `60` | Rate-limit window in seconds |
| `THROTTLE_LIMIT` | no | `100` | Max requests per window per client |

---

## Design Principles

- **Hexagonal Architecture (Ports and Adapters)**: The domain and application layers only depend on port interfaces — never on Prisma, SendGrid, Handlebars, or any broker. The `NotificationsModule` is the only place where ports are bound to concrete adapters.

- **Two-Layer Validation**: Messages are validated at the envelope level first, then at the type-specific data level. Invalid messages are ACK'd (not retried) to avoid poison-pill queue scenarios.

- **Anti-IDOR by Design**: The `userId` is always extracted from the JWT `sub` claim via `@GetUser('id')`, never from the request URL. All `read/:id` and `DELETE /:id` operations are owner-scoped and return `404` for unauthorized access.

- **Strategy Pattern (Broker Selection)**: `MessagingOrchestratorService` selects `AzureMessagingConsumer` or `RabbitMQMessagingConsumer` at startup. Only one is active per instance. The common dispatch logic lives in `BaseMessagingConsumer`.

- **Just-in-Time User Provisioning**: `UsuariosSyncService` upserts JWT user data into the local `usuarios` table on each authenticated request, eliminating inter-service synchronization calls.

- **Graceful Degradation**: Failed template lookups fall back to the default locale template. Logo attachment failures are logged but do not block email sending. RabbitMQ consumers reconnect automatically every 5 seconds after connection loss.
