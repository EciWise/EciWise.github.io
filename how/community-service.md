---
layout: default
title: Comunidad Service
---

# Comunidad Service

## Overview

`wise-comunidad` is the microservice responsible for managing the academic community module within the ECIWISE+ platform. It provides students, tutors, and administrators with structured tools for real-time communication, discussion forums, content moderation, and voting.

The service handles three core domains:

- **Real-time chat**: Chat groups with instant messaging via WebSockets (Socket.IO), including typing indicators and group membership management.
- **Forums and threads**: Discussion forums organized by academic subjects, with thread creation, likes, editing, and responses with real-time voting.
- **Content moderation**: A full reporting pipeline with status management and audit logs, accessible to administrators.

---

## Module Flow

The following use case diagram shows the actions available to students and teachers within the academic forums module, including participation, moderation, and content reporting.

[![usecase-forums.png](comunidad/usecase-forums.png)](comunidad/usecase-forums.png)

The following diagram shows the administrative actions available over forums and categories within the community module.

[![usecase-admin.png](comunidad/usecase-admin.png)](comunidad/usecase-admin.png)

The community module supports distinct user roles with different capabilities.

**Students and Tutors** can participate in forum discussions by creating threads and posting responses. They can vote on responses (útil / no útil) in real time, like forums and threads, and send and receive messages within chat groups. They can also report content they consider inappropriate.

**Administrators** have full access to the report management system, including listing all reports, reviewing their statuses, updating moderation states, and viewing usage statistics.

The following sequence diagram shows the interaction flow when a student accesses a forum and reads a thread.

[![seq-forum-read.png](comunidad/seq-forum-read.png)](comunidad/seq-forum-read.png)

---

## Architecture

The service follows a layered architecture. Every request passes through the Security Layer, where the JWT guard validates the Bearer token and resolves the authenticated user before reaching any controller. The Presentation Layer contains the REST controllers grouped by domain. The Business Layer holds the services and domain logic. The Persistence Layer uses Prisma ORM with PostgreSQL. Real-time features are implemented via a Socket.IO gateway operating in the `/chat` namespace. Asynchronous notifications are dispatched through Azure Service Bus.

### Runtime Environment

The service is built with **NestJS 11.x** and **TypeScript**, backed by a **PostgreSQL 14+** database. Schema changes are managed through Prisma Migrate. Asynchronous notifications are delivered via **Azure Service Bus**.

### Package Structure
src/

├── auth/              # JWT guards, decorators, role validation

├── chats/             # Chat groups (REST + WebSocket gateway)

├── forums/            # Forum management

├── threads/           # Discussion threads

├── responses/         # Thread responses

├── votes/             # Voting system (WebSocket)

├── reportes/          # Content moderation and reports

├── prisma/            # Prisma service

├── config/            # Environment configuration and validation

└── main.ts            # Entry point
### Runtime Dependencies

| Dependency | Version |
|---|---|
| NestJS | 11.x |
| TypeScript | — |
| Prisma ORM | — |
| PostgreSQL | 14+ |
| Passport JWT | — |
| Socket.IO | — |
| class-validator | — |
| class-transformer | — |
| Azure Service Bus SDK | — |
| Jest | — |

---

## JWT-based Identity

On every request, the service reads and validates the Bearer JWT. All endpoints are protected by the JWT guard except those explicitly marked with `@Public()`. Roles are enforced via `@Roles()` and `RolesGuard`.

| Claim | Purpose |
|---|---|
| `sub` | Identifies the authenticated user |
| `rol` | User role: `Estudiante`, `Tutor`, `Admin` |

---

## Endpoints

### Chats

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/chats` | Yes | Create a chat group |
| `GET` | `/chats` | Yes | List groups for the authenticated user |
| `GET` | `/chats/:id` | Yes | Get a group with its messages |
| `GET` | `/chats/:id/messages` | Yes | Get messages for a group |
| `POST` | `/chats/:id/messages` | Yes | Send a message (REST) |
| `DELETE` | `/chats/:id` | Yes | Delete a chat group |

### Foros

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/forums` | Yes | Create a forum |
| `GET` | `/forums` | Yes | List all forums |
| `GET` | `/forums/:id` | Yes | Get a forum with its threads |
| `POST` | `/forums/:id/threads` | Yes | Create a thread in a forum |
| `POST` | `/forums/:id/like` | Yes | Like a forum |
| `POST` | `/forums/:id/close` | Yes (creator only) | Close a forum |
| `GET` | `/forums/materias` | Yes | List available subjects |

