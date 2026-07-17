---
layout: default
title: Game Service
---

# Game Service

## Overview

`game` is the real-time multiplayer game server for the ECIWise platform. It implements an agar.io-style game in which players move a circular cell around a shared world, absorb food to grow, and eliminate smaller players. The server is built with **Go** and **Fiber v2**, communicating exclusively over WebSocket.

The service handles four core concerns:

- **WebSocket management**: authenticated connections, per-connection read/write pumps, and graceful shutdown.
- **Room lifecycle**: rooms are created on demand, run at a fixed tick rate, and destroyed automatically when empty.
- **Game simulation**: spatial grid collision detection for food absorption and player-eating at O(N) cost per tick.
- **Security**: mandatory JWT authentication on every connection with rate limiting per IP.

---

## C4 — Level 1: System Context

```mermaid
graph TB
    subgraph Actors["Actors"]
        EST["Estudiante\nplays · splits · ejects"]
        OPS["Operations\npolls /health"]
    end

    GAME["Game Service\nReal-time multiplayer arena\nGo 1.22 · Fiber v2"]

    subgraph Ecosystem["ECIWISE+ Ecosystem"]
        FE["ECIWISE+ Frontend\nAngular SPA · canvas client"]
        AUTH["Auth Service\nissues the HS256 JWT"]
    end

    EST --> FE
    FE -->|"WSS /ws/game?token=JWT\nbidirectional game frames"| GAME
    FE -->|login| AUTH
    AUTH -.->|"shared JWT_SECRET\nvalidated locally — no HTTP call"| GAME
    OPS -->|"GET /health\nrooms · players"| GAME
```

### Actors

| Actor | Interaction |
|---|---|
| Estudiante | Connects over WebSocket, moves, splits, ejects mass, competes on the leaderboard |
| Operations | Polls `/health` for room and player counts — the only unauthenticated route |

### Neighbouring systems

| System | Relationship |
|---|---|
| Auth Service | Issues the JWT. Game validates it locally with `golang-jwt` on the upgrade request |
| Frontend | The only client; carries the token as a **query parameter** because browsers cannot set headers on a WebSocket handshake |

Game is **fully in-memory and stateless across restarts**: no database, no message broker, no calls to any other service. A restart drops every room — matches are ephemeral by design.

---

## C4 — Level 2: Containers

```mermaid
graph TB
    FE["ECIWISE+ Frontend\nAngular · canvas"]

    subgraph GameSystem["Game Service — single Go process"]
        HTTP["Fiber v2 HTTP server\nrecover · logger · CORS\nReadTimeout 15s · IdleTimeout 60s"]
        MW["Middleware chain\nrate limiter → upgrade check → JWT"]
        WSL["WebSocket layer\ngofiber/websocket v2\nGameHandler per connection"]
        HUB["game.Hub\nroom registry · RWMutex · WaitGroup"]
        ROOMS["game.Room goroutines\none Run() loop each · 30 fps"]
    end

    FE <-->|"WSS frames (JSON)"| HTTP
    HTTP --> MW
    MW --> WSL
    WSL -->|"Register / Unregister"| HUB
    HUB -->|"join · leave channels"| ROOMS
    ROOMS -->|"state broadcast"| WSL
```

| Container | Technology | Responsibility |
|---|---|---|
| HTTP server | Fiber v2 (fasthttp) | `/health` plus the WebSocket upgrade routes |
| Middleware chain | Fiber middleware | Rate limit → upgrade check → JWT, cheapest rejection first |
| WebSocket layer | `gofiber/websocket` | One `ReadPump` + one `WritePump` goroutine per client |
| Hub | Go stdlib | Room registry, player/colour counters, graceful drain |
| Rooms | Go goroutines | The simulation itself — one loop per room, 30 ticks/s |

There is **no persistence container**. All state lives in the room goroutines' stacks and heaps.

Two route prefixes are registered — `/ws/game` and `/game/ws/game` — so the service works both when exposed directly and when mounted behind a `/game` path prefix at the gateway.

---

## C4 — Level 3: Components

