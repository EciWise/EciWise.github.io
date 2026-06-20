---
layout: default
title: Notifications Service
---

# Notifications Service

## Overview

The `notifications` service is the asynchronous messaging and email delivery backbone of the ECIWISE platform. It consumes events from multiple upstream microservices via message queues, renders Handlebars email templates with i18n support, sends emails through SendGrid, and persists notifications in PostgreSQL for REST API consumption.

- **Email Delivery**: Sends transactional emails (individual, role-based, and mass) via the SendGrid API with configurable sender identity and template-driven content.
- **Template Engine**: Maintains 17 Handlebars HTML templates covering authentication events, tutoring lifecycle, materials, chat, and forum activity — with optional internationalization (es, en, de, pt, fr).
- **Message Queue Consumer**: Consumes from Azure Service Bus or RabbitMQ (selectable at runtime) with a strategy-based broker abstraction, two-layer message validation, and dual action flags (`mandarCorreo` + `guardar`).

---

## Notification Flow

The notification lifecycle begins when an upstream service (Community, Materials, or Tutorships) publishes a message to a shared queue. The service validates the envelope and data, resolves the appropriate template, sends the email via SendGrid, and optionally persists a notification record in the database.

[![Notification Lifecycle](notifications/cicloNotificacion.png)](notifications/cicloNotificacion.png)

A notification passes through three phases:

1. **Received** — The message enters via Azure Service Bus or RabbitMQ.
2. **Validation** — The `NotificationEnvelopeValidator` checks the envelope structure, then a type-specific validator (`IndividualDataValidator`, `RolDataValidator`, or `MasivoDataValidator`) checks the data payload. Invalid messages are ACK'd and discarded with a warning log.
3. **Processing** — If the user exists in the local database, the service evaluates two independent flags: `mandarCorreo` (send email via SendGrid) and `guardar` (persist a notification record). Both can be true, false, or independently set.

---

## Architecture

The service follows a **modular NestJS architecture** with a strategy-based broker abstraction. The `MessagingOrchestratorService` selects between `AzureMessagingConsumer` and `RabbitMQMessagingConsumer` at startup based on the `MESSAGING_BROKER` environment variable. Only one broker is active per runtime instance.

[![Internal Architecture](notifications/notificacionArquitectura.png)](notifications/notificacionArquitectura.png)

### Runtime Environment

The service is built with **NestJS 11** on **TypeScript**, using **Prisma 7** with the `@prisma/adapter-pg` adapter for PostgreSQL connection pooling. Email delivery is handled by **SendGrid** (`@sendgrid/mail` v8.x). Message consumption supports both **Azure Service Bus** (`@azure/service-bus` v7.9.5) and **RabbitMQ** (`amqplib` v2.0.1). Authentication uses **Passport-JWT** with HS256 validation against a shared `JWT_SECRET`.

### Package Structure