### Threads

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/threads` | Yes | Create a thread |
| `GET` | `/threads/:id` | Yes | Get a thread with its responses |
| `POST` | `/threads/:id/like` | Yes | Like a thread |
| `POST` | `/threads/:id/edit` | Yes (author only) | Edit a thread |

### Reportes

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/reportes` | Yes | Create a content report |
| `GET` | `/reportes` | `Admin` | List all reports |
| `GET` | `/reportes/mis-reportes` | Yes | List reports submitted by the current user |
| `GET` | `/reportes/estadisticas` | `Admin` | Get moderation statistics |
| `GET` | `/reportes/:id` | `Admin` | Get a report by ID |
| `PATCH` | `/reportes/:id/estado` | `Admin` | Update a report's status |

Full interactive documentation is available at `/api` when the server is running (Swagger/OpenAPI).

---

## WebSocket Events (Chat)

The chat system uses Socket.IO on the `/chat` namespace. Authentication is performed via JWT in the connection handshake.

### Connection

```javascript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000/chat', {
  auth: {
    token: 'tu-jwt-token'
  }
});
```

### Client → Server Events

| Event | Payload | Description |
|---|---|---|
| `joinGroup` | `{ grupoId: string }` | Join a chat group. Required before sending messages. |
| `leaveGroup` | `{ grupoId: string }` | Leave a chat group |
| `sendMessage` | `{ grupoId: string, contenido: string }` | Send a message to a group |
| `typing` | `{ grupoId: string }` | Signal that the user is typing |
| `stopTyping` | `{ grupoId: string }` | Signal that the user stopped typing |

### Server → Client Events

| Event | Description |
|---|---|
| `newMessage` | A new message was sent in the group |
| `userJoined` | A user joined the group |
| `userLeft` | A user left the group |
| `userTyping` | A user is typing |
| `userStoppedTyping` | A user stopped typing |

> **Important**: `joinGroup` must be emitted before `sendMessage`. The server validates group membership before allowing any action.

---

## Service Communication


This service operates as a self-contained service that relies exclusively on the JWT token for user identity and authorization. It does not make runtime calls to the authentication microservice — JWT signatures are validated locally using the shared `JWT_SECRET`.

### Authorization

All role checks (`Estudiante`, `Tutor`, `Admin`) and ownership validation are performed locally using the JWT claims and the data stored in this service's own database. Duplicate reports are prevented at the database level.

### Notifications

Outbound notifications are sent asynchronously to the `mail.envio.individual` queue on Azure Service Bus.

## Flow 1 — Forum Participation and Notifications

The following sequence diagram shows the full interaction flow for creating or responding to a forum post, including optional file attachment and the asynchronous notification dispatch to subscribed users via the notification service.

[![seq-participation-notifications.png](comunidad/seq-participation-notifications.png)](comunidad/seq-participation-notifications.png)

---

## Containers Diagram

> _No diagram available at this time._

The service is deployed using Docker with a multi-stage `Dockerfile`. The build stage compiles the application and the runtime stage serves it on port `3000`. All configuration is injected through environment variables. Database schema changes are applied automatically by Prisma Migrate on startup.

---

## C4 — Level 1: System Context

### Actors

| Actor | Description |
|---|---|
| `Estudiante` | Creates threads, posts in forums, votes on responses, and sends chat messages. |
| `Tutor` | Participates in forums and chat groups with the same capabilities as students. |
| `Administrador` | Manages all reported content and moderation states across the platform. |

### Systems

| System | Type | Description |
|---|---|---|
| `ECIWISE Front` | Internal | Web application (Angular SPA) through which users interact with the platform. |
| `ECIWISE Comunidad` | Internal | Community microservice: real-time chat, forums, threads, votes, and content moderation. |
| `Microservicio de Autenticación` | External | Authenticates users and issues JWTs signed with role and identity claims. |
| `Azure Service Bus` | External | Asynchronous messaging for outbound email notifications. |

### Relationships