```mermaid
graph TB
    subgraph WsPkg["package ws"]
        JWTMW["JWTMiddleware\nreads ?token= · HMAC-SHA256\nstores claims in Locals"]
        GH["GameHandler\nper-connection lifecycle"]
    end

    subgraph ConfigPkg["package config"]
        CFG["Config\nPort · JWTSecret · FrontendURL\nWorld size · TickRateMs\nMaxPlayersPerRoom · FoodCount"]
    end

    subgraph GamePkg["package game"]
        HUB["Hub\nfindOrCreateRoom · Register\nUnregister · Shutdown · onRoomEmpty"]
        ROOM["Room\nRun() — owns ALL mutable state\njoin · leave · input channels"]
        CLIENT["Client\nReadPump · WritePump\nsend chan (buffer 64)"]
        PLAYER["Player + Cell\nsplit · merge · eject · powerups"]
        SGRID["SpatialGrid\n15x15 · Insert · QueryRadius"]
        FGRID["FoodGrid\n15x15 · Insert · QueryRadius"]
        FOOD["Food · Virus · PowerUp"]
        MSG["Message types\nMsgError · MovePayload\nPlayerSnapshot"]
        CLAIMS["JWTClaims\nsub · email · rol · nombre"]
    end

    CFG --> HUB
    JWTMW --> CLAIMS
    JWTMW --> GH
    GH -->|"NewClient + Register"| HUB
    GH -->|"go WritePump / ReadPump"| CLIENT
    HUB -->|"owns"| ROOM
    ROOM -->|"join / leave / input chans"| CLIENT
    ROOM --> PLAYER
    ROOM --> SGRID
    ROOM --> FGRID
    ROOM --> FOOD
    CLIENT --> MSG
    CLIENT --> CLAIMS
    PLAYER --> SGRID
```

| Component | Role |
|---|---|
| `ws.JWTMiddleware` | Validates the token **before** the upgrade; puts `*game.JWTClaims` in Fiber `Locals` |
| `ws.GameHandler` | Registers the client, starts `WritePump` as a goroutine, runs `ReadPump` inline, unregisters on return |
| `game.Hub` | Finds or creates a room per mode, tracks counts atomically, drains on shutdown |
| `game.Room` | The actor: its `Run()` goroutine exclusively owns the world state |
| `game.Client` | Socket I/O only — never touches world state directly |
| `game.SpatialGrid` / `FoodGrid` | 15×15 uniform bucketing that turns collision checks from O(N²) into ~O(N) |

### Game modes

| Mode | Query value | Timed |
|---|---|---|
| Classic | `classic` (default) | no |
| Pomodoro | `pomodoro` | yes |
| Battle Royale | `battleroyale` | yes |

`ParseMode` normalises anything unrecognised to `classic`, so a malformed query never rejects a connection.

---

## C4 — Level 4: Code

```mermaid
classDiagram
    class Hub {
        -sync.RWMutex mu
        -map~string,Room~ rooms
        -sync.WaitGroup wg
        -atomic.Int32 totalPlayers
        -atomic.Int32 colorCounter
        -Config cfg
        +Register(c, mode) void
        +Unregister(c) void
        +Shutdown() void
        +RoomCount() int
        +PlayerCount() int
        +ColorIndex() int
        -findOrCreateRoom(mode) Room
        -createRoom(mode) Room
        -onRoomEmpty(id) void
    }

    class Room {
        +string id
        +GameMode mode
        -chan join
        -chan leave
        -chan clientInput input
        +Run() void
    }

    class Client {
        -string playerID
        -Player player
        -Conn conn
        -JWTClaims claims
        -Room room
        -chan send
        -sync.Once closeOnce
        +SendJSON(v) void
        +ReadPump() void
        +WritePump() void
        -closeConn() void
    }

    class Player {
        +string ID
        +string Name
        +string Color
        +bool Sprinting
        +Spawn(worldW, worldH, owner) void
        +UpdateCells(dt, worldW, worldH, now) void
        +Split(now) void
        +Eject(nextID, worldW, worldH) []Food
        +Largest() Cell
        +TotalScore() int
        +Snapshots(now, out) []PlayerSnapshot
        +IsSprinting() bool
        +HasShield(now) bool
        -resolveCells(dt, now) void
        -mergeDelay(radius) Duration
        -speedFor(radius, now) float64
    }

    class Cell {
        +int ID
        +float64 X
        +float64 Y
        +float64 Radius
    }

    class SpatialGrid {
        +Clear() void
        +Insert(c, x, y) void
        +QueryRadius(x, y, radius) []Cell
    }

    class FoodGrid {
        +Clear() void
        +Insert(f, x, y) void
        +QueryRadius(x, y, radius) []Food
    }

    class JWTClaims {
        +string Sub
        +string Email
        +string Rol
        +string Nombre
        +string Apellido
        +DisplayName() string
    }

    Hub "1" --> "*" Room : owns
    Room "1" --> "*" Client : join/leave chans
    Client "1" --> "1" Player
    Client --> JWTClaims
    Player "1" --> "*" Cell
    Room --> SpatialGrid
    Room --> FoodGrid
```

### The concurrency contract

The design is **CSP / actor**, and the invariant is worth stating explicitly because it is what makes the server safe without locks on the hot path:

> All mutable world state lives exclusively inside `Room.Run()`. External goroutines never touch it — they communicate only through the `join`, `leave` and `input` channels.

Consequences visible in the code:

| Mechanism | Why |
|---|---|
| `Hub.Register` sets `client.room` **before** either pump starts | `ReadPump`'s deferred cleanup dereferences it |
| `Client.send` is buffered (64) and frames are **dropped** when full | A slow client must never block the 30 fps game loop |
| `closeOnce sync.Once` | `ReadPump` and `WritePump` can both reach cleanup; the socket closes once |
| `snapshotPool sync.Pool` | Recycles `[]PlayerSnapshot` arrays to cut per-tick heap allocation |
| `Hub.wg` (`WaitGroup`) | `Shutdown()` waits for every room goroutine to finish before Fiber stops |
| `atomic.Int32` counters | `/health` reads player/room counts without taking the hub lock |

Tuning constants live in `game/constants.go` — `InitialRadius` 22, `FoodRadius` 8, `MinEatRatio` 1.1 (a blob must be 10 % larger to absorb another), `BaseSpeed` 200 u/s decaying by `speed = BaseSpeed / (1 + radius/120)`, mass decay of 1.5 % every 5 ticks, and a 10-player leaderboard.

---

## Concurrency Model

```mermaid
graph LR
    subgraph Main["Main Process"]
        Main2["main goroutine\nFiber.Listen()"]
        Signal["signal goroutine\nSIGINT / SIGTERM"]
    end

    subgraph PerConn["Per WebSocket Connection"]
        HandlerG["Handler goroutine\nblocks on ReadPump"]
        WritePump["WritePump goroutine\ndrains send chan"]
    end

    subgraph PerRoom["Per Room"]
        RunLoop["Room.Run goroutine\nselect: join / leave / ticker"]
    end

    Main2 -->|"go signal handler"| Signal
    Signal -->|"hub.Shutdown()\nwg.Wait()"| Main2
    HandlerG -->|"go WritePump"| WritePump
    HandlerG -->|"input chan"| RunLoop
    HandlerG -->|"leave chan"| RunLoop
    RunLoop -->|"send chan"| WritePump
    Hub2["Hub.Register()\nRWMutex fast-path"] -->|"join chan"| RunLoop
```

---

## WebSocket Connection Lifecycle

```mermaid
sequenceDiagram
    participant B as Browser
    participant RL as Rate Limiter
    participant J as JWT Middleware
    participant H as Hub
    participant R as Room

    B->>RL: GET /ws/game (Upgrade: websocket) + ?token=JWT
    alt Rate limit reached
        RL-->>B: 429 Too Many Requests
    end
    RL->>J: validate JWT
    alt Token invalid or expired
        J-->>B: 401 Unauthorized
    end
    J->>H: Register(client)
    H->>R: join ← client
    R-->>B: {"type":"init", food:[...], playerId, color, worldWidth, worldHeight}
    loop Every 33 ms (30 fps)
        R-->>B: {"type":"state", players:[...], foodAdded:[...], foodRemoved:[...], tick}
    end
    B->>R: {"type":"move", "dx":0.5, "dy":-0.2}
    B->>R: close WebSocket
    R->>H: onRoomEmpty(id) — if room becomes empty
    Note over R: Room.Stop() → Run() returns → wg.Done()
```

---

## Room State Machine

```mermaid
stateDiagram-v2
    state "game.Room" as Room {
        [*] --> Waiting : newRoom()
        Waiting --> Active : first player joins
        Active --> Active : join / leave (count > 0)
        Active --> Full : playerCount == MaxPlayersPerRoom
        Full --> Active : player leaves
        Active --> Stopped : last player leaves\nonRoomEmpty() + Stop()
        Stopped --> [*] : Run() returns · wg.Done()
    }

    state "game.Player" as Player {
        [*] --> Alive : onJoin()
        Alive --> Dead : absorbed by larger player
        Alive --> Disconnected : WebSocket closed
        Dead --> [*] : DiedMsg sent · closeConn()
        Disconnected --> [*] : onLeave() called
    }
```

---

## Game Tick Flow (30 fps)

```mermaid
flowchart TD
    T(["ticker.C — every 33 ms"])
    D["Drain input channel\nupdate DX/DY for each player"]
    M["Move players\nPlayer.Update(dt)"]
    RG["Rebuild SpatialGrid\nRebuild FoodGrid"]
    FC["For each alive player\nFoodGrid.QueryRadius(x, y, radius)"]
    FH{"dist < radius?"}
    FA["AbsorbFood()\nspawn replacement\ninsert into FoodGrid"]
    PC["For each alive player\nSpatialGrid.QueryRadius(x, y, radius)"]
    PH{"CanEat(neighbor)\n&& dist < radius?"}
    PK["Mark victim.Alive = false\nrecord kill"]
    KP["Process kills\nAbsorbPlayer + DiedMsg\ncloseConn()"]
    BS["broadcastState()\nPool → Snapshot → Marshal\n→ return Pool → send chan"]

    T --> D --> M --> RG --> FC
    FC --> FH
    FH -->|yes| FA --> FC
    FH -->|no, next| FC
    FC -->|done| PC
    PC --> PH
    PH -->|yes| PK --> PC
    PH -->|no, next| PC
    PC -->|done| KP --> BS
```

---

## JWT Authentication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant M as JWT Middleware
    participant P as jwt.ParseWithClaims
    participant L as fiber.Locals

    C->>M: Authorization: Bearer {token}
    Note over M: extractToken() checks header first,\nthen ?token= query param
    M->>P: ParseWithClaims(raw, &JWTClaims{}, keyFunc)
    P-->>M: token, err
    alt err != nil or !token.Valid
        M-->>C: 401 {"error": "..."}
    end
    M->>M: verify claims.Sub != "" && claims.Email != ""
    alt Missing required fields
        M-->>C: 401 {"error": "token payload is missing required fields"}
    end
    M->>L: c.Locals("user", claims)
    M-->>C: c.Next() — upgrade to WebSocket
```

---

## Spatial Grid

```mermaid
graph TB
    World["World 3000×3000 units"]
    Grid["SpatialGrid 15×15 cells\n(200 units per cell)"]
    FGrid2["FoodGrid 15×15 cells\n(200 units per cell)"]

    World --> Grid
    World --> FGrid2

    Query["QueryRadius(x, y, r)\nonly checks overlapping cells\nO(N) instead of O(N²)"]
    Grid --> Query
    FGrid2 --> Query

    subgraph Benefits["Performance"]
        Benefit1["Food collision\n200 food items × N players"]
        Benefit2["Player collision\nN players × N players"]
    end
    Query --> Benefits
```

---

## Graceful Shutdown

```mermaid
sequenceDiagram
    participant OS as OS Signal
    participant SG as Signal Goroutine
    participant Hub as game.Hub
    participant Rooms as All Active Rooms
    participant WG as sync.WaitGroup
    participant Fiber as Fiber Server

    OS->>SG: SIGINT / SIGTERM
    SG->>Hub: hub.Shutdown()
    Hub->>Rooms: broadcast stop signal
    Rooms->>WG: wg.Done() (each room on Run() exit)
    SG->>WG: wg.Wait()
    WG-->>SG: all rooms stopped
    SG->>Fiber: server.Shutdown()
```

---

## Configuration

| Variable | Required | Default | Description |
|---|---|---|---|
| `JWT_SECRET` | Yes | — | HMAC-SHA256 secret (shared with Auth service) |
| `PORT` | No | `8080` | HTTP port |
| `FRONTEND_URL` | No | `http://localhost:5173` | Allowed CORS origin |
| `MAX_PLAYERS_PER_ROOM` | No | `50` | Maximum players before a room is full |
| `WORLD_WIDTH` | No | `3000` | World width in game units |
| `WORLD_HEIGHT` | No | `3000` | World height in game units |
| `TICK_RATE_MS` | No | `33` | Milliseconds between ticks (~30 fps) |
| `FOOD_COUNT` | No | `200` | Number of food pellets in the world at any time |

---

## WebSocket Protocol

### Client → Server

```json
{ "type": "move", "dx": 0.5, "dy": -0.2 }
```

`dx`, `dy` in range `[-1, 1]`.

### Server → Client

| Type | When | Key fields |
|---|---|---|
| `init` | On connection | `playerId`, `color`, `worldWidth`, `worldHeight`, full `food` list |
| `state` | Every tick (~33 ms) | `tick`, `players[]`, `foodAdded[]`, `foodRemoved[]`, optional `leaderboard` (every 10 ticks) |
| `died` | Player absorbed | `killedBy`, `finalScore` |
| `error` | Auth or internal error | `message` |

---

## Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/health` | None | Server health: active rooms and player count |
| `WS` | `/ws/game` | JWT | Primary game WebSocket channel |
| `WS` | `/game/ws/game` | JWT | Alternate path (same handler) |

---

## Local Execution

```bash
go mod download
go run .

# With Docker
docker compose up --build
```

---

## Further Reading

- Source repository: [EciWise/game](https://github.com/EciWise/game)
- Spatial grid: `game/grid.go`
- Room loop: `game/room.go`
- JWT middleware: `ws/jwt.go`
