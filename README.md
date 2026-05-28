

What is this?
CollabCode lets multiple developers edit the same file simultaneously — changes sync in milliseconds, cursors appear live, and you can run the code right in the browser. No installation, no setup, just share a room link.
Built to explore the hard parts of distributed systems: conflict resolution (CRDT), real-time messaging (WebSocket + Redis pub/sub), and secure code execution (Docker sandboxing).

Demo
Open two browser tabs → same room URL → type in one → watch it appear in the other.
Supported languages: Python · JavaScript · Go · C++ · Java

Architecture
┌─────────────────────────────────────────────────────────┐
│                      Client (React)                     │
│   Monaco Editor  ·  WebSocket client  ·  Cursor layer   │
└────────────────────────┬────────────────────────────────┘
                         │ WebSocket
┌────────────────────────▼────────────────────────────────┐
│                  Go WebSocket Server                    │
│          Room manager  ·  OT/CRDT engine                │
└──────┬─────────────────────────────────┬────────────────┘
       │ Pub/Sub                         │ Execute
┌──────▼──────┐                 ┌────────▼───────┐
│    Redis    │                 │ Docker Sandbox  │
│  (rooms,    │                 │ (Python, JS,    │
│   sessions) │                 │  Go, Java, C++) │
└─────────────┘                 └────────────────┘
       │
┌──────▼──────┐
│ PostgreSQL  │
│  (auth,     │
│   history)  │
└─────────────┘
Key design decisions
DecisionChoiceWhySync algorithmYjs CRDTPeer-to-peer friendly, no server-side transform computation, handles offline editsMessage brokerRedis pub/subStateless server instances can broadcast to all peers in a room without shared memoryCode sandboxDocker + gVisorProcess isolation, resource caps (CPU 0.5, RAM 128MB, 10s timeout), no host accessAuthJWT (RS256)Stateless verification, short-lived access tokens + Redis-stored refresh tokensDBPostgreSQLRoom history, user accounts; Redis is ephemeral session/presence state only

Features

Live collaboration — multiple cursors, named presence indicators, sub-100ms sync latency on LAN
Conflict-free editing — Yjs CRDT merges concurrent edits without last-write-wins data loss
In-browser execution — run code in a Docker-sandboxed container, stdout/stderr stream back via WebSocket
Room system — create public or private rooms, share via link, no account required for guests
Syntax highlighting — Monaco Editor (the VS Code engine) with full language support
Session history — replay edits, restore previous snapshots (authenticated users)
Rate limiting — per-user execution rate limit (10 runs/min) to prevent abuse


Tech Stack
Frontend

React 18 + TypeScript
Monaco Editor — VS Code's editor engine, full LSP-like experience in the browser
Yjs — CRDT library handling real-time sync and offline merge
Zustand — lightweight global state for room/session data
WebSocket API — native browser WebSocket, reconnects automatically

Backend

Go 1.22 — concurrent WebSocket handling with goroutines
gorilla/websocket — production WebSocket library for Go
Redis 7 — pub/sub for cross-instance broadcasting, presence TTLs
PostgreSQL 16 — persistent storage for users, rooms, code snapshots
Docker SDK — spawns isolated containers per execution request
JWT (RS256) — asymmetric auth tokens

Infrastructure

Docker Compose — local development (server + Redis + Postgres + sandbox runner)
AWS ECS / GCP Cloud Run — production deployment targets
Nginx — reverse proxy, WebSocket upgrade headers


Getting Started
Prerequisites

Go 1.22+
Node.js 20+
Docker + Docker Compose
Redis 7 (or use the compose file)

Clone and run locally
bashgit clone https://github.com/yourusername/collabcode.git
cd collabcode

# Start Redis + PostgreSQL via Docker
docker compose up -d redis postgres

# Run the backend
cd server
cp .env.example .env     # fill in DB_URL, REDIS_URL, JWT_SECRET
go mod tidy
go run main.go

