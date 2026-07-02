---
layout: default
title: Auth Service
---

# Auth Service

## Overview

`wise_auth` is the authentication and authorization microservice for the ECIWise platform. It handles user registration, email/password login, OAuth 2.0 sign-in via Google, and JWT issuance shared across all other services.

The service handles two core domains:

- **Identity management**: register and store users with hashed passwords, manage profile data (name, semester, avatar), and track login state.
- **Token issuance**: issue HS256 JWTs carrying the claims (`sub`, `email`, `nombre`, `apellido`, `rol`) consumed by every downstream service without a round-trip to Auth.

---

## System Architecture

```mermaid
graph TB
    subgraph Clients["Clients"]
        Browser["Browser / Frontend"]
        Services["Other Microservices\n(Tutoring, Materials, Talk…)"]
    end

    subgraph AuthService["Wise Auth — NestJS 11"]
        Controller["AuthController\n/auth/*"]
        Service["AuthService\nbusiness logic"]
        Password["PasswordService\nArgon2id (+ bcrypt legacy)"]
        subgraph Guards["Guards"]
            JwtGuard["JwtAuthGuard\nglobal"]
            RolesGuard["RolesGuard\nhandler-level"]
            RateLimit["RateLimitGuard\npublic endpoints"]
            GoogleGuard["GoogleAuthGuard\nOAuth flow"]
        end
        subgraph Strategies["Passport Strategies"]
            JwtStrategy["JwtStrategy\nHS256 validation"]
            GoogleStrategy["GoogleStrategy\nOAuth 2.0"]
        end
        Prisma["PrismaService\nPostgreSQL"]
    end

    subgraph External["External"]
        Google["Google OAuth 2.0\nAPI"]
        DB[("PostgreSQL\nusuarios table")]
    end

    Browser -->|"POST /auth/register\nPOST /auth/login"| Controller
    Browser -->|"GET /auth/google"| Controller
    Services -->|"Bearer JWT\n(validated locally, no HTTP call)"| JwtGuard
    Controller --> Guards
    Guards --> Service
    Service --> Password
    Service --> Strategies
    GoogleStrategy <-->|OAuth redirect| Google
    Service --> Prisma
    Prisma --> DB
```

---

## Authentication Flows

### Email / Password Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant R as RateLimitGuard
    participant Ctrl as AuthController
    participant S as AuthService
    participant P as Prisma (PostgreSQL)

    C->>R: POST /auth/register { nombre, apellido, email, password }
    R-->>Ctrl: pass (under limit)
    Ctrl->>S: register(dto)
    S->>P: find user by email
    P-->>S: null (not found)
    S->>S: passwordService.hash(password) → Argon2id
    S->>P: create Usuario { rol: estudiante }
    P-->>S: saved user
    S->>S: jwt.sign({ sub, email, nombre, apellido, rol })
    S-->>Ctrl: AuthResponseDto { access_token, user }
    Ctrl-->>C: 201 Created + JWT

    note over C,P: Login follows the same path but verifies with Argon2id (bcrypt kept for legacy hashes) and re-hashes on the way in — see Password Security below
