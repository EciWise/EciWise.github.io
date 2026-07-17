---
layout: default
title: Notifications Service
---

# Notifications Service

## Overview

The `notifications` service is the asynchronous messaging and email delivery backbone of the ECIWISE platform. It follows a **hexagonal (ports and adapters)** architecture: the domain core defines port interfaces, the application layer orchestrates use cases, and the infrastructure layer wires concrete adapters — none of which the domain or application layers know about directly.

- **Email Delivery**: Sends transactional emails (individual, role-based, and bulk) through a **swappable provider** — SendGrid API or SMTP (nodemailer) — selectable at runtime via `EMAIL_PROVIDER`, with configurable sender identity and template-driven content.
- **Template Engine**: Maintains 17 Handlebars HTML templates covering authentication events, tutoring lifecycle, materials and forum activity — with optional internationalization (es, en, de, pt, fr).
- **Message Queue Consumer**: Consumes from Azure Service Bus or RabbitMQ (selectable at runtime via `MESSAGING_BROKER`) with two-layer message validation.
- **REST API**: Exposes notification management endpoints. All user identity is derived from the JWT `sub` claim — never from the URL — preventing IDOR attacks.

The service has **two independent runtime switches**, both resolved at startup and both invisible to the domain: `MESSAGING_BROKER` picks where messages come *from*, and `EMAIL_PROVIDER` picks where emails go *to*.

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
    SMTP["SmtpEmailSender\n[nodemailer / SMTP]"]
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
  POUT -->|"EMAIL_PROVIDER=sendgrid"| SG
  POUT -->|"EMAIL_PROVIDER=smtp"| SMTP
  POUT --> HBS
```

### Email Provider Selection

`EmailSenderPort` has **two interchangeable implementations**. The composition root binds exactly one at startup based on `EMAIL_PROVIDER`; `DispatchNotificationService` only ever sees the port, so neither the domain nor the application layer knows which provider is live.

```mermaid
graph LR
  DNS["DispatchNotificationService\n(application — knows only the port)"]
  PORT["EmailSenderPort\nsendToRecipient · sendBulk"]

  subgraph Root["notifications.module.ts — composition root"]
    SWITCH{"envs.emailProvider"}
  end

  SG["SendgridEmailSender\n@sendgrid/mail\none API call for bulk"]
  SMTP["SmtpEmailSender\nnodemailer\none message per recipient"]

  API["SendGrid HTTPS API"]
  RELAY["SMTP relay\nGmail · Outlook · Mailpit"]

  DNS --> PORT --> SWITCH
  SWITCH -->|"'sendgrid' (default)"| SG --> API
  SWITCH -->|"'smtp'"| SMTP --> RELAY