# Run the frontend (new terminal)
cd client
npm install
npm run dev
Open http://localhost:5173 — create a room, share the URL, open it in another tab.
Environment variables
env# server/.env
PORT=8080
DATABASE_URL=postgres://postgres:password@localhost:5432/collabcode
REDIS_URL=redis://localhost:6379
JWT_SECRET_PATH=./keys/private.pem
JWT_PUBLIC_PATH=./keys/public.pem
DOCKER_SANDBOX_IMAGE=collabcode-sandbox:latest
MAX_EXECUTION_TIME_SECONDS=10
MAX_EXECUTION_MEMORY_MB=128
Build the sandbox image
bashcd sandbox
docker build -t collabcode-sandbox:latest .

Project Structure
collabcode/
├── client/                  # React frontend
│   ├── src/
│   │   ├── components/
│   │   │   ├── Editor/      # Monaco integration + cursor overlay
│   │   │   ├── Room/        # Room UI, presence list
│   │   │   └── Output/      # Execution output panel
│   │   ├── store/           # Zustand state slices
│   │   ├── hooks/           # useWebSocket, useCollaboration
│   │   └── lib/
│   │       └── crdt.ts      # Yjs document + provider setup
│   └── package.json
│
├── server/                  # Go backend
│   ├── main.go
│   ├── ws/                  # WebSocket hub, room manager
│   ├── crdt/                # Server-side Yjs state sync (y-websocket protocol)
│   ├── exec/                # Docker sandbox execution engine
│   ├── auth/                # JWT middleware
│   ├── db/                  # PostgreSQL queries (sqlc generated)
│   └── config/
│
├── sandbox/                 # Docker sandbox image
│   ├── Dockerfile
│   └── runner.sh            # Entrypoint with resource limits
│
├── docs/
│   ├── architecture.md      # ADR and design decisions
│   └── crdt-vs-ot.md        # Why CRDT over Operational Transforms
│
└── docker-compose.yml

The Hard Part: CRDT Sync
The sync layer uses Yjs, a high-performance CRDT implementation. Every client holds a local Y.Doc. Edits produce binary update messages broadcast via WebSocket. The server acts as a relay and persists doc state in Redis.
Client A types "hello"
  → Yjs encodes delta as Uint8Array
  → WebSocket sends to server
  → Server publishes to Redis channel "room:{id}"
  → All subscribers (other server instances) receive it
  → Forward to connected clients in that room
  → Client B's Y.Doc applies update, Monaco reflects change
Conflicts are impossible by construction — CRDT merges are commutative, associative, and idempotent. Order of arrival doesn't matter; the result is always the same.
See docs/crdt-vs-ot.md for the detailed tradeoff analysis (this doc is worth reading before any interview).

Code Execution Sandbox
Each run request:

Server picks an execution container from a warm pool
Code is written to a tmpfs volume (never touches host disk)
Container starts with hard limits: 0.5 CPU, 128MB RAM, 10s wall-clock timeout
stdout/stderr stream back through a Unix socket → WebSocket to client
Container is destroyed after execution completes

go// exec/runner.go (simplified)
func (r *Runner) Execute(ctx context.Context, req ExecRequest) (<-chan string, error) {
    containerID, err := r.pool.Acquire(ctx)
    if err != nil {
        return nil, err
    }
    defer r.pool.Release(containerID)

    out := make(chan string, 64)
    go r.stream(ctx, containerID, req.Code, req.Language, out)
    return out, nil
}

Scalability Notes
The server is stateless with respect to rooms — all room state lives in Redis. This means:

Horizontal scaling: spin up N server instances behind a load balancer; any instance can handle any WebSocket from any room
Sticky sessions not required: Yjs updates route through Redis pub/sub, not in-process memory
Bottleneck: Redis pub/sub at very high room counts → solution is Redis Cluster or per-shard room assignment

For 10,000 concurrent rooms at ~5 users each: estimated 2 Go server instances (4 vCPU each) + 1 Redis node handles it comfortably.

Roadmap

 Voice chat (WebRTC)
 Git integration — commit directly from the editor
 LSP support (go-to-definition, autocomplete) via language server proxy
 Mobile-responsive layout
 End-to-end encrypted private rooms