```
notifications/
├── src/
│   ├── alerta/                    # REST API module
│   │   ├── alerta.controller.ts   # 6 REST endpoints
│   │   ├── alerta.service.ts      # Email sending, DB CRUD, template rendering
│   │   ├── dto/                   # Data transfer objects
│   │   │   ├── notificacion.dto.ts
│   │   │   ├── rolMail.dto.ts
│   │   │   └── unicoMail.dto.ts
│   │   └── enums/
│   │       ├── template.enum.ts   # 17 template keys → email subjects
│   │       └── type.enum.ts       # Notification types
│   ├── auth/                      # JWT authentication
│   │   ├── jwt.strategy.ts        # HS256 token validation
│   │   ├── jwt-auth.guard.ts      # Route protection
│   │   ├── usuarios-sync.service.ts  # JIT user provisioning
│   │   └── public.decorator.ts    # @Public() decorator
│   ├── config/
│   │   └── env.ts                 # Joi-validated environment variables
│   ├── messaging/                 # Queue consumer layer
│   │   ├── consumers/
│   │   │   ├── base-messaging.consumer.ts       # Shared processing logic
│   │   │   ├── azure-messaging.consumer.ts      # Azure Service Bus consumer
│   │   │   └── rabbitmq-messaging.consumer.ts   # RabbitMQ consumer
│   │   ├── contracts/             # Message schemas (Zod)
│   │   │   ├── notification-envelope.schema.ts
│   │   │   ├── individual.schema.ts
│   │   │   ├── rol.schema.ts
│   │   │   └── masivo.schema.ts
│   │   ├── interfaces/
│   │   │   └── messaging-provider.interface.ts  # IMessagingProvider
│   │   ├── messaging.module.ts
│   │   └── messaging.orchestrator.ts           # Broker selection
│   ├── prisma/
│   │   ├── prisma.module.ts
│   │   └── prisma.service.ts
│   ├── templates/                 # 17 Handlebars email templates
│   │   ├── CancelacionTutoriaEstudiante.hbs
│   │   ├── CancelacionTutoriaTutor.hbs
│   │   ├── CompletacionTutoriaEstudiante.hbs
│   │   ├── CompletacionTutoriaTutor.hbs
│   │   ├── ConfirmacionTutoriaEstudiante.hbs
│   │   ├── ConfirmacionTutoriaTutor.hbs
│   │   ├── RechazoTutoriaEstudiante.hbs
│   │   ├── RechazoTutoriaTutor.hbs
│   │   ├── SolicitudTutoriaEstudiante.hbs
│   │   ├── SolicitudTutoriaTutor.hbs
│   │   ├── cambioDeRol.hbs
│   │   ├── cuentaEliminada.hbs
│   │   ├── mencionRespuesta.hbs
│   │   ├── mencionThread.hbs
│   │   ├── nuevoMaterialSubido.hbs
│   │   ├── nuevoThreadEnForo.hbs
│   │   └── nuevoUsuario.hbs
│   ├── app.module.ts
│   └── main.ts
├── prisma/
│   └── schema.prisma
├── test/
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
├── dockerfile
└── package.json
```

### Runtime Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `@nestjs/common` | ^11.1.8 | NestJS core framework |
| `@nestjs/core` | ^11.0.1 | NestJS core framework |
| `@nestjs/jwt` | ^11.0.2 | JWT token handling |
| `@nestjs/passport` | ^11.0.5 | Passport.js integration |
| `@nestjs/swagger` | ^11.2.1 | API documentation (Swagger) |
| `@prisma/client` | ^7.0.1 | Database ORM |
| `@prisma/adapter-pg` | ^7.1.0 | PostgreSQL adapter for Prisma 7 |
| `@azure/service-bus` | ^7.9.5 | Azure Service Bus client |
| `amqplib` | ^2.0.1 | RabbitMQ client |
| `@sendgrid/mail` | ^8.1.6 | SendGrid email API |
| `class-validator` | ^0.14.2 | DTO validation |
| `class-transformer` | ^0.5.1 | Object transformation |
| `joi` | ^18.0.1 | Environment variable validation |
| `passport-jwt` | ^4.0.1 | JWT Passport strategy |

---

## JWT-based Identity

The service validates JWT tokens using HMAC-SHA256 with a shared `JWT_SECRET`. All routes are protected by `JwtAuthGuard` unless explicitly marked with the `@Public()` decorator.

[![JWT Model](notifications/jwt.png)](notifications/jwt.png)

| Claim | Purpose |
|-------|---------|
| `sub` | User identifier (UUID) |
| `email` | User email address |
| `nombre` | First name |
| `apellido` | Last name |
| `rol` | Role name (e.g., `admin`, `tutor`, `estudiante`) |

The `UsuariosSyncService` performs **Just-in-Time User Provisioning**: on each authenticated request, it upserts the JWT user into the local `usuarios` table, creating the role if it does not exist. This keeps the local user registry in sync with the central `auth` service without inter-service calls.

---