```

Binding happens once, at module construction:

```ts
{
  // Selección del proveedor de correo por env (EMAIL_PROVIDER).
  provide: EMAIL_SENDER,
  useClass:
    envs.emailProvider === 'smtp' ? SmtpEmailSender : SendgridEmailSender,
}
```

**Why a second provider exists.** SendGrid requires a verified sender identity, and sending on behalf of a domain the team does not own makes messages fail DMARC and land in spam. The SMTP adapter authenticates against the sender's *own* mailbox (e.g. Gmail with an app password), so the provider signs the message with its own SPF/DKIM and it is actually delivered — at no cost and with no domain to own. Pointing `SMTP_HOST` at `localhost:1025` targets a local mailcatcher (Mailpit), which makes the whole email path testable offline with no credentials and no real delivery.

| Aspect | `SendgridEmailSender` (default) | `SmtpEmailSender` |
|--------|--------------------------------|-------------------|
| Transport | SendGrid HTTPS API (`@sendgrid/mail`) | SMTP over TLS (`nodemailer`) |
| Selected by | `EMAIL_PROVIDER=sendgrid` | `EMAIL_PROVIDER=smtp` |
| Required config | `SENDGRID_API_KEY` | `SMTP_HOST`, `SMTP_PORT` (+ optional `SMTP_USER` / `SMTP_PASS`) |
| Credentials | API key, revocable per key, no mailbox password | Mailbox password / app password |
| Bulk strategy | **One API call** using `personalizations` | **One message per recipient** in a loop |
| Deliverability | Needs a verified sender / domain; DMARC fails without it | Signed by the relay's own SPF/DKIM — delivered as the mailbox owner |
| Cost | Free tier, then per-message billing | Free (uses an existing mailbox) |
| TLS | Handled by the provider's API over HTTPS | Port `465` = implicit TLS; `587`/`1025` = STARTTLS negotiated by nodemailer |
| Local testing | Requires a real API key and real sends | `SMTP_HOST=localhost:1025` → Mailpit, no credentials, no real delivery |
| Best for | Production volume, delivery analytics, suppression lists | Zero-cost institutional sending, local dev, contingency if SendGrid is unavailable |

Both adapters embed the institutional logo as an inline `cid:logo` attachment and both fail soft on a missing logo file — the email is still sent. The only behavioural difference the domain could ever notice is bulk fan-out: SendGrid batches recipients into a single request, while SMTP sends one message per recipient so that addresses are never exposed to each other.

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
  class SmtpEmailSender
  class HandlebarsTemplateRenderer

  ManageNotificationsService ..|> ManageNotificationsUseCase : implements
  DispatchNotificationService ..|> DispatchNotificationUseCase : implements
  PrismaNotificationRepository ..|> NotificationRepositoryPort : implements
  PrismaUserRepository ..|> UserRepositoryPort : implements
  SendgridEmailSender ..|> EmailSenderPort : implements
  SmtpEmailSender ..|> EmailSenderPort : implements
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
│   │       ├── email/                     # Dos adaptadores, un solo puerto
│   │       │   ├── sendgrid-email-sender.adapter.ts  # EMAIL_PROVIDER=sendgrid (default)
│   │       │   └── smtp-email-sender.adapter.ts      # EMAIL_PROVIDER=smtp (nodemailer)
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
| Messaging | Azure Service Bus (`@azure/service-bus` 7.x) and RabbitMQ (`amqplib`) — one active per instance (`MESSAGING_BROKER`) |
| Email | SendGrid (`@sendgrid/mail` 8.x) and SMTP (`nodemailer` 9.x) — one active per instance (`EMAIL_PROVIDER`) |
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
  participant E as EmailSenderPort (SendGrid / SMTP)
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
      Note over E: adapter bound at startup by EMAIL_PROVIDER<br/>SendGrid → 202 Accepted · SMTP → 250 OK
      E-->>D: delivered
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

## C4 — Level 1: System Context

```mermaid
graph TB
    subgraph Actors["Actors"]
        USR["Estudiante · Tutor · Admin\nreads the in-app inbox\nreceives the emails"]
    end

    NOTIF["Notifications Service\nTransactional email + in-app inbox\nNestJS 11"]

    subgraph Producers["Producer Services (publish only)"]
        AUTH["Auth Service\nverification codes · CSV welcome"]
        COMM["Community Service"]
        MATS["Materials Service\nupload confirmations"]
        TUTS["Tutoring Service"]
    end

    FE["ECIWISE+ Frontend\nAngular SPA"]

    subgraph External["External / Infrastructure"]
        BROKER["Message Broker\nAzure Service Bus or RabbitMQ\n3 queues"]
        MAIL["Email Provider\nSendGrid API or SMTP relay"]
        PG[("PostgreSQL\nnotifications · usuarios · roles")]
    end

    USR --> FE
    FE -->|"REST + Bearer JWT\n/notificacion"| NOTIF
    AUTH --> BROKER
    COMM --> BROKER
    MATS --> BROKER
    TUTS --> BROKER
    BROKER -->|"consumes 3 queues"| NOTIF
    NOTIF -->|"renders + sends"| MAIL
    MAIL -->|"inbox"| USR
    NOTIF -->|"Prisma"| PG