```

### Google OAuth 2.0 Flow

```mermaid
sequenceDiagram
    participant C as Browser
    participant Ctrl as AuthController
    participant G as Google OAuth API
    participant S as AuthService
    participant P as PostgreSQL

    C->>Ctrl: GET /auth/google
    Ctrl->>G: redirect to Google consent screen
    G-->>C: user grants permission
    C->>Ctrl: GET /auth/google/callback?code=…
    Ctrl->>G: exchange code for profile
    G-->>Ctrl: { googleId, email, nombre, apellido, avatarUrl }

    alt Domain not allowed
        Ctrl-->>C: redirect /auth/callback#error=email_domain_not_allowed
    end

    Ctrl->>S: validateGoogleUser(googleUserDto)
    S->>P: upsert Usuario by google_id / email
    P-->>S: user record
    S->>S: jwt.sign({ sub, email, nombre, apellido, rol })
    S-->>Ctrl: { access_token, user }
    Ctrl-->>C: HTTP 307 → /auth/callback#token=JWT&user={…}

    note over C,Ctrl: Token is in URL fragment (#), not query string — never logged by servers
```

---

## Password Security

Passwords are hashed with **Argon2id** — the algorithm ranked first by the OWASP Password Storage Cheat Sheet — behind a single `PasswordService`. Argon2id is *memory-hard*, so it resists GPU/ASIC cracking far better than bcrypt. All hashing, verification, and rehash logic lives in one place; the rest of the service never touches a crypto primitive directly.

### Hashing parameters

| Parameter | Value | Rationale |
|---|---|---|
| Algorithm | Argon2id | Memory-hard, OWASP first choice |
| Memory cost | 19 MiB | OWASP-recommended minimum |
| Iterations (time cost) | 2 | Balanced with the memory cost |
| Parallelism | 1 | Single lane, deterministic cost |
| Library | `@node-rs/argon2` | NAPI prebuilds — works with the image's `npm ci --ignore-scripts` |

### Transparent migration from bcrypt

The service previously used bcrypt (cost 12). Rather than force a password reset, legacy hashes are migrated **silently on login**:

- `PasswordService.verify` detects the algorithm by prefix — `$argon2…` uses Argon2id, `$2a/$2b/$2y…` falls back to bcrypt.
- After a successful login, if the stored hash is not Argon2id, the password is re-hashed with Argon2id and persisted (**rehash-on-login**). Each user is upgraded the next time they authenticate.

### Anti-enumeration (constant-time login)

When the email does not exist — or belongs to an OAuth-only account with no password — the login still performs a **dummy Argon2id verification of equal cost** before returning `401`. This removes the timing side channel that would otherwise let an attacker discover which emails are registered.

```mermaid
sequenceDiagram
    participant C as Client
    participant S as AuthService
    participant PS as PasswordService
    participant P as PostgreSQL

    C->>S: POST /auth/login { email, password }
    S->>P: findByEmail(email)
    alt user not found / OAuth-only
        S->>PS: verifyDummy(password)
        note over S,PS: equal-cost decoy hash → constant response time
        S-->>C: 401 invalid_credentials
    else user exists
        S->>PS: verify(storedHash, password)
        note over PS: argon2.verify, or bcrypt.compare for legacy hashes
        opt storedHash is not Argon2id
            S->>PS: hash(password) → argon2id
            S->>P: update password (silent upgrade)
        end
        S-->>C: 200 access_token + user
    end
```

> Design rationale and trade-offs are recorded in [ADR-010 — Password Hashing Migration from bcrypt to Argon2id](/docs/architecture-decisions/#adr-010--wise_auth-password-hashing-migration-from-bcrypt-to-argon2id).

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

---

## Role & Guard System

```mermaid
flowchart TD
    Request["Incoming Request"]
    JwtGuard{"JwtAuthGuard\n(global)"}
    Public{"@Public()\ndecorator?"}
    ValidToken{"Valid JWT?"}
    RolesDecorator{"@Roles(…)\non handler?"}
    RolesGuard{"RolesGuard\nchecks JWT rol claim"}
    Handler["Route Handler"]
    Reject401["401 Unauthorized"]
    Reject403["403 Forbidden"]

    Request --> JwtGuard
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

---

## Package Structure

```mermaid
graph TD
    AppModule["AppModule\n(root)"]
    AuthModule["AuthModule"]
    PrismaModule["PrismaModule"]
    Config["config/\nenvs.ts (Joi)"]

    AppModule --> AuthModule
    AppModule --> PrismaModule
    AppModule --> Config

    AuthModule --> Controller["auth.controller.ts\nroute handlers"]
    AuthModule --> Service["auth.service.ts\nbusiness logic"]
    AuthModule --> Password["password.service.ts\nArgon2id · bcrypt legacy\nrehash-on-login"]
    AuthModule --> Guards["guards/\nJwtAuthGuard\nRolesGuard\nRateLimitGuard\nGoogleAuthGuard"]
    AuthModule --> Strategies["strategies/\nJwtStrategy\nGoogleStrategy"]
    AuthModule --> Decorators["decorators/\n@Public  @Roles  @GetUser"]
    AuthModule --> DTOs["dto/\nRegisterDto  LoginDto\nChangePasswordDto\nAuthResponseDto"]
    AuthModule --> Enums["enums/\nRole  EstadoUsuario"]
```

---

## Data Model

```mermaid
erDiagram
    Usuario {
        string id PK
        string email UK
        string nombre
        string apellido
        string telefono
        int semestre
        string google_id UK
        string avatar_url
        RolEnum rol
        EstadoUsuario estado
        boolean email_verificado
        datetime ultimo_login
        datetime createdAt
        datetime updatedAt
    }
```

### Role Enum

```mermaid
graph LR
    estudiante["estudiante\n(default on register)"]
    tutor["tutor\n(assigned by admin)"]
    admin["admin\n(full access)"]
    estudiante -->|promoted| tutor
    tutor -->|promoted| admin
```

---

## JWT-based Identity

All services in ECIWise share the same HS256 secret. The Auth service issues the token; downstream services validate it locally without a network call to Auth.

### JWT Claims

| Claim | Type | Description |
|---|---|---|
| `sub` | string (UUID) | User identifier |
| `email` | string | User email address |
| `nombre` | string | First name |
| `apellido` | string | Last name |
| `rol` | string | `estudiante`, `tutor`, or `admin` |

---

## Endpoints

### Public (no JWT required)

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/register` | Create a student account (email + password) |
| `POST` | `/auth/login` | Authenticate with email + password; returns JWT |
| `GET` | `/auth/google` | Initiate Google OAuth 2.0 flow |
| `GET` | `/auth/google/callback` | Google callback; redirects frontend with JWT in URL fragment |

### Protected (JWT required)

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/cambiar-contrasena` | Change the authenticated user's password |

### Register request body

```json
{
  "nombre": "Daniel",
  "apellido": "Useche",
  "email": "daniel@mail.escuelaing.edu.co",
  "password": "s3cur3P@ss"
}
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
    "rol": "estudiante"
  }
}
```

### Error codes

| HTTP | Meaning |
|---|---|
| 400 | Validation error in request body |
| 401 | Invalid credentials |
| 403 | Role not permitted |
| 409 | Email already registered |
| 429 | Rate limit exceeded on public endpoint |

---

## Deployment

### Environment Variables

| Variable | Required | Example | Purpose |
|---|---|---|---|
| `PORT` | Yes | `3000` | HTTP port |
| `DATABASE_URL` | Yes | `postgresql://…` | PostgreSQL connection (Prisma runtime) |
| `DIRECT_URL` | Yes | `postgresql://…` | Direct connection (Prisma migrations) |
| `JWT_SECRET` | Yes | `supersecret32chars+` | HS256 signing secret |
| `JWT_EXPIRATION` | Yes | `7d` | Token TTL |
| `GOOGLE_CLIENT_ID` | Yes | `123.apps.googleusercontent.com` | Google OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | Yes | `GOCSPX-…` | Google OAuth client secret |
| `GOOGLE_CALLBACK_URL` | Yes | `http://localhost:3000/auth/google/callback` | OAuth redirect URI |
| `FRONTEND_URL` | Yes | `http://localhost:4200` | Frontend origin for post-auth redirect |

### Local Execution

```bash
npm install
cp .env.example .env
npx prisma generate
npx prisma migrate deploy
npm run start:dev
```

### API Documentation

Swagger is served at `/api/docs` when the service is running.

---

## Further Reading

- Source repository: [EciWise/wise_auth](https://github.com/EciWise/wise_auth)
- Swagger: `/api/docs` at runtime
- Prisma schema: `prisma/schema.prisma`