## Data Model

The service uses three tables. The `notifications` table is owned by this service; `usuarios` and `roles` are shared with other microservices and kept in sync via JWT-based JIT provisioning.

[![Database Schema](notifications/esquemaBaseDeDatos.png)](notifications/esquemaBaseDeDatos.png)

### notifications

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | `@id @default(uuid())` |
| `user_id` | String | FK to `usuarios.id` |
| `fecha_creacion` | Timestamptz | `@default(now())` |
| `asunto` | String | Email subject / notification title |
| `resumen` | String | Notification body / summary |
| `visto` | Boolean | `@default(false)` — read status |

**Composite primary key**: `(id, user_id)`

### usuarios

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | `@id` |
| `email` | String | `@unique`, indexed |
| `nombre` | String | First name |
| `apellido` | String | Last name |
| `rol_id` | Int | FK to `roles.id`, `@default(1)` |
| `estado_id` | Int | Account status |

### roles

| Column | Type | Notes |
|--------|------|-------|
| `id` | Int | `@id @default(autoincrement())` |
| `nombre` | String | `@unique` — role name |
| `descripcion` | String? | Optional description |
| `activo` | Boolean | Active status |

---

## Endpoints

All endpoints are under the `/notificacion` prefix.

### Notification Queries

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/notificacion/:userId` | Get all notifications for a user (sorted by date desc) |
| `GET` | `/notificacion/unread-count/:userId` | Count unread notifications |
| `GET` | `/notificacion/unread-chat-count/:userId` | Count unread chat notifications (subject starts with "Hay un nuevo mensaje") |
| `PATCH` | `/notificacion/read-all/:userId` | Mark all notifications as read |
| `PATCH` | `/notificacion/read/:id` | Mark a single notification as read |
| `DELETE` | `/notificacion/:id` | Delete a notification |

### DTOs

**NotificacionDto** (response):
- `id: number`, `asunto: string`, `resumen: string`, `visto: boolean`, `fechaCreacion: Date`, `type: string`

**UnicoMailDto** (individual email input):
- Required: `email`, `template`, `resumen`, `guardar`, `type`
- Optional: `name`, `mandarCorreo` (default true), `year` (default current year), `tema`, `material_author`, `link`, `oldRole`, `newRole`, `materia`, `fileName`, `fecha`, `tutor`, `estudiante`, `mensaje`, `modalidad`, `hora`, `nombreGrupo`, `taggedBy`, `threadTitle`, `createdBy`, `mentionedBy`

**RolMailDto** (role-based email input):
- Required: `rol`, `template`, `resumen`, `guardar`, `type`
- Optional: `mandarCorreo` (default true), `material_name`, `material_subject`, `material_topic`, `material_author`, `link`

---

## Message Broker Integration

The service supports two mutually exclusive message brokers, selected at startup via the `MESSAGING_BROKER` environment variable (`'azure'` | `'rabbitmq'`, default `'azure'`).

### Broker Selection

The `MessagingOrchestratorService` instantiates the appropriate consumer during `OnModuleInit` and stops it during `OnModuleDestroy`. Only one broker is active per instance.

### Queue Topology

| Broker | Queue / Exchange | Type | Purpose |
|--------|------------------|------|---------|
| **Azure Service Bus** | `mail.envio.individual` | Queue | Single-recipient emails |
| | `mail.envio.rol` | Queue | Role-based emails |
| | `mail.envio.masivo` | Queue | Mass/batch emails |
| **RabbitMQ** | `notifications` | Exchange (topic, durable) | Message routing |
| | `notification.individual` | Queue (bound to exchange) | Single-recipient emails |
| | `notification.rol` | Queue (bound to exchange) | Role-based emails |
| | `notification.masivo` | Queue (bound to exchange) | Mass/batch emails |

### Message Envelope Structure

Every message must conform to the `NotificationEnvelope` format:

```json
{
  "eventType": "notification",
  "notificationType": "individual" | "rol" | "masivo",
  "language": "es" | "en" | "de" | "pt" | "fr",
  "data": { ... }
}
```

### Validation Pipeline

Messages go through a **two-layer validation** process:

1. **Envelope layer** — `NotificationEnvelopeValidator` checks `eventType`, `notificationType`, `language`, and `data` presence.
2. **Data layer** — Type-specific validators check required fields:

| Type | Required Fields |
|------|-----------------|
| `individual` | `email` (string), `template` (string), `resumen` (string), `guardar` (boolean, optional) |
| `rol` | `rol` (string), `template` (string), `resumen` (string), `guardar` (boolean, optional) |
| `masivo` | `emails` (non-empty string array), `template` (string), `resumen` (string), `guardar` (boolean, optional) |

Failed validation results in the message being ACK'd (not retried) with a warning log to prevent poison pill scenarios.

### Sequence Diagrams

#### Individual Notification

[![Individual Notification Sequence](notifications/notificacionIndividual.png)](notifications/notificacionIndividual.png)

The upstream service publishes a message to `mail.envio.individual`. The orchestrator selects the active provider, delegates to `BaseMessagingConsumer.processEnvelope()`, validates the envelope and data, looks up the user by email, and conditionally sends the email and/or persists the notification.

#### Role-Based Notification

[![Role-Based Notification Sequence](notifications/notificacionRol.png)](notifications/notificacionRol.png)

The upstream service publishes a message to `mail.envio.rol` with a role name. The service queries all users with that role and sends an individual email to each. If `guardar=true`, a notification record is created for each recipient.

#### Mass Notification

[![Mass Notification Sequence](notifications/notificacionMasiva.png)](notifications/notificacionMasiva.png)

The upstream service publishes a message to `mail.envio.masivo` with an array of email addresses. SendGrid personalizations group all recipients into a single API call. If `guardar=true`, individual notification records are created for each email.

---

## Email Templates

The service maintains **17 Handlebars HTML templates** in `src/templates/`. Each template is a full HTML email with inline CSS. A logo CID attachment (`cid:logo`) is loaded from `docs/logo.png` if the file exists.

### Template Catalog

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
| `nuevoMensaje` | Hay un nuevo mensaje del Grupo: {name} | Chat |
| `nuevoThreadEnForo` | Se ha creado un nuevo hilo en el foro | Forum |
| `mencionThread` | Has sido mencionado en un hilo del foro | Forum |
| `mencionRespuesta` | Has sido mencionado en una respuesta del foro | Forum |

### Template Variables by Category

**Auth templates**: `name`, `year`, `oldRole`, `newRole`

**Tutoring templates**: `name`, `year`, `materia`, `fecha`, `hora`, `tutor`, `estudiante`, `modalidad`, `razon`, `mensaje`

**Materials templates**: `name`, `year`, `fileName`, `materia`, `tema`, `material_author`, `link`

**Chat templates**: `name`, `year`, `nombreGrupo`, `mensaje`

**Forum templates**: `name`, `year`, `nombreGrupo`, `threadTitle`, `createdBy`, `mentionedBy`, `taggedBy`, `fecha`

### i18n Resolution

Template resolution is controlled by `TRANSLATED_TEMPLATES_ENABLED` (default `false`). Supported locales: `es`, `en`, `de`, `pt`, `fr`. Resolution order:

1. `{templateName}.{locale}.hbs` (if locale enabled and file exists)
2. `{templateName}.hbs` (fallback)
3. Inline HTML `<p>Hola {{name}},</p><p>{{mensaje}}</p>` (hardcoded fallback)

The fallback locale is configurable via `TRANSLATED_TEMPLATES_FALLBACK` (default `'es'`).

---

## Service Communication

[![Service Connections](notifications/notifications-service-connections.png)](notifications/notifications-service-connections.png)

### Producers

The following upstream services publish messages to the notification queues:

| Producer Service | Broker | Queues | Message Types |
|-----------------|--------|--------|---------------|
| `community/` | Azure Service Bus | `mail.envio.*` | Individual, Rol, Masivo |
| `materials/` | Azure Service Bus | `mail.envio.*` | Individual, Rol, Masivo |
| `tutorships/` | Azure Service Bus | `mail.envio.*` | Individual, Rol, Masivo |

### JWT Trust Model

All backend services validate JWTs independently by verifying the HMAC-SHA256 signature against a shared `JWT_SECRET` — no callback to the `auth` service. The notifications service extracts user claims from the token and upserts them into the local `usuarios` table via `UsuariosSyncService`.

### Database Sharing

The service shares the `usuarios` and `roles` tables with `auth/`, `community/`, `materials/`, and `tutorships/`. The `notifications` table is unique to this service. User data is kept in sync through JIT provisioning from JWTs rather than direct inter-service database access.

---

## C4 — Level 1: System Context

[![System Context](notifications/notificaciones.png)](notifications/notificaciones.png)

### Actors

| Actor | Description |
|-------|-------------|
| Student | Receives email notifications and views in-app notifications |
| Tutor | Receives email notifications and views in-app notifications |
| Administrator | Receives system-wide email notifications |

### Systems

| System | Type | Description |
|--------|------|-------------|
| Notifications Service | Internal | Email delivery and notification persistence |
| Auth Service | Internal | JWT authentication and user management |
| Community Service | Internal | Forums, chat, content moderation |
| Materials Service | Internal | Study materials management |
| Tutorships Service | Internal | Tutoring session management |
| SendGrid | External | Email delivery API |
| PostgreSQL | External | Notification and user data persistence |

### Relationships

| From | To | Protocol | Description |
|------|----|----------|-------------|
| Community Service | Notifications Service | Azure Service Bus | Publishes notification events |
| Materials Service | Notifications Service | Azure Service Bus | Publishes notification events |
| Tutorships Service | Notifications Service | Azure Service Bus | Publishes notification events |
| Notifications Service | SendGrid | HTTPS | Sends transactional emails |
| Notifications Service | PostgreSQL | TCP | Reads/writes notification and user data |
| Student / Tutor / Administrator | Notifications Service | HTTPS | Views notifications via REST API |

---

## C4 — Level 2: Containers

[![Containers](notifications/contenedores.png)](notifications/contenedores.png)

### Containers

| Container | Technology | Description |
|-----------|------------|-------------|
| Notifications API | NestJS 11, TypeScript | REST API for notification queries and email sending |
| AzureMessagingConsumer | `@azure/service-bus` | Consumes from Azure Service Bus queues |
| RabbitMQMessagingConsumer | `amqplib` | Consumes from RabbitMQ exchange/queues |
| MessagingOrchestrator | NestJS | Selects and manages the active broker provider |
| BaseMessagingConsumer | NestJS | Shared message processing logic (validation, email, persistence) |
| AlertaService | NestJS | Business logic for email sending and notification CRUD |
| AlertaController | NestJS | REST endpoints for notification queries |
| UsuariosSyncService | NestJS | JIT user provisioning from JWT |
| JwtStrategy | Passport-JWT | HS256 token validation |
| PrismaService | Prisma 7 | Database access layer |

### External Systems

| System | Description |
|--------|-------------|
| Azure Service Bus | Default message broker (3 queues) |
| RabbitMQ | Alternative message broker (exchange + 3 queues) |
| SendGrid API | Email delivery provider |
| PostgreSQL | Notification and user data persistence |

### Relationships

| From | To | Protocol | Description |
|------|----|----------|-------------|
| Community / Materials / Tutorships | Azure Service Bus | AMQP | Publishes `mail.envio.*` messages |
| Community / Materials / Tutorships | RabbitMQ | AMQP | Publishes to `notifications` exchange |
| AzureMessagingConsumer | Azure Service Bus | AMQP | Subscribes to 3 queues |
| RabbitMQMessagingConsumer | RabbitMQ | AMQP | Subscribes to 3 queues |
| BaseMessagingConsumer | AlertaService | Method call | Delegates email sending and persistence |
| AlertaController | AlertaService | Method call | REST API business logic |
| AlertaService | SendGrid API | HTTPS | Sends emails |
| AlertaService | PrismaService | Method call | Database CRUD |
| UsuariosSyncService | PrismaService | Method call | Upserts users from JWT |

---

## Deployment

[![Deployment](notifications/despliegue.png)](notifications/despliegue.png)

The service is deployed as an **Azure Web App** (`eciwise-alert`) running **Node.js 20**. The CI/CD pipeline uses **GitHub Actions** (`main_eciwise-alert.yml`) triggered on push to `main`. The pipeline runs `npm install`, `npm run build`, `npm test`, and deploys to Azure using OIDC authentication.

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PORT` | yes | — | HTTP server port |
| `MAIL_FROM` | yes | — | Sender email address |
| `SENDGRID_API_KEY` | yes | — | SendGrid API key |
| `JWT_SECRET` | yes | — | HMAC secret for JWT verification |
| `MESSAGING_BROKER` | no | `'azure'` | `'azure'` or `'rabbitmq'` |
| `SWAGGER_ENABLED` | no | `true` | Enable Swagger docs at `/api` |
| `TRANSLATED_TEMPLATES_ENABLED` | no | `false` | Enable i18n template resolution |
| `TRANSLATED_TEMPLATES_FALLBACK` | no | `'es'` | Fallback locale |
| `SERVICE_BUS_CONNECTION_STRING` | conditional | — | Required when `MESSAGING_BROKER=azure` |
| `RABBITMQ_URL` | conditional | `'amqp://guest:guest@localhost:5672'` | Required when `MESSAGING_BROKER=rabbitmq` |
| `DATABASE_URL` | yes | — | PostgreSQL session-mode pooler URL |
| `DIRECT_URL` | yes | — | PostgreSQL transaction-mode pooler URL |