```

### Actors

| Actor | Interaction |
|---|---|
| Any authenticated user | Reads and manages their own in-app notifications; receives the emails |

### Neighbouring systems

| System | Relationship |
|---|---|
| Producer services | **Fire-and-forget** over the broker. They never call Notifications over HTTP |
| Auth Service | Also the JWT issuer — Notifications validates locally and mirrors users just-in-time |
| Email provider | SendGrid or an SMTP relay, chosen by `EMAIL_PROVIDER` |
| Message broker | Azure Service Bus or RabbitMQ, chosen by `MESSAGING_BROKER` |

Notifications is a **pure sink**: it consumes and sends, but publishes nothing back. Its only synchronous surface is the inbox REST API for the frontend.

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
    SG_A["SendgridEmailSender\n[outbound adapter]"]
    SMTP_A["SmtpEmailSender\n[outbound adapter]"]
    PRISMA["PrismaService"]
  end

  subgraph External["External Systems"]
    ASB["Azure Service Bus\n[3 queues]"]
    RABBITMQ["RabbitMQ\n[exchange + 3 queues]"]
    SENDGRID["SendGrid API\n[HTTPS]"]
    SMTP_S["SMTP relay\n[Gmail · Outlook · Mailpit]"]
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
  DNS -->|"EMAIL_PROVIDER=sendgrid"| SG_A --> SENDGRID
  DNS -->|"EMAIL_PROVIDER=smtp"| SMTP_A --> SMTP_S
  MNS --> PRISMA
  DNS --> PRISMA
  SYNC --> PRISMA
  JWT --> SYNC
  PRISMA --> PG
```

---

## C4 — Level 3: Components

The service is a textbook hexagon: two **inbound** ports (driving) and four **outbound** ports (driven). Every adapter sits on the edge; the two application services in the middle import nothing from `infrastructure/`.

```mermaid
graph TB
    subgraph InboundAdapters["Inbound adapters (driving)"]
        REST["NotificationController\nREST /notificacion\nJwtAuthGuard · Throttler"]
        ORCH["MessagingOrchestratorService\npicks the broker at boot"]
        AZC["AzureMessagingConsumer"]
        RMQC["RabbitMQMessagingConsumer"]
        BASEC["BaseMessagingConsumer\nshared envelope handling"]
    end

    subgraph InboundPorts["Inbound ports"]
        MNUC["ManageNotificationsUseCase\n«interface»"]
        DNUC["DispatchNotificationUseCase\n«interface»"]
    end

    subgraph Core["Application core"]
        MNS["ManageNotificationsService\ninbox · read · delete"]
        DNS["DispatchNotificationService\nrender + send + persist"]
    end

    subgraph OutboundPorts["Outbound ports"]
        NRP["NotificationRepositoryPort\n«interface»"]
        URP["UserRepositoryPort\n«interface»"]
        ESP["EmailSenderPort\n«interface»"]
        TRP["TemplateRendererPort\n«interface»"]
    end

    subgraph OutboundAdapters["Outbound adapters (driven)"]
        PNR["PrismaNotificationRepository"]
        PUR["PrismaUserRepository"]
        SG["SendgridEmailSender"]
        SMTP["SmtpEmailSender"]
        HBS["HandlebarsTemplateRenderer"]
    end

    subgraph Auth["Auth"]
        JWTS["JwtStrategy\nHS256"]
        SYNC["UsuariosSyncService\njust-in-time mirror"]
    end

    PG[("PostgreSQL")]

    REST --> MNUC
    ORCH --> AZC
    ORCH --> RMQC
    AZC --> BASEC
    RMQC --> BASEC
    BASEC --> DNUC

    MNUC -.->|implemented by| MNS
    DNUC -.->|implemented by| DNS

    MNS --> NRP
    DNS --> NRP
    DNS --> URP
    DNS --> ESP
    DNS --> TRP

    NRP -.->|implemented by| PNR
    URP -.->|implemented by| PUR
    ESP -.->|implemented by| SG
    ESP -.->|implemented by| SMTP
    TRP -.->|implemented by| HBS

    REST --> JWTS
    JWTS --> SYNC
    PNR --> PG
    PUR --> PG
    SYNC --> PG
```

