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

## System Architecture

```mermaid
graph TD
    Browser["Browser / Game Client"]
    Fiber["Fiber v2\nHTTP Server"]
    RateLimit["Rate Limiter\n20 conn / IP / min"]
    JWTMw["JWT Middleware\nHMAC-SHA256"]
    Handler["ws.GameHandler"]
    Hub["game.Hub\nsync.RWMutex · WaitGroup"]
    Room1["game.Room 1\nRun() goroutine · 30 fps"]
    Room2["game.Room 2\nRun() goroutine · 30 fps"]
    RoomN["game.Room N\nRun() goroutine · 30 fps"]
    SGrid["SpatialGrid\n15×15 cells · O(N)"]
    FGrid["FoodGrid\n15×15 cells · O(N)"]

    Browser -->|"WSS /ws/game\n?token=JWT"| Fiber
    Fiber --> RateLimit --> JWTMw --> Handler
    Handler -->|"Register(client)"| Hub
    Hub -->|"join chan"| Room1
    Hub -->|"join chan"| Room2
    Hub -->|"join chan"| RoomN
    Room1 --- SGrid
    Room1 --- FGrid
```

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