---

## Design Patterns & Best Practices

- **Strategy Pattern (Broker Selection)**: The `IMessagingProvider` interface defines `start()` and `stop()` methods. `AzureMessagingConsumer` and `RabbitMQMessagingConsumer` are concrete implementations selected at runtime by `MessagingOrchestratorService` based on the `MESSAGING_BROKER` environment variable. Only one broker is active per instance.

- **Template Method Pattern (Message Processing)**: `BaseMessagingConsumer` contains the shared processing logic (`processEnvelope`, `processIndividualMessage`, `processRolMessage`, `processMasivoMessage`). Concrete consumers only implement broker-specific connection handling, inheriting all validation and dispatch logic.

- **Two-Layer Validation**: Messages are validated first at the envelope level (structure, event type, notification type) and then at the data level (type-specific required fields). This separation ensures well-formed messages before business logic executes, and invalid messages are safely ACK'd to prevent poison pill scenarios.

- **Dual Action Flags**: Each message can independently trigger email sending (`mandarCorreo`) and database persistence (`guardar`). This allows upstream producers to control notification behavior precisely — e.g., send email only, persist only, or both.

- **Graceful Degradation**: Failed template lookups fall back to inline HTML; logo attachment failures are logged but do not block email sending; invalid messages are ACK'd (not retried) to avoid queue poisoning. RabbitMQ consumers handle connection loss with automatic reconnection every 5 seconds.

- **Just-in-Time User Provisioning**: The `UsuariosSyncService` upserts JWT user data into the local `usuarios` table on each authenticated request, eliminating the need for inter-service user synchronization calls while keeping the local registry current.

- **Connection Pooling with Prisma 7**: Uses `@prisma/adapter-pg` for session-mode and transaction-mode connection pooling via `DATABASE_URL` and `DIRECT_URL`, optimizing database connection management in a cloud environment.