| Component | Layer | Role |
|---|---|---|
| `NotificationController` | inbound adapter | The inbox REST API, always scoped to the authenticated user |
| `MessagingOrchestratorService` | inbound adapter | Reads `MESSAGING_BROKER` and starts exactly one consumer |
| `BaseMessagingConsumer` | inbound adapter | Envelope validation and flag handling shared by both brokers |
| `ManageNotificationsService` | core | Inbox queries, mark-read, delete |
| `DispatchNotificationService` | core | Resolves recipients, renders the template, sends, persists |
| `UsuariosSyncService` | auth | Mirrors the JWT's user into the local `usuarios` table on first sight |

### Ports and their adapters

| Port | Direction | Token | Adapter(s) | Selected by |
|---|---|---|---|---|
| `ManageNotificationsUseCase` | inbound | `MANAGE_NOTIFICATIONS_USE_CASE` | `ManageNotificationsService` | — |
| `DispatchNotificationUseCase` | inbound | `DISPATCH_NOTIFICATION_USE_CASE` | `DispatchNotificationService` | — |
| `NotificationRepositoryPort` | outbound | `NOTIFICATION_REPOSITORY` | `PrismaNotificationRepository` | — |
| `UserRepositoryPort` | outbound | `USER_REPOSITORY` | `PrismaUserRepository` | — |
| `EmailSenderPort` | outbound | `EMAIL_SENDER` | `SendgridEmailSender`, `SmtpEmailSender` | `EMAIL_PROVIDER` |
| `TemplateRendererPort` | outbound | `TEMPLATE_RENDERER` | `HandlebarsTemplateRenderer` | — |

Tokens are `Symbol`s rather than strings, so two ports can never collide in the DI container.

---

## C4 — Level 4: Code

```mermaid
classDiagram
    class DispatchNotificationUseCase {
        <<interface>>
        +userExists(email) Promise~boolean~
        +userExistsById(userId) Promise~boolean~
        +sendIndividual(data, locale) Promise~void~
        +saveIndividual(data) Promise~void~
        +sendByRole(data, locale) Promise~void~
        +saveByRole(data) Promise~void~
        +sendBulk(emails, data, locale) Promise~void~
    }

    class ManageNotificationsUseCase {
        <<interface>>
        +findByUser(userId) Promise~Notification[]~
        +countUnread(userId) Promise~number~
        +markRead(userId, id) Promise~Notification~
        +markAllRead(userId) Promise~number~
        +deleteById(userId, id) Promise~void~
    }

    class DispatchNotificationService {
        -NotificationRepositoryPort notifications
        -UserRepositoryPort users
        -EmailSenderPort email
        -TemplateRendererPort renderer
    }

    class ManageNotificationsService {
        -NotificationRepositoryPort notifications
    }

    class NotificationRepositoryPort {
        <<interface>>
        +findByUser(userId) Promise~Notification[]~
        +countUnread(userId) Promise~number~
        +countUnreadChat(userId) Promise~number~
        +markAllRead(userId) Promise~number~
        +markRead(userId, id) Promise~Notification~
        +deleteById(userId, id) Promise~void~
        +deleteAllByUser(userId) Promise~number~
        +create(notification) Promise~void~
    }

    class UserRepositoryPort {
        <<interface>>
        +findByEmail(email) Promise~NotificationUser~
        +findById(id) Promise~NotificationUser~
        +findByRole(role) Promise~NotificationUser[]~
    }

    class EmailSenderPort {
        <<interface>>
        +sendToRecipient(to, subject, html) Promise~void~
        +sendBulk(recipients, subject, html) Promise~void~
    }

    class TemplateRendererPort {
        <<interface>>
        +render(template, context, locale) string
    }

    class PrismaNotificationRepository
    class PrismaUserRepository
    class SendgridEmailSender
    class SmtpEmailSender
    class HandlebarsTemplateRenderer

    ManageNotificationsUseCase <|.. ManageNotificationsService
    DispatchNotificationUseCase <|.. DispatchNotificationService

    DispatchNotificationService ..> NotificationRepositoryPort
    DispatchNotificationService ..> UserRepositoryPort
    DispatchNotificationService ..> EmailSenderPort
    DispatchNotificationService ..> TemplateRendererPort
    ManageNotificationsService ..> NotificationRepositoryPort

    NotificationRepositoryPort <|.. PrismaNotificationRepository
    UserRepositoryPort <|.. PrismaUserRepository
    EmailSenderPort <|.. SendgridEmailSender
    EmailSenderPort <|.. SmtpEmailSender
    TemplateRendererPort <|.. HandlebarsTemplateRenderer
```