| From | To | Protocol | Description |
|---|---|---|---|
| `Estudiante`, `Tutor`, `Administrador` | `ECIWISE Front` | `HTTPS` | All actors use the frontend to access the platform. |
| `ECIWISE Front` | `ECIWISE Comunidad` | `JSON/HTTPS + Bearer JWT` | Calls REST endpoints with `Authorization: Bearer <token>`. |
| `ECIWISE Front` | `ECIWISE Comunidad` | `WebSocket + Bearer JWT` | Connects to the `/chat` namespace for real-time messaging. |
| `ECIWISE Front` | `Microservicio de Autenticación` | `JSON/HTTPS` | Logs in and obtains the JWT. |
| `ECIWISE Comunidad` | `Microservicio de Autenticación` | Local signature validation | Validates JWT signature locally with the shared secret `JWT_SECRET` (no runtime call). |
| `ECIWISE Comunidad` | `Azure Service Bus` | `AMQP` | Publishes notifications to the `mail.envio.individual` queue. |

---

## C4 — Level 2: Containers

### Actor

| Actor | Description |
|---|---|
| `Usuario` | `Estudiante`, `Tutor`, or `Administrador`. |

### Containers

| Container | Technology | Description |
|---|---|---|
| `ECIWISE Front` | `Angular SPA` | Community web interface. |
| `ECIWISE Comunidad API` | `NestJS 11, TypeScript` | Stateless REST + WebSocket API on port `3000`. Validates the JWT on every request and exposes the chat, forum, thread, vote, and moderation modules. |
| `Base de datos` | `PostgreSQL 14+` | `wise_comunidad` schema, versioned with Prisma Migrate. |
| `Azure Service Bus` | `Azure Messaging` | Handles the `mail.envio.individual` queue for asynchronous email notifications. |

### External System

| System | Description |
|---|---|
| `Microservicio de Autenticación` | Issues JWTs with the claims `sub` and `rol`. |

### Relationships

| From | To | Protocol | Description |
|---|---|---|---|
| `Usuario` | `ECIWISE Front` | `HTTPS` | Uses the platform. |
| `ECIWISE Front` | `ECIWISE Comunidad API` | `JSON/HTTPS` | Calls `/chats/*`, `/forums/*`, `/threads/*`, `/reportes/*` with `Authorization: Bearer <token>`. |
| `ECIWISE Front` | `Microservicio de Autenticación` | `JSON/HTTPS` | Login — obtains the JWT. |
| `ECIWISE Comunidad API` | `Microservicio de Autenticación` | Local signature validation | Validates the JWT signature locally with the shared secret `JWT_SECRET` (no runtime call). |
| `ECIWISE Comunidad API` | `Base de datos` | `TCP 5432` | Reads and writes via Prisma ORM. Prisma Migrate runs on startup. |
| `ECIWISE Comunidad API` | `Azure Service Bus` | `AMQP` | Sends outbound notifications to the `mail.envio.individual` queue. |

---

## Design Patterns & Best Practices

**Layered Architecture**
The service is organized into clearly separated modules per domain: Controller → Service → Prisma Repository. Each module has a single responsibility and only communicates with the layer directly below it.

**Module Pattern (NestJS)**
Each domain (chats, forums, threads, responses, votes, reportes) is encapsulated in its own NestJS module with its own controller, service, and Prisma access, keeping boundaries explicit and dependencies injected.

**DTO Pattern**
Request and response objects use `class-validator` decorators for input validation, preventing raw data from reaching the service layer and keeping the API contract explicit.

**Role-Based Authorization**
Permission checks are enforced via `@Roles()` and `RolesGuard` applied at the controller level, with ownership validation performed in service classes. All endpoints require JWT authentication except those explicitly decorated with `@Public()`.

**Real-Time via WebSocket Gateway**
The chat module exposes a Socket.IO gateway in the `/chat` namespace, handling group membership, message broadcasting, and typing indicators independently from the REST layer.

**Centralized Exception Handling**
NestJS exception filters provide consistent HTTP error responses across all controllers.

**Content Sanitization**
Thread content is sanitized before persistence to prevent XSS attacks.

**Database Migrations as Code**
Schema changes are version-controlled through Prisma Migrate scripts, making the database schema reproducible and auditable across environments.

**Duplicate Prevention at Database Level**
Duplicate reports are enforced via database constraints, not only application logic.