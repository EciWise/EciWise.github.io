---
layout: default
title: Actors, Roles & Permissions
---

# Actors, Roles & Permissions

Actors represent the external entities that interact with the ECIWise system. Identity and authorization are managed centrally by the `wise_auth` microservice, which issues HS256 JWTs consumed by every downstream service without a round-trip to Auth.

---

## Roles

The `wise_auth` service defines three roles in `Role` enum:

```mermaid
graph LR
  subgraph Roles["Role enum — wise_auth"]
    EST["estudiante\n(default on registration)"]
    TUT["tutor\n(assigned by admin)"]
    ADM["admin\n(full platform access)"]
  end

  ADM -->|"can promote to"| TUT
  ADM -->|"can manage"| EST
```

| Role | Value | Description | Assigned by |
|------|-------|-------------|-------------|
| Student | `estudiante` | Default role on registration. Access to academic support tools, tutoring booking, study, forums, and AI recommendations. | Automatic on registration |
| Tutor | `tutor` | Authorized user (monitor) who offers academic tutoring sessions. Manages availability and conducts sessions. | Administrator |
| Admin | `admin` | Full platform access. Manages users, content, monitors, and institutional statistics. | System / manual |

---

## Account States

```mermaid
stateDiagram-v2
  [*] --> pendiente_activacion : Email/password registration
  [*] --> activo : Google OAuth registration
  pendiente_activacion --> activo : Email verified
  activo --> suspendido : Admin suspends account
  activo --> inactivo : Account deactivated
  suspendido --> activo : Admin reactivates
  inactivo --> activo : Admin reactivates
```

| State | Description |
|-------|-------------|
| `pendiente_activacion` | Registered via email, awaiting email verification |
| `activo` | Active account with full access |
| `inactivo` | Deactivated account |
| `suspendido` | Suspended by administrator |

---

## Authentication & JWT

All services receive and validate the JWT locally (HS256, shared `JWT_SECRET`) — no HTTP call to `wise_auth` per request.

```mermaid
sequenceDiagram
  participant U as User
  participant FE as Frontend
  participant AUTH as wise_auth
  participant SVC as Any Microservice

  U->>FE: Login (email/password or Google OAuth)
  FE->>AUTH: POST /auth/login or GET /auth/google
  AUTH-->>FE: JWT { sub, email, nombre, apellido, rol }
  FE->>SVC: Request + Bearer JWT
  SVC->>SVC: Validate JWT locally (HS256)
  SVC-->>FE: Response (identity from token claims)
```

**JWT Claims:**

| Claim | Type | Purpose |
|-------|------|---------|
| `sub` | UUID | User identifier — used as `userId` in all services |
| `email` | string | User email address |
| `nombre` | string | First name |
| `apellido` | string | Last name |
| `rol` | string | Role (`estudiante`, `tutor`, `admin`) |

---

## Use Case: Estudiante

```mermaid
graph LR
  ST(("Estudiante"))

  subgraph Tutorias["Tutorias"]
    T1["Buscar tutorias disponibles"]
    T2["Reservar tutoria"]
    T3["Cancelar / reprogramar reserva"]
    T4["Calificar tutoria"]
    T5["Ver historial de tutorias"]
    T6["Ver reputacion del tutor"]
  end

  subgraph Estudio["Estudio y Practica"]
    E1["Practicar con Flashcards"]
    E2["Participar en Quiz"]
    E3["Ver leaderboard"]
    E4["Recibir recomendaciones IA"]
  end

  subgraph Comunicacion["Comunicacion"]
    C1["Enviar mensajes de chat"]
    C2["Unirse a videollamada"]
    C3["Crear / responder en foros"]
  end

  subgraph Materiales["Materiales"]
    M1["Buscar y descargar materiales"]
    M2["Subir materiales"]
    M3["Calificar materiales"]
  end

  subgraph Cuenta["Cuenta"]
    A1["Registrarse / Iniciar sesion"]
    A2["Ver notificaciones"]
    A3["Gestionar perfil"]
    A4["Gestionar tareas pendientes"]
  end

  ST --> Tutorias
  ST --> Estudio
  ST --> Comunicacion
  ST --> Materiales
  ST --> Cuenta
```