### Why `send*` and `save*` are separate operations

`DispatchNotificationUseCase` deliberately exposes sending and persisting as **atomic, independent** operations rather than one `notify()`. The envelope carries two flags — `mandarCorreo` and `guardar` — and the consumer composes them:

| `mandarCorreo` | `guardar` | Result |
|---|---|---|
| `true` | `true` | Email sent **and** stored in the in-app inbox |
| `true` | `false` | Email only — e.g. Auth's verification codes, which must not linger in an inbox |
| `false` | `true` | In-app only, no email |

That is why `userExists` / `userExistsById` are part of the port: a recipient may have no local mirror row yet (Auth can email an address that has never logged in), so the consumer checks before attempting to persist.

### Queue and routing-key contract

| Queue | RabbitMQ routing key | Payload |
|---|---|---|
| `mail.envio.individual` | `notification.individual` | One recipient resolved by email or id |
| `mail.envio.rol` | `notification.rol` | Every user holding a role |
| `mail.envio.masivo` | `notification.masivo` | An explicit list of addresses |

On RabbitMQ the three queues bind to the `notifications` topic exchange (`notification.*`); on Azure Service Bus they are three plain queues. `BaseMessagingConsumer` holds the shared envelope logic, so neither broker adapter reimplements it.

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
    MAILPIT["Mailpit :1025\n(mailcatcher — no credentials)"]
  end

  SENDGRID["SendGrid API\n(external · HTTPS)"]
  SMTP_EXT["SMTP relay\n(external · Gmail/Outlook)"]

  GHA -->|"Azure OIDC"| WEBAPP
  WEBAPP -->|AMQP| ASB
  WEBAPP -->|"HTTPS · EMAIL_PROVIDER=sendgrid"| SENDGRID
  WEBAPP -->|"SMTP/TLS · EMAIL_PROVIDER=smtp"| SMTP_EXT
  SVC --> PG
  SVC --> RMQ
  SVC -->|"EMAIL_PROVIDER=smtp\nSMTP_HOST=localhost:1025"| MAILPIT
  SVC -.->|"EMAIL_PROVIDER=sendgrid"| SENDGRID
