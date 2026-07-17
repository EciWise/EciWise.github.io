---
layout: default
title: Auth Service
---

# Auth Service

## Overview

`wise_auth` is the authentication, authorization and user-management microservice for the ECIWise platform. It is the identity authority: it issues the HS256 JWTs that every other service validates locally, and it owns the `usuarios` table that the rest of the platform refers to by `sub` (user id).

The service ships four functional modules inside a single NestJS 11 process:

| Module | Responsibility |
|---|---|
| **Auth** | Registration with email verification codes, login, Google OAuth 2.0, password change/reset, JWT issuance |
| **Gestión de Usuarios** | User directory, profiles, admin CRUD, CSV bulk load, role/status changes, statistics |
| **IA** | Student AI feature data, prediction requests/results, tutor↔student assignments |
| **Feature Flags** | Platform-wide on/off switches read by the frontend |

Internally it follows a **hexagonal (ports & adapters)** layout: application services depend on interfaces declared in `src/domain/ports`, and the concrete Prisma / RabbitMQ / cache adapters are bound to those ports by injection token. See [C4 — Level 4](#c4--level-4-code) below.

---

## C4 — Level 1: System Context

Who talks to Auth, and what Auth talks to.

```mermaid
graph TB
    subgraph Actors["Actors"]
        EST["Estudiante\nregisters · logs in\nmanages own profile"]
        TUT["Tutor / Profesor\nreads assigned students"]
        ADM["Administrador\nmanages users · roles\nfeature flags"]
    end

    AUTH["Wise Auth\nIdentity authority for ECIWISE+\nNestJS 11 · Node 24"]

    subgraph Ecosystem["ECIWISE+ Ecosystem"]
        FE["ECIWISE+ Frontend\nAngular SPA"]
        SVCS["Other Microservices\nTutoring · Materials · Talk\nGame · Gamification · Study · Community"]
        NOTIF["Notifications Service\nsends the emails"]
        IAW["IA Workers\nrendimiento · deserción\nPython"]
    end

    subgraph External["External / Infrastructure"]
        GOOGLE["Google OAuth 2.0"]
        PG[("PostgreSQL\nusuarios · roles · estados\ndatos_ia · asignaciones · flags")]
        REDIS[("Redis\nverification codes\nstatistics cache")]
        MQ["RabbitMQ\nia.requests · ia.results\nnotifications"]
    end

    EST --> FE
    TUT --> FE
    ADM --> FE

    FE -->|"REST + Bearer JWT"| AUTH
    FE -->|"GET /auth/google"| AUTH
    AUTH <-->|"OAuth 2.0 consent + profile"| GOOGLE
    SVCS -.->|"validate JWT locally\nshared HS256 secret — no HTTP call"| AUTH

    AUTH -->|"Prisma"| PG
    AUTH -->|"codes · cache"| REDIS
    AUTH -->|"publishes verification\n& welcome emails"| MQ
    MQ --> NOTIF
    AUTH -->|"publishes prediction requests"| MQ
    MQ --> IAW
    IAW -->|"publishes results"| MQ
    MQ -->|"consumed by Auth"| AUTH
```

### Actors

| Actor | Interaction |
|---|---|
| Estudiante | Registers (email + code, or Google), logs in, edits profile, fills AI data |
| Tutor | Authenticates; reads the students assigned to them via `/ia/estudiantes` |
| Administrador | Manages users, roles, statuses, CSV bulk load, statistics, feature flags |

### Neighbouring systems

| System | Relationship |
|---|---|
| Frontend (Angular) | Only synchronous consumer of the REST API; stores the JWT |
| Other microservices | Consume the JWT **offline** — they share `JWT_SECRET` and never call Auth |
| Notifications Service | Receives `masivo` envelopes over RabbitMQ and actually sends the mail |
| IA Workers | Consume `ia.requests`, publish back to `ia.results`, which Auth persists |
| Google OAuth 2.0 | Federated sign-in for institutional / Gmail accounts |

---

## C4 — Level 2: Containers

Auth is a **single deployable process** (Azure Web App `eciwise-auth`) plus the stateful infrastructure it owns.

```mermaid
graph TB
    FE["ECIWISE+ Frontend\nAngular SPA"]
    GOOGLE["Google OAuth 2.0"]

    subgraph AuthSystem["Wise Auth — Azure Web App 'eciwise-auth'"]
        API["REST API\nNestJS 11 · Node 24 · Express\nhelmet · CORS · ValidationPipe\nglobal JwtAuthGuard + RolesGuard\nSwagger at /api/docs"]
        PUB["RabbitMQ Adapters\nNotificationsPublisherService (confirm channel)\nMessagingService (publish + consume)"]
    end

    subgraph Infra["Stateful infrastructure"]
        PG[("PostgreSQL\nPrisma 6 · schema owner")]
        REDIS[("Redis\ncache-manager + Keyv\nfallback: in-memory")]
        MQ["RabbitMQ\nexchanges: ia.requests · ia.results\nnotifications (topic)"]
    end

    NOTIF["Notifications Service"]
    IAW["IA Workers\nrendimiento · deserción"]

    FE -->|"HTTPS / REST\nBearer JWT"| API
    API <-->|"OAuth redirect + token exchange"| GOOGLE
    API -->|"reads / writes"| PG
    API -->|"verification codes\nstats cache"| REDIS
    API --> PUB
    PUB -->|"publish notifications.masivo\npublish ia.requests"| MQ
    MQ -->|"consume ia.results"| PUB
    MQ --> NOTIF
    MQ <--> IAW
```

| Container | Technology | Responsibility |
|---|---|---|
| REST API | NestJS 11, Node 24, Express | All HTTP endpoints, guards, validation, JWT signing |
| RabbitMQ adapters | `amqplib` | Outbound email + prediction requests; inbound prediction results |
| PostgreSQL | Prisma 6 | System of record for users, roles, states, AI data, assignments, flags |
| Redis | `cache-manager-redis-yet` + Keyv | Verification codes (HMAC'd) and statistics cache; degrades to memory |
| RabbitMQ | `amqplib` | Async integration with Notifications and the IA workers |

> **Redis is not optional in production.** Verification codes live in the cache. With the in-memory fallback a code issued by instance A cannot be verified by instance B, so multi-instance deployments require `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD`.

---

## C4 — Level 3: Components

Inside the NestJS process, by module. Arrows are compile-time dependencies (`imports` / injection).

```mermaid
graph TB
    subgraph AppModule["AppModule (root)"]
        CACHEMOD["CacheModule\nglobal · Redis or memory"]
    end

    subgraph AuthMod["AuthModule"]
        AC["AuthController\n/auth/*"]
        AS["AuthService\nregister · login · OAuth\npassword change / reset"]
        PS["PasswordService\nArgon2id · bcrypt legacy"]
        VCS["VerificationCodeService\n6-digit codes · HMAC · attempts"]
        GUARDS["Guards\nJwtAuthGuard · RolesGuard\nRateLimitGuard · GoogleAuthGuard"]
        STRATS["Strategies\nJwtStrategy · GoogleStrategy"]
    end

    subgraph GestionMod["GestionUsuariosModule"]
        GC["GestionUsuariosController\n/gestion-usuarios/*"]
        GS["GestionUsuariosService\ndirectory · profiles · CSV\nstats · role/status"]
    end

    subgraph IaMod["IaModule"]
        IC["IaController\n/ia/*"]
        IS["IaService\nAI data · predictions\ntutor assignments"]
    end

    subgraph FlagsMod["FeatureFlagsModule"]
        FC["FeatureFlagsController\n/feature-flags/*"]
        FS["FeatureFlagsService\nwhitelist FEATURE_KEYS"]
    end

    subgraph PersistMod["PersistenceModule"]
        REPOS["Prisma repositories\nUsuario · Rol · EstadoUsuario\nDatosIa · AsignacionTutor · FeatureFlag"]
    end

    subgraph MsgMod["MessagingModule"]
        MS["MessagingService\npublish ia.requests\nconsume ia.results"]
    end

    subgraph NotifMod["NotificationsModule"]
        NPS["NotificationsPublisherService\nconfirm channel · mandatory"]
    end

    subgraph InfraMod["Infrastructure"]
        CA["NestjsCacheAdapter"]
        PSVC["PrismaService"]
    end

    AC --> AS
    AC --> GUARDS
    AS --> PS
    AS --> VCS
    AS --> REPOS
    AS --> MS
    AS --> NPS
    AS --> CA
    VCS --> CA
    GUARDS --> STRATS

    GC --> GS
    GS --> REPOS
    GS --> NPS
    GS --> CA

    IC --> IS
    IS --> REPOS
    IS --> MS

    FC --> FS
    FS --> REPOS

    REPOS --> PSVC
    MS --> REPOS
    CA --> CACHEMOD
```

| Component | Role |
|---|---|
| `AuthService` | Orchestrates the two-step register, login, Google upsert, password flows |
| `PasswordService` | The only place that touches a crypto primitive — Argon2id + bcrypt legacy |
| `VerificationCodeService` | Issues/verifies one-time codes; stores only an HMAC, counts attempts |
| `GestionUsuariosService` | Admin + self-service user management, CSV bulk load, statistics |
| `IaService` | AI feature data, prediction triggering, tutor↔student assignments |
| `FeatureFlagsService` | Reads/writes `feature_flags`, rejecting keys outside `FEATURE_KEYS` |
| Prisma repositories | Implement the domain repository ports; the only code that knows Prisma |
| `MessagingService` | Implements `IPrediccionPublisher`; also the `ia.results` consumer |
| `NotificationsPublisherService` | Implements `INotificationPublisher` over a confirm channel |

---

## C4 — Level 4: Code

The hexagonal core. Application services depend **only** on the interfaces in `src/domain/ports`; adapters are bound by the string tokens in `INJECTION_TOKENS`, so no application class imports Prisma or `amqplib`.

```mermaid
classDiagram
    class AuthService {
        -IUsuarioRepository usuarioRepository
        -JwtService jwtService
        -PasswordService passwordService
        -VerificationCodeService verificationCodeService
        -IPrediccionPublisher prediccionPublisher
        -INotificationPublisher notificationPublisher
        -ICacheService cacheService
        +register(dto) PendingResponseDto
        +verifyEmail(dto) AuthResponseDto
        +resendRegistrationCode(email) SuccessResponseDto
        +login(dto) AuthResponseDto
        +changePassword(userId, dto) UserResponseDto
        +requestPasswordChange(userId, dto) PendingResponseDto
        +confirmPasswordChange(userId, dto) UserResponseDto
        +requestPasswordReset(email) SuccessResponseDto
        +resetPassword(dto) SuccessResponseDto
        +validateGoogleUser(dto) AuthResponseDto
    }

    class PasswordService {
        -string dummyHash
        +hash(plain) Promise~string~
        +verify(hash, plain) Promise~boolean~
        +needsRehash(hash) boolean
        +verifyDummy(plain) Promise~void~
    }

    class VerificationCodeService {
        -ICacheService cache
        +ttlMinutes number
        +issue(scopeKey, payload) Promise~string~
        +verify(scopeKey, code) Promise~T~
        +peekPayload(scopeKey) Promise~T~
        +discard(scopeKey) Promise~void~
        -hash(code) string
        -matches(storedHash, code) boolean
    }

    class IUsuarioRepository {
        <<interface>>
        +findByEmail(email)
        +findById(id)
        +findFirst(criteria)
        +create(data)
        +update(id, data)
    }

    class IPrediccionPublisher {
        <<interface>>
        +publishPrediction(routingKey, payload) boolean
    }

    class INotificationPublisher {
        <<interface>>
        +publishAccountsCreated(accounts) Promise~number~
        +publishVerificationCode(input) Promise~boolean~
    }

    class ICacheService {
        <<interface>>
        +get(key) Promise~T~
        +set(key, value, ttlMs) Promise~void~
        +del(key) Promise~void~
    }

    class PrismaUsuarioRepository
    class MessagingService
    class NotificationsPublisherService
    class NestjsCacheAdapter

    AuthService ..> IUsuarioRepository : IUsuarioRepository token
    AuthService ..> IPrediccionPublisher : IPrediccionPublisher token
    AuthService ..> INotificationPublisher : INotificationPublisher token
    AuthService ..> ICacheService : ICacheService token
    AuthService --> PasswordService
    AuthService --> VerificationCodeService
    VerificationCodeService ..> ICacheService

    IUsuarioRepository <|.. PrismaUsuarioRepository
    IPrediccionPublisher <|.. MessagingService
    INotificationPublisher <|.. NotificationsPublisherService
    ICacheService <|.. NestjsCacheAdapter
```

### Ports and their adapters

| Port (`src/domain/ports`) | Token | Adapter |
|---|---|---|
| `IUsuarioRepository` | `IUsuarioRepository` | `PrismaUsuarioRepository` |
| `IRolRepository` | `IRolRepository` | `PrismaRolRepository` |
| `IEstadoUsuarioRepository` | `IEstadoUsuarioRepository` | `PrismaEstadoUsuarioRepository` |
| `IDatosIaRepository` | `IDatosIaRepository` | `PrismaDatosIaRepository` |
| `IAsignacionTutorRepository` | `IAsignacionTutorRepository` | `PrismaAsignacionTutorRepository` |
| `IFeatureFlagRepository` | `IFeatureFlagRepository` | `PrismaFeatureFlagRepository` |
| `IPrediccionPublisher` | `IPrediccionPublisher` | `MessagingService` (RabbitMQ) |
| `INotificationPublisher` | `INotificationPublisher` | `NotificationsPublisherService` (RabbitMQ) |
| `ICacheService` | `ICacheService` | `NestjsCacheAdapter` (Redis / memory) |

---

## Authentication Flows

### Registration — two steps, code-verified

The account is **not** created on `POST /auth/register`. The pending registration (including the already-hashed password) is parked in the cache behind a 6-digit code; the row is only written when the code is confirmed. This avoids orphan accounts for addresses that are never verified.

```mermaid
sequenceDiagram
    participant C as Client
    participant Ctrl as AuthController
    participant S as AuthService
    participant VCS as VerificationCodeService
    participant Cache as Redis
    participant MQ as RabbitMQ → Notifications
    participant DB as PostgreSQL

    C->>Ctrl: POST /auth/register { nombre, apellido, email, password, datosIa? }
    Ctrl->>S: register(dto)
    S->>S: isAllowedEmailDomain(email)?
    S->>DB: findByEmail(email)
    DB-->>S: null
    S->>S: passwordService.hash(password) → Argon2id
    S->>VCS: issue("reg:pending:<email>", pending)
    VCS->>Cache: set HMAC(code) + payload, TTL 10 min
    VCS-->>S: code (plaintext, for the email)
    S->>MQ: publishVerificationCode(...)
    MQ-->>S: broker ACK (else 503 verification_email_failed)
    S-->>C: 200 { pending: true, email }

    C->>Ctrl: POST /auth/register/verify { email, code }
    Ctrl->>S: verifyEmail(dto)
    S->>VCS: verify("reg:pending:<email>", code)
    VCS->>Cache: get + compare HMAC (timing-safe)
    VCS-->>S: pending payload (code consumed)
    S->>DB: create Usuario { rolId: 1, estadoId: 1 }
    DB-->>S: saved user
    S->>S: invalidate statistics caches
    opt datosIa provided
        S->>MQ: publish ia.requests (rendimiento)
    end
    S-->>C: 201 { access_token, user }
```

`POST /auth/register/resend` re-issues a code for the same pending payload and always answers `{ success: true }`, so it never reveals whether a registration is in flight.

### Login

```mermaid
sequenceDiagram
    participant C as Client
    participant R as RateLimitGuard
    participant S as AuthService
    participant PS as PasswordService
    participant DB as PostgreSQL

    C->>R: POST /auth/login { email, password }
    R-->>S: pass (≤ 5 req/min per IP+path)
    S->>DB: findByEmail(email)

    alt user not found or OAuth-only (no password)
        S->>PS: verifyDummy(password)
        note over S,PS: equal-cost decoy hash → constant response time
        S-->>C: 401 invalid_credentials
    else user exists
        S->>PS: verify(storedHash, password)
        alt invalid
            S-->>C: 401 invalid_credentials
        end
        alt estado = suspendido / inactivo
            S-->>C: 400 account_suspended / account_inactive
        end
        opt needsRehash(storedHash)
            S->>PS: hash(password) → Argon2id
            note over S,DB: rehash-on-login — silent bcrypt upgrade
        end
        S->>DB: update { ultimo_login, password? }
        S-->>C: 200 { access_token, user }
    end
```

### Google OAuth 2.0

```mermaid
sequenceDiagram
    participant C as Browser
    participant Ctrl as AuthController
    participant G as Google OAuth API
    participant S as AuthService
    participant DB as PostgreSQL

    C->>Ctrl: GET /auth/google
    Ctrl->>G: redirect to consent screen
    G-->>C: user grants permission
    C->>Ctrl: GET /auth/google/callback?code=…
    Ctrl->>G: exchange code for profile
    G-->>Ctrl: { googleId, email, nombre, apellido, avatarUrl }

    alt domain not allowed
        Ctrl-->>C: 307 → /auth/callback#error=email_domain_not_allowed
    end

    Ctrl->>S: validateGoogleUser(dto)
    S->>DB: findFirst by google_id or email

    alt existing user, no google_id
        S->>DB: link account (google_id, avatar_url, ultimo_login)
    else existing user
        S->>DB: update ultimo_login, avatar_url
    else new user
        S->>DB: create Usuario { rolId: 1, estadoId: 1 }
    end

    S-->>Ctrl: { access_token, user }
    Ctrl-->>C: 307 → /auth/callback#token=JWT&user={…}

    note over C,Ctrl: Data travels in the URL fragment (#), never the query string —<br/>so the JWT is not written to access logs or leaked via Referer
```

Google sign-in issues **no verification code**: Google has already proven ownership of the address.

### Password change and reset

Three distinct flows share `VerificationCodeService` under different scope keys:

```mermaid
flowchart TD
    subgraph Forced["Forced change — CSV accounts"]
        F1["POST /auth/cambiar-contrasena\nJWT · mustChangePassword=true\nskips currentPassword check"]
        F1 --> F2["password updated\nmustChangePassword=false"]
    end

    subgraph Profile["From the profile — authenticated"]
        P1["POST /auth/cambiar-contrasena/solicitar\nverify currentPassword\nhash new password"]
        P1 --> P2["code → pwd:change:userId\nemail sent"]
        P2 --> P3["POST /auth/cambiar-contrasena/confirmar\ncode → apply stored hash"]
    end

    subgraph Forgot["Forgot password — public"]
        G1["POST /auth/olvido-contrasena\nalways { success: true }"]
        G1 --> G2["code → pwd:reset:email\nonly if account has a password"]
        G2 --> G3["POST /auth/restablecer-contrasena\ncode + newPassword"]
    end
```

| Flow | Scope key | Auth | Anti-enumeration |
|---|---|---|---|
| Forced change | — | JWT | n/a |
| Profile change | `pwd:change:<userId>` | JWT | n/a — identity already proven |
| Forgot password | `pwd:reset:<email>` | Public | Generic `{ success: true }` either way |

---

## Password Security

Passwords are hashed with **Argon2id** — the algorithm ranked first by the OWASP Password Storage Cheat Sheet — behind a single `PasswordService`. Argon2id is *memory-hard*, so it resists GPU/ASIC cracking far better than bcrypt. All hashing, verification, and rehash logic lives in one place; the rest of the service never touches a crypto primitive directly.

### Hashing parameters

| Parameter | Value | Rationale |
|---|---|---|
| Algorithm | Argon2id | Memory-hard, OWASP first choice |
| Memory cost | 19 MiB (`19456`) | OWASP-recommended minimum |
| Iterations (time cost) | 2 | Balanced with the memory cost |
| Parallelism | 1 | Single lane, deterministic cost |
| Library | `@node-rs/argon2` | NAPI prebuilds — works with the image's `npm ci --ignore-scripts` |

### Transparent migration from bcrypt

The service previously used bcrypt (cost 12). Rather than force a password reset, legacy hashes are migrated **silently on login**:

- `PasswordService.verify` detects the algorithm by prefix — `$argon2…` uses Argon2id, `$2a/$2b/$2y…` falls back to bcrypt.
- After a successful login, `needsRehash` reports any hash that is not `$argon2id$`; the password is re-hashed and persisted in the same `update` that writes `ultimo_login` (**rehash-on-login**). Each user is upgraded the next time they authenticate.

### Anti-enumeration (constant-time login)

When the email does not exist — or belongs to an OAuth-only account with no password — the login still performs a **dummy Argon2id verification of equal cost** (`verifyDummy`, against a decoy hash precomputed at startup) before returning `401`. This removes the timing side channel that would otherwise let an attacker discover which emails are registered.

> Design rationale and trade-offs are recorded in [ADR-010 — Password Hashing Migration from bcrypt to Argon2id](/docs/architecture-decisions/#adr-010--wise_auth-password-hashing-migration-from-bcrypt-to-argon2id).

---

## Verification Codes

One mechanism serves registration, password change and password reset.

```mermaid
flowchart TD
    Issue["issue(scopeKey, payload?)"]
    Gen["randomInt → 6 digits"]
    Store["cache.set(verify:scopeKey,\n{ HMAC-SHA256(code), attempts: 0,\nexpiresAt, payload }, TTL)"]
    Mail["plaintext code → email only"]

    Issue --> Gen --> Store
    Gen --> Mail

    Verify["verify(scopeKey, code)"]
    Lookup{"entry exists\nand not expired?"}
    Match{"timingSafeEqual\nHMAC match?"}
    Attempts{"attempts + 1\n>= max (5)?"}
    OK["delete entry\nreturn payload"]
    Expired["400 code_expired"]
    TooMany["delete entry\n400 too_many_attempts"]
    Invalid["rewrite with remaining TTL\n400 invalid_code"]

    Verify --> Lookup
    Lookup -->|no| Expired
    Lookup -->|yes| Match
    Match -->|yes| OK
    Match -->|no| Attempts
    Attempts -->|yes| TooMany
    Attempts -->|no| Invalid
```

Design points:

- **Only an HMAC is stored**, never the code. With just 10⁶ possible codes, a bare SHA-256 would be reversible instantly by precomputing every hash; HMAC with a server-side secret (`VERIFICATION_CODE_SECRET`, falling back to `JWT_SECRET`) means leaking the cache is not enough.
- **Timing-safe comparison** (`timingSafeEqual`) on the derived hashes.
- **Attempt cap** (default 5) consumes the code, and failed attempts keep the *remaining* TTL rather than resetting it.
- **Single use** — a correct code is deleted before its payload is returned.

| Scope key | Purpose | Payload stored |
|---|---|---|
| `reg:pending:<email>` | Registration | Full pending registration + password hash |
| `pwd:change:<userId>` | Profile password change | New password hash |
| `pwd:reset:<email>` | Forgot password | — (none) |

---

## JWT Flow Across Services

```mermaid
flowchart LR
    AuthSvc["Auth Service\nissues JWT (HS256)"]
    JWT(["JWT\nsub · email · nombre\napellido · rol"])

    AuthSvc -->|signs with JWT_SECRET| JWT

    JWT -->|"Bearer token"| TutoringSvc["Tutoring Service\nvalidates locally"]
    JWT -->|"Bearer token"| MaterialsSvc["Materials Service\nvalidates locally"]
    JWT -->|"Bearer token"| TalkSvc["Talk Service\nJwtAuthFilter"]
    JWT -->|"x-jwt-token header"| GamifSvc["Gamification Service\nRabbitMQ header"]
    JWT -->|"?token= query param"| GameSvc["Game Service\nWebSocket upgrade"]

    note1["No service makes an HTTP call\nback to Auth during normal requests.\nAll validate the shared HS256 secret locally."]
```

### JWT Claims

| Claim | Type | Description |
|---|---|---|
| `sub` | string (UUID) | User identifier |
| `email` | string | User email address |
| `nombre` | string | First name |
| `apellido` | string | Last name |
| `rol` | string | `estudiante`, `tutor`, or `admin` |

The login/register response carries more than the token claims — `avatarUrl`, `programaPrincipal`, `programaSecundario`, `mustChangePassword` and `hasPassword` are returned in the `user` object so the frontend can route (e.g. force the password-change screen) without a second call.

---

## Role & Guard System

`JwtAuthGuard` and `RolesGuard` are registered **globally** in `main.ts`, so every route is authenticated unless it opts out with `@Public()`.

```mermaid
flowchart TD
    Request["Incoming Request"]
    Helmet["helmet + CORS\n(origin: FRONTEND_URL)"]
    Validation["ValidationPipe\nwhitelist · forbidNonWhitelisted"]
    RateLimit{"RateLimitGuard\non the route?"}
    Limit{"≤ 5 req/min\nper IP + path?"}
    JwtGuard{"JwtAuthGuard\n(global)"}
    Public{"@Public()\ndecorator?"}
    ValidToken{"Valid JWT?"}
    RolesDecorator{"@Roles(…)\non handler?"}
    RolesGuard{"RolesGuard\nchecks JWT rol claim"}
    Handler["Route Handler"]
    Reject401["401 Unauthorized"]
    Reject403["403 Forbidden"]
    Reject429["429 too_many_requests"]

    Request --> Helmet --> Validation --> RateLimit
    RateLimit -->|yes| Limit
    RateLimit -->|no| JwtGuard
    Limit -->|no| Reject429
    Limit -->|yes| JwtGuard
    JwtGuard --> Public
    Public -->|yes| Handler
    Public -->|no| ValidToken
    ValidToken -->|no| Reject401
    ValidToken -->|yes| RolesDecorator
    RolesDecorator -->|no| Handler
    RolesDecorator -->|yes| RolesGuard
    RolesGuard -->|rol matches| Handler
    RolesGuard -->|rol mismatch| Reject403
```

`RateLimitGuard` is **in-memory and per instance** (5 requests / 60 s, keyed by IP + path, with eviction above 10 000 entries). It is a speed bump against brute force, not a distributed limiter — Azure API Management provides the global rate limiting at the edge.

---

## Package Structure

```mermaid
graph TD
    AppModule["AppModule\n(root)"]
    AppModule --> CacheModule["CacheModule (global)\nRedis or in-memory"]
    AppModule --> AuthModule["AuthModule"]
    AppModule --> PrismaModule["PrismaModule"]
    AppModule --> GestionModule["GestionUsuariosModule"]
    AppModule --> IaModule["IaModule"]
    AppModule --> FlagsModule["FeatureFlagsModule"]

    AuthModule --> AC["auth.controller.ts"]
    AuthModule --> AS["auth.service.ts"]
    AuthModule --> PW["password.service.ts\nArgon2id · bcrypt legacy"]
    AuthModule --> VC["verification-code.service.ts\nHMAC codes"]
    AuthModule --> G["guards/\nJwtAuthGuard · RolesGuard\nRateLimitGuard · GoogleAuthGuard"]
    AuthModule --> ST["strategies/\nJwtStrategy · GoogleStrategy"]
    AuthModule --> DEC["decorators/\n@Public · @Roles · @GetUser"]
    AuthModule --> DTO["dto/\nRegisterDto · LoginDto · VerifyEmailDto\nResetPasswordDto · AuthResponseDto…"]
    AuthModule --> EN["enums/\nRole · EstadoUsuario · TipoToken"]

    Domain["domain/\nports/repositories · ports/outbound\ntokens/injection-tokens.ts"]
    Infra["infrastructure/\npersistence/ (6 Prisma repositories)\ncache/nestjs-cache.adapter.ts"]
    Msg["messaging/\nMessagingService · constants"]
    Notif["notifications/\nNotificationsPublisherService"]
    Docs["docs/\nswagger.ts · *.docs.ts"]
    Config["config/\nenvs.ts (Joi)"]

    AuthModule --> Domain
    AuthModule --> Msg
    AuthModule --> Notif
    GestionModule --> Notif
    IaModule --> Msg
    Domain -.->|implemented by| Infra
    AppModule --> Config
    AppModule --> Docs
```

---

## Data Model

Roles and states are **tables, not enums** — they are seeded rows referenced by `rol_id` / `estado_id`, so they can be extended without a migration of the `usuarios` table.

```mermaid
erDiagram
    Rol ||--o{ Usuario : "clasifica"
    EstadoUsuario ||--o{ Usuario : "habilita"
    Usuario ||--o| DatosIaEstudiante : "tiene"
    Usuario ||--o{ AsignacionTutor : "como tutor"
    Usuario ||--o{ AsignacionTutor : "como estudiante"

    Usuario {
        uuid id PK
        string email UK
        string nombre
        string apellido
        string telefono
        string biografia
        int semestre
        string programa_principal
        string programa_secundario
        string password "nullable — OAuth-only accounts"
        string google_id UK
        string avatar_url
        int rol_id FK
        int estado_id FK
        datetime ultimo_login
        boolean must_change_password
        json disponibilidad
        datetime created_at
        datetime updated_at
    }

    Rol {
        int id PK
        string nombre UK "estudiante · tutor · admin"
        string descripcion
        boolean activo
    }

    EstadoUsuario {
        int id PK
        string nombre UK "activo · inactivo · suspendido"
        string descripcion
        boolean activo
    }

    DatosIaEstudiante {
        uuid id PK
        uuid usuario_id FK,UK
        int gender
        float study_time_weekly
        int absences
        string prediccion_rendimiento
        string prediccion_desercion
        float confianza_desercion
        float probabilidad_exito
        datetime fecha_prediccion
    }

    AsignacionTutor {
        uuid id PK
        uuid tutor_id FK
        uuid estudiante_id FK
        datetime created_at
    }

    FeatureFlag {
        string key PK
        boolean enabled
        datetime updated_at
    }
```

- `DatosIaEstudiante` holds ~30 optional model-input columns (only a representative subset is shown) plus the last stored prediction. Fields are nullable to allow progressive filling: a small subset at registration, the rest in the dedicated form.
- `AsignacionTutor` is unique on `(tutor_id, estudiante_id)` and drives what a tutor sees. An **admin sees every student without any assignment row**.
- `FeatureFlag` has no relations — the absence of a row means *enabled*, so switching a feature off is opt-in.

### Role progression

```mermaid
graph LR
    estudiante["estudiante\n(rol_id = 1, default)"]
    tutor["tutor\n(assigned by admin)"]
    admin["admin\n(full access)"]
    estudiante -->|"PATCH /gestion-usuarios/:id/rol"| tutor
    tutor -->|"PATCH /gestion-usuarios/:id/rol"| admin
```

---

## AI Prediction Integration

Auth owns the student data the models consume, so it is both the **producer** of prediction requests and the **consumer** of their results.

```mermaid
sequenceDiagram
    participant S as AuthService / IaService
    participant MQ as RabbitMQ
    participant W as IA Worker (Python)
    participant R as PrismaDatosIaRepository

    S->>MQ: publish ia.requests (rk: rendimiento | desercion)
    note over MQ: direct exchange · durable queues<br/>ia.rendimiento.requests · ia.desercion.requests
    MQ->>W: consume request { usuarioId, studentName, features }
    W->>W: run model
    W->>MQ: publish ia.results (rk: rendimiento | desercion)
    MQ->>S: consume (queue ia.results, prefetch 1)
    S->>R: upsert(usuarioId, { prediccion…, fechaPrediccion })
    note over S,R: ack on success · nack(requeue=false) on error
```

| Exchange | Type | Routing keys | Queues |
|---|---|---|---|
| `ia.requests` | direct | `rendimiento`, `desercion` | `ia.rendimiento.requests`, `ia.desercion.requests` |
| `ia.results` | direct | `rendimiento`, `desercion` | `ia.results` |
| `notifications` | topic | `notification.masivo` | `notification_masivo` |

Both adapters reconnect on a 5 s timer. The notifications publisher additionally uses a **confirm channel** with `mandatory: true`: `publishVerificationCode` waits for the broker ACK and returns `false` if it never comes, which `AuthService` turns into `503 verification_email_failed` — the user is never told a code was sent when it was not. See [IA Predictions](./ia-predictions.md) and [Notifications Service](./notifications-service.md).

---

## Endpoints

### Auth — public (no JWT)

All are rate-limited at 5 req/min per IP + path.

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/register` | Step 1 — validate, park pending registration, email a code → `{ pending, email }` |
| `POST` | `/auth/register/verify` | Step 2 — confirm code, create account, return JWT (201) |
| `POST` | `/auth/register/resend` | Re-send the pending code → always `{ success: true }` |
| `POST` | `/auth/login` | Authenticate with email + password → JWT |
| `POST` | `/auth/olvido-contrasena` | Request a reset code → always `{ success: true }` |
| `POST` | `/auth/restablecer-contrasena` | Confirm code + set the new password |
| `GET` | `/auth/google` | Start the Google OAuth 2.0 flow |
| `GET` | `/auth/google/callback` | Google callback → 307 to the frontend with the JWT in the fragment |

### Auth — protected (JWT)

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/cambiar-contrasena` | Change password (forced flow for CSV accounts) |
| `POST` | `/auth/cambiar-contrasena/solicitar` | Step 1 — verify current password, email a code |
| `POST` | `/auth/cambiar-contrasena/confirmar` | Step 2 — confirm the code and apply |

### Gestión de Usuarios (`/gestion-usuarios`)

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/` | admin | Filtered user list |
| `GET` | `/me` | any | Own profile |
| `GET` | `/directorio` | any | Lightweight directory (public fields only) for participant pickers |
| `POST` | `/carga-masiva` | admin | CSV bulk load; emails temporary passwords |
| `PATCH` | `/:id/rol` | admin | Change a user's role |
| `PATCH` | `/:id/estado` | admin | Change a user's status |
| `PATCH` | `/me/info-personal` | any | Update own personal info |
| `DELETE` | `/:id` | admin | Delete a user |
| `DELETE` | `/me/cuenta` | any | Delete own account |
| `GET` | `/estadisticas/usuarios` | admin | User statistics (cached) |
| `GET` | `/estadisticas/roles` | admin | Per-role statistics (cached) |
| `GET` | `/estadisticas/crecimiento` | admin | Growth over time |

### IA (`/ia`)

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/me` | estudiante | Own AI data |
| `PUT` | `/me` | estudiante | Update own AI data |
| `PUT` | `/me/prediccion` | estudiante | Update data and trigger a prediction |
| `GET` | `/estudiantes` | tutor · admin | Students (assigned ones for a tutor; all for an admin) |
| `GET` | `/estudiantes/:id` | tutor · admin | One student's AI detail |
| `GET` | `/metricas` | tutor · admin | Aggregated metrics |
| `GET` | `/estadisticas` | admin | Global AI statistics |
| `GET` | `/asignaciones` | admin | List tutor↔student assignments |
| `POST` | `/asignaciones` | admin | Create an assignment |
| `DELETE` | `/asignaciones/:id` | admin | Remove an assignment |

### Feature Flags (`/feature-flags`)

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/` | public | Current state of every flag — read by the frontend at boot |
| `PUT` | `/:key` | admin | Toggle one flag |

Valid keys: `tutorias`, `materials`, `games`, `practica`, `study`, `tasks`, `chat`, `ia`, `gamification`. Anything else is rejected on write and ignored on read.

### Register request body

```json
{
  "nombre": "Daniel",
  "apellido": "Useche",
  "email": "daniel@mail.escuelaing.edu.co",
  "password": "s3cur3P@ss"
}
```

```json
{ "pending": true, "email": "daniel@mail.escuelaing.edu.co" }
```

### Login response

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "user": {
    "id": "uuid",
    "email": "daniel@mail.escuelaing.edu.co",
    "nombre": "Daniel",
    "apellido": "Useche",
    "rol": "estudiante",
    "avatarUrl": null,
    "mustChangePassword": false,
    "hasPassword": true
  }
}
```

### Error codes

Errors are returned as **machine-readable codes**, not prose — the frontend owns the wording.

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `email_domain_not_allowed` | Address outside the institutional / Gmail allowlist |
| 400 | `code_expired` | No pending code, or it expired |
| 400 | `invalid_code` | Wrong code (attempt counted) |
| 400 | `too_many_attempts` | Attempt cap reached; the code is burned |
| 400 | `account_suspended` / `account_inactive` | Login blocked by user state |
| 400 | `current_password_required` | Missing current password on a non-forced change |
| 401 | `invalid_credentials` | Unknown email or wrong password (constant time) |
| 401 | `invalid_current_password` | Current password did not match |
| 403 | `password_change_not_allowed_for_oauth_users` | OAuth-only account has no password |
| 404 | `user_not_found` | — |
| 409 | `email_taken` | Email already registered |
| 429 | `too_many_requests` | Rate limit exceeded on a public endpoint |
| 503 | `verification_email_failed` | Broker never confirmed the verification email |

---

## Deployment

```mermaid
graph TB
    subgraph GH["GitHub"]
        REPO["EciWise/wise_auth\nmain"]
        WF["Actions\nmain_eciwise-auth.yml\nnpm ci · build · zip"]
        CI["PR CI\nlint · tsc · unit tests\nSonar · security"]
    end

    subgraph Azure["Azure"]
        WA["Web App 'eciwise-auth'\nNode 24 · Production slot\nSCM_DO_BUILD_DURING_DEPLOYMENT=false"]
        APIM["API Management\nglobal rate limiting"]
        PGA[("Azure PostgreSQL")]
        REDISA[("Azure Cache for Redis\nTLS")]
    end

    MQ["RabbitMQ"]
    GHCR["GHCR image\ndocker-publish.yml"]

    REPO --> CI
    REPO --> WF
    WF -->|"OIDC login · deploy zip"| WA
    REPO --> GHCR
    APIM --> WA
    WA --> PGA
    WA --> REDISA
    WA <--> MQ
```

### Environment Variables

| Variable | Required | Example | Purpose |
|---|---|---|---|
| `PORT` | Yes | `3000` | HTTP port |
| `DATABASE_URL` | Yes | `postgresql://…` | PostgreSQL connection (Prisma runtime) |
| `DIRECT_URL` | Yes | `postgresql://…` | Direct connection (Prisma migrations) |
| `JWT_SECRET` | Yes | 32+ chars | HS256 signing secret — shared with every service |
| `JWT_EXPIRATION` | Yes | `7d` | Token TTL |
| `GOOGLE_CLIENT_ID` | Yes | `123.apps.googleusercontent.com` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Yes | `GOCSPX-…` | Google OAuth client secret |
| `GOOGLE_CALLBACK_URL` | Yes | `http://localhost:3000/auth/google/callback` | OAuth redirect URI (HTTPS enforced in production) |
| `FRONTEND_URL` | Yes | `http://localhost:4200` | CORS origin and post-auth redirect target |
| `RABBITMQ_URL` | No | `amqp://guest:guest@localhost:5672` | Broker for notifications + IA |
| `REDIS_HOST` | No* | `xxx.redis.cache.windows.net` | Cache host — falls back to memory |
| `REDIS_PORT` | No* | `6380` | Cache port (TLS) |
| `REDIS_PASSWORD` | No* | — | Cache password |
| `VERIFICATION_CODE_TTL_MIN` | No | `10` | Verification code validity in minutes |
| `VERIFICATION_MAX_ATTEMPTS` | No | `5` | Failed attempts before the code is burned |
| `VERIFICATION_CODE_SECRET` | No | 32+ chars | HMAC pepper for codes — defaults to `JWT_SECRET` |

\* All three Redis variables are required **together**; if any is missing the cache silently degrades to in-memory, which breaks verification codes across multiple instances. Configuration is validated by Joi at boot (`src/config/envs.ts`) — the process refuses to start on a bad config.

### Local Execution

```bash
npm install
cp .env.template .env
npx prisma generate
npx prisma migrate deploy
npm run start:dev
```

### API Documentation

Swagger is served at `/api/docs` when the service is running.

---

## Further Reading

- Source repository: [EciWise/wise_auth](https://github.com/EciWise/wise_auth)
- [Actors, Roles & Permissions](/docs/actors-roles-permissions/) — the platform-wide role matrix
- [Security](/docs/security/) — comparative security components across services
- [IA Predictions](./ia-predictions.md) — the models behind `ia.requests` / `ia.results`
- [Notifications Service](./notifications-service.md) — the consumer of the emails Auth publishes
- Swagger: `/api/docs` at runtime
- Prisma schema: `prisma/schema.prisma`
</content>
</invoke>