---

## Use Case: Tutor / Monitor

```mermaid
graph LR
  TU(("Tutor / Monitor"))

  subgraph Disponibilidad["Disponibilidad"]
    D1["Publicar disponibilidad recurrente"]
    D2["Modificar disponibilidad"]
    D3["Cancelar disponibilidad"]
  end

  subgraph GestionTutorias["Gestion de Tutorias"]
    G1["Ver tutorias programadas"]
    G2["Registrar asistencia"]
    G3["Registrar observaciones"]
    G4["Evaluar participacion del estudiante"]
    G5["Cancelar tutoria"]
  end

  subgraph Comunicacion["Comunicacion"]
    C1["Enviar mensajes de chat"]
    C2["Unirse a videollamada"]
    C3["Crear / responder en foros"]
  end

  subgraph Materiales["Materiales"]
    M1["Buscar y descargar materiales"]
    M2["Ver historial de tutorias"]
    M3["Ver reputacion de estudiantes"]
  end

  TU --> Disponibilidad
  TU --> GestionTutorias
  TU --> Comunicacion
  TU --> Materiales
```

---

## Use Case: Administrador

```mermaid
graph LR
  AD(("Administrador"))

  subgraph Usuarios["Gestion de Usuarios"]
    U1["Gestionar usuarios"]
    U2["Cambiar rol de usuarios"]
    U3["Gestionar tutores autorizados"]
    U4["Activar / suspender cuentas"]
    U5["Enviar mensajes a tutores"]
  end

  subgraph Contenido["Gestion de Contenido"]
    C1["Subir colecciones de preguntas"]
    C2["Modificar colecciones de preguntas"]
    C3["Eliminar colecciones de preguntas"]
  end

  subgraph Estadisticas["Estadisticas e Informes"]
    E1["Ver estadisticas institucionales"]
    E2["Ver historial de tutorias del sistema"]
    E3["Generar reportes"]
    E4["Ver reputacion de tutores y estudiantes"]
  end

  AD --> Usuarios
  AD --> Contenido
  AD --> Estadisticas
```

---

## Permissions Matrix

| Functionality | Estudiante | Tutor | Admin |
|---|:---:|:---:|:---:|
| Register / Login (email or Google) | x | x | x |
| Manage own profile | x | x | x |
| View notifications | x | x | x |
| **Tutorias** | | | |
| Search available tutoring sessions | x | x | x |
| Book a tutoring session | x | | |
| Cancel a reservation | x | | |
| Reschedule a tutoring session | x | | |
| Rate a tutoring session | x | | |
| View own tutoring history | x | x | x |
| View tutor reputation | x | | x |
| View student reputation | | x | x |
| Publish / modify / cancel availability | | x | |
| View scheduled sessions | | x | |
| Register attendance | | x | |
| Register observations | | x | |
| Evaluate student participation | | x | |
| Cancel a tutoria (as tutor) | | x | |
| View system-wide tutoring history | | | x |
| Manage authorized tutors | | | x |
| **Study & Practice** | | | |
| Practice with FlashCards | x | | |
| Participate in Quiz | x | | |
| View leaderboard | x | | |
| Receive AI recommendations | x | | |
| Manage own tasks (Todo) | x | | |
| **Communication** | | | |
| Send chat messages | x | x | |
| Join a video call | x | x | |
| Create a forum post | x | x | |
| Reply to a forum post | x | x | |
| Send messages to tutors (admin) | | | x |
| **Materials** | | | |
| Search and download materials | x | x | x |
| Upload materials | x | x | |
| Rate materials | x | x | |
| **Administration** | | | |
| Manage users (roles, status) | | | x |
| Upload / modify / delete question collections | | | x |
| View institutional statistics | | | x |
| Generate reports | | | x |