```

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | yes | — | HTTP server port |
| `MAIL_FROM` | yes | — | Sender email address (must be a valid email) |
| `EMAIL_PROVIDER` | no | `'sendgrid'` | `'sendgrid'` or `'smtp'` — selects the `EmailSenderPort` adapter |
| `SENDGRID_API_KEY` | if sendgrid | — | SendGrid API key |
| `SMTP_HOST` | if smtp | — | SMTP server host (e.g. `smtp.gmail.com`, `localhost` for Mailpit) |
| `SMTP_PORT` | if smtp | — | SMTP port — `465` implicit TLS · `587` STARTTLS · `1025` Mailpit |
| `SMTP_USER` | no | — | SMTP username. Omit for credential-less relays like Mailpit |
| `SMTP_PASS` | no | — | SMTP password / app password. Omit for Mailpit |
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

**Conditional validation.** `src/config/env.ts` validates the environment with Joi at boot and makes each provider's credentials required *only when that provider is selected* — the same pattern already used for `MESSAGING_BROKER`. Choosing `smtp` therefore does not force a dummy SendGrid key into the environment, and vice versa:

```ts
EMAIL_PROVIDER: joi.string().valid('sendgrid', 'smtp').default('sendgrid'),
SENDGRID_API_KEY: joi.string().when('EMAIL_PROVIDER', {
  is: 'sendgrid', then: joi.required(), otherwise: joi.optional().allow(''),
}),
SMTP_HOST: joi.string().when('EMAIL_PROVIDER', {
  is: 'smtp', then: joi.required(), otherwise: joi.optional().allow(''),
}),
// Credenciales opcionales: Gmail/Outlook las requieren; Mailpit no.
SMTP_USER: joi.string().optional().allow(''),
SMTP_PASS: joi.string().optional().allow(''),
```

A misconfigured provider fails **at startup with an explicit message**, not at the moment the first notification needs to go out.

### Provider Configuration Recipes

| Scenario | Configuration |
|----------|--------------|
| Production (SendGrid) | `EMAIL_PROVIDER=sendgrid` · `SENDGRID_API_KEY=SG.…` · `MAIL_FROM=<verified sender>` |
| Institutional mailbox (Gmail) | `EMAIL_PROVIDER=smtp` · `SMTP_HOST=smtp.gmail.com` · `SMTP_PORT=587` · `SMTP_USER=<mailbox>` · `SMTP_PASS=<app password>` · `MAIL_FROM=<same mailbox>` |
| Local development | `EMAIL_PROVIDER=smtp` · `SMTP_HOST=localhost` · `SMTP_PORT=1025` — no user, no password, nothing leaves the machine |

For SMTP, `MAIL_FROM` should match the authenticated mailbox: the relay signs for its own domain, so a mismatched `From` is exactly what triggers the DMARC failure this adapter exists to avoid.

---

## Design Principles

- **Hexagonal Architecture (Ports and Adapters)**: The domain and application layers only depend on port interfaces — never on Prisma, SendGrid, nodemailer, Handlebars, or any broker. The `NotificationsModule` is the only place where ports are bound to concrete adapters. Adding the SMTP provider is the clearest proof the pattern pays for itself: a whole second email transport landed as **one new adapter class and one ternary in the composition root**, with zero changes to the domain, the use cases, the templates, or the consumers.

- **Two-Layer Validation**: Messages are validated at the envelope level first, then at the type-specific data level. Invalid messages are ACK'd (not retried) to avoid poison-pill queue scenarios.

- **Anti-IDOR by Design**: The `userId` is always extracted from the JWT `sub` claim via `@GetUser('id')`, never from the request URL. All `read/:id` and `DELETE /:id` operations are owner-scoped and return `404` for unauthorized access.

- **Strategy Pattern (Broker Selection)**: `MessagingOrchestratorService` selects `AzureMessagingConsumer` or `RabbitMQMessagingConsumer` at startup. Only one is active per instance. The common dispatch logic lives in `BaseMessagingConsumer`.

- **Strategy Pattern (Email Provider Selection)**: The same idea applied to the outbound edge — `EMAIL_PROVIDER` binds `EmailSenderPort` to `SendgridEmailSender` or `SmtpEmailSender` at composition time. The two switches are orthogonal: any broker can feed any email provider (RabbitMQ + Mailpit for a fully offline local stack, Azure SB + SendGrid in production).

- **Just-in-Time User Provisioning**: `UsuariosSyncService` upserts JWT user data into the local `usuarios` table on each authenticated request, eliminating inter-service synchronization calls.

- **Graceful Degradation**: Failed template lookups fall back to the default locale template. Logo attachment failures are logged but do not block email sending. RabbitMQ consumers reconnect automatically every 5 seconds after connection loss.
