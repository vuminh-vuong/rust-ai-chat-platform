# AI Chat Platform — Full-Stack Production System

A production-grade, real-time AI chat platform built from scratch with **Rust** (backend) and **Svelte 5** (frontend). Designed for high throughput, low latency, and clean maintainability at scale.

> **Note:** Source code is private. This document outlines the architecture, technical decisions, and engineering patterns implemented in the project — intended as a portfolio reference.

---

## Table of Contents

- [System Overview](#system-overview)
- [Tech Stack](#tech-stack)
- [Backend Architecture](#backend-architecture)
- [Frontend Architecture](#frontend-architecture)
- [Key Engineering Highlights](#key-engineering-highlights)
- [API Design](#api-design)
- [Security Model](#security-model)
- [Real-Time Infrastructure](#real-time-infra)
- [Payment Integration](#payment-integration)
- [AI Integration Layer](#ai-integration-layer)
- [Deployment](#deployment)
- [Skills Demonstrated](#skills-demonstrated)

---

## System Overview

```
┌──────────────────────────────────────────────────────────┐
│                      CLIENT (Browser)                    │
│              Svelte 5 + TailwindCSS 4 + Vite 6           │
└────────────────────────┬─────────────────────────────────┘
                         │  REST + WebSocket
┌────────────────────────▼─────────────────────────────────┐
│                  BACKEND (Rust / Actix-Web 4)            │
│                                                          │
│  ┌────────────┐  ┌──────────┐  ┌─────────┐  ┌────────┐  │
│  │ Middleware │  │Endpoints │  │Services │  │  Auth  │  │
│  └────────────┘  └──────────┘  └─────────┘  └────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │              SeaORM  ·  PostgreSQL               │    │
│  └──────────────────────────────────────────────────┘    │
│  ┌──────────────┐   ┌──────────────┐                     │
│  │  Redis PubSub│   │  Moka Cache  │                     │
│  └──────────────┘   └──────────────┘                     │
└──────────────────────────────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   AI Provider Layer  │
              │  (Gemini / OpenAI)   │
              └─────────────────────┘
```

---

## Tech Stack

### Backend
| Category | Technology |
|---|---|
| Language | Rust (edition 2021) |
| Web Framework | Actix-Web 4 |
| Async Runtime | Tokio (full features) |
| ORM | SeaORM 2.0 (rc) + SQLx 0.8 |
| Database | PostgreSQL (via rustls, no OpenSSL runtime dep) |
| Migrations | SeaORM Migration CLI |
| Caching | Moka (in-process, async-aware) |
| Message Broker | Redis Pub/Sub |
| WebSocket | Actix Actors (`actix-web-actors`) |
| Auth | JWT (jsonwebtoken 9) + Argon2 password hashing |
| OAuth2 | Google · Microsoft · Apple (server-side token verification) |
| Rate Limiting | `actix-governor` + custom `governor` strategies |
| Validation | `validator` crate (derive macros) |
| Serialization | `serde` / `serde_json` / `rmp-serde` (MessagePack) |
| HTTP Client | `reqwest` (rustls backend) |
| Email | `lettre` (TLS) |
| Tracing | `tracing` + `tracing-actix-web` + JSON subscriber |
| API Docs | `utoipa` + Swagger UI (feature-flagged) |
| Error Handling | `thiserror` (typed) + `anyhow` (context propagation) |
| Cross-Compile | `cross` (Windows host → Linux target) |
| Containerization | Docker multi-stage build |

### Frontend
| Category | Technology |
|---|---|
| Framework | Svelte 5 (Runes API) |
| Build Tool | Vite 6 |
| Styling | TailwindCSS 4 + PostCSS |
| Component Library | bits-ui + shadcn-svelte |
| Table | TanStack Table Core v8 |
| Realtime | Socket.IO Client v4 |
| Schema Validation | Zod v4 |
| Dates | @internationalized/date |

---

## Backend Architecture

### Cargo Workspace — Modular Crates

```
backend/
├── src/                  # Binary entry point (main.rs, config.rs, state.rs, swagger.rs)
├── crates/
│   ├── appcore/          # Shared foundation: auth, CRUD traits, DB pool, error types, validation
│   ├── dbcontext/        # SeaORM entity definitions & table relationships
│   ├── migration/        # Schema migrations (up/down)
│   ├── endpoint/         # HTTP handlers, one file per domain resource
│   ├── middleware/       # Auth guard, logging middleware, firewall
│   └── service/          # Business logic: session, payment, WebSocket, PubSub
└── macro/                # Custom proc-macro crate (code-gen utilities)
```

### FastEndpoints-Inspired Pattern

The backend mirrors the .NET **FastEndpoints** architecture. Each domain resource implements a typed `GetEndpoint` / `PostEndpoint` trait with a three-phase lifecycle:

```
handle_before()  →  handle()  →  handle_after()
```

This enforces a consistent contract across all endpoints: pre-processing, core logic, post-processing (side effects, cache invalidation, audit logging).

### Request Pipeline

```
HTTP Request
    │
    ▼
[1] FirewallMiddleware      — IP blocklist with progressive ban durations
    │
    ▼
[2] RateLimitMiddleware     — Token-bucket / sliding-window per route
    │
    ▼
[3] TracingLogger           — Structured JSON request logs (tracing-actix-web)
    │
    ▼
[4] AuthMiddleware          — JWT validation → inject Claims into request extensions
    │
    ▼
[5] GuardMiddleware         — Role / permission check per endpoint
    │
    ▼
[6] Endpoint Handler        — Business logic via service layer
    │
    ▼
HTTP Response
```

### Caching Strategy

- **In-process (Moka):** Hot-path data (user sessions, quota, config) — TTL-based, async-aware, thread-safe.
- **Redis Pub/Sub:** Cross-instance event propagation for real-time chat message fan-out.
- Cache keys are namespaced; cache invalidation is triggered explicitly inside `handle_after()`.

### Error Handling

A central `AppError` enum implements `ResponseError` from Actix-Web, mapping each variant to the correct HTTP status code automatically:

```
AppError::NotFound       → 404
AppError::Unauthorized   → 401
AppError::Forbidden      → 403
AppError::Validation(_)  → 422
AppError::Database(_)    → 500
```

This eliminates scattered `.unwrap()` calls and keeps handler code clean.

---

## Frontend Architecture

```
src/
├── components/         # Feature-components (chat, payment, session, sidebar…)
├── lib/
│   ├── services/       # API call abstraction per domain
│   ├── stores/         # Svelte stores (reactive state)
│   ├── styles/         # Global style utilities
│   └── components/     # Primitive / shared UI components
└── services/           # Top-level services (auth.js)
```

### Notable UI Modules

| Component | Description |
|---|---|
| `ChatContainer` | Full chat viewport, session-aware |
| `MessageList` + `ChatMessage` | Virtualized message rendering with bot/user discrimination |
| `TypingIndicator` | Real-time "AI is thinking…" state via WebSocket events |
| `VoiceRecording` | Browser MediaRecorder API integration for voice input |
| `PaymentGateway` / `PaymentPlans` / `PaymentCallback` | Full payment flow: plan selection → gateway → confirmation |
| `SessionList` | Multi-session sidebar (history navigation) |
| `TaskBoard` | Kanban-style task management panel |
| `PrivateNotes` | Per-user note-taking with local persistence |
| `QuotaDisplay` | Real-time API usage / credit tracking |
| `Broadcast` | Admin broadcast message system |
| `LoginPopup` | OAuth2 login (Google / Microsoft / Apple) |

---

## Key Engineering Highlights

### 1. Dual-Token Authentication

- **Access Token** — short-lived (48h), contains user claims, validated on every request via middleware (no DB hit when token is fresh in cache).
- **Refresh Token** — long-lived (30 days), used only to issue new access tokens, stored in DB for revocation support.
- Token rotation on refresh: old refresh token is invalidated, new pair issued.

### 2. OAuth2 — Server-Side Token Verification

The backend never blindly trusts tokens from the frontend. For each OAuth2 login (Google / Microsoft / Apple), the backend makes a direct server-to-provider verification call before creating a session. This prevents token spoofing at the client layer.

### 3. Progressive IP Firewall

Automated threat response with escalating ban durations:

```
1st violation  →  5 minutes
2nd violation  →  30 minutes
3rd violation  →  2 hours
4th+ violation →  24 hours
```

Implemented in `FirewallMiddleware` using a `DashMap` (concurrent hash map, lock-free reads).

### 4. Rate Limiting — Multiple Strategies

`actix-governor` + custom governor keyers support:
- **Fixed Window** — hard cap per time period
- **Sliding Window** — smoother, avoids boundary bursts
- **Token Bucket** — allows bursts within overall rate
- **Per-IP Concurrency** — max simultaneous requests per client

Different strategies are applied per route group (e.g., auth endpoints are stricter than read endpoints).

### 5. AI API Swapper

A transparent proxy layer that accepts **OpenAI-compatible API requests** and routes them to **Google Gemini**. This means any OpenAI SDK can target this backend with zero client-side changes. Features:

- Model name mapping (`gpt-4o` → `gemini-2.0-flash-exp`, etc.)
- In-memory response caching (TTL 5min)
- Per-key rate limiting (60 req/min)
- Retry with exponential backoff (3 retries, 1s initial delay)
- Streaming (SSE) with heartbeat keep-alive (15s interval, 8KB chunks)
- Usage/token tracking

### 6. Real-Time Chat via Redis Pub/Sub + WebSocket Actors

```
User sends message
    │
    ▼
Backend HTTP endpoint saves message, publishes to Redis channel "chat:asks"
    │
    ▼
PubSubService listens on "chat:asks", receives AI response
    │
    ▼
WsBroadcaster fans out to relevant WebSocket sessions via Actix Actor messages
    │
    ▼
Frontend receives MessageEvent via Socket.IO, renders immediately
```

`WsSession` is an Actix Actor; connections are managed via `WsBroadcaster` (a shared `Arc<RwLock<HashMap>>` of session → actor address). Geolocation is injected from Cloudflare `CF-IPCountry` headers for locale-aware AI responses.

### 7. Entitlement / Quota System

Each user has associated credits/quotas tracked in DB and cached in Moka. The `entitlement` endpoint module enforces limits before forwarding messages to the AI layer—preventing cost overruns without round-trips to a third-party metering service.

### 8. Custom Proc-Macro Crate

A `macro/` crate provides derive macros that eliminate boilerplate for entity-to-DTO conversion and endpoint trait implementations. This follows the same philosophy as `serde` — compiler-time code generation, zero runtime overhead.

---

## API Design

- **RESTful**, versioned under `/r1/`
- Auto-generated **Swagger UI** (feature-flagged via `--features swagger`, disabled in production builds)
- Consistent response envelope:
  ```json
  { "status": "success" | "error", "data": {...} | null, "message": "..." }
  ```
- Pagination via `?page=&page_size=` on all list endpoints
- Soft-delete pattern (`is_deleted` flag) on mutable entities

Domain endpoints: `users`, `usersessions`, `conversations`, `messages`, `sessions`, `pricing_plans`, `payment`, `payment_methods`, `entitlements`, `health`, `websocket`

---

## Security Model

| Layer | Mechanism |
|---|---|
| Password storage | Argon2id (memory-hard, side-channel-resistant) |
| Token transport | HTTP-only cookies (session) + Authorization header (API) |
| Token validation | RS256 JWT with embedded claims, middleware-level check |
| SQL injection | SeaORM typed query builder — no raw SQL in business logic |
| CORS | Configured per environment (`actix-cors`) |
| IP abuse | Progressive firewall (`DashMap`-backed, sub-millisecond check) |
| Secrets | Environment variables override config file, placeholders auto-filtered |
| TLS | `rustls` throughout — no OpenSSL runtime dependency |

---

## Real-Time Infra

```
Browser
  │  Socket.IO (WebSocket)
  ▼
actix-web-actors (WsSession Actor)
  │  Actix message passing
  ▼
WsBroadcaster  (Arc<RwLock<HashMap<Uuid, Addr<WsSession>>>>)
  ▲
  │  Redis Pub/Sub listener (PubSubService)
  │
Redis ← published by AI response pipeline
```

Heartbeat messages keep long-idle connections alive across load balancers and proxies.

---

## Payment Integration

Three payment gateways fully integrated:

| Gateway | Region | Method |
|---|---|---|
| **Momo** | Vietnam | E-wallet, QR code |
| **PayPal** | International | Credit card / PayPal balance |

Config is layered (env var > TOML file > default), with automatic placeholder filtering — secrets never land in version control. Gateway selection is runtime-configurable without code changes.

---

## AI Integration Layer

- **Model routing:** Configurable model map (OpenAI names → Gemini model IDs)
- **Context injection:** Each WebSocket `MessageEvent` carries `previous_question` + `previous_answer` — the AI layer receives sliding conversation context without the client managing history
- **Streaming:** Server-Sent Events (SSE) with chunked transfer, heartbeat every 15s
- **Retry logic:** 3 attempts with 1s base delay, exponential backoff on 429/5xx
- **Caching:** Identical prompts cached for 5 minutes to reduce API costs

---

## Deployment

```
Windows Dev Machine
    │
    │  cross (cargo cross-compile)
    ▼
Linux x86_64 binary  ──►  Docker image (multi-stage, debian:bookworm-slim)
                                │
                                ▼
                          VPS / Cloud VM
                          docker-compose up
```

- **Multi-stage Docker build:** builder stage (Rust toolchain) → runtime stage (minimal Debian image). Final image contains only the binary + config.
- **Cross-compilation** from Windows to Linux using `cross` with OpenSSL vendored (static link, no system OpenSSL required at runtime).
- **ngrok** wrapper scripts included for local-to-public tunneling during development / webhook testing (Momo, PayPal callbacks).

---

## Skills Demonstrated

### Systems Programming
- Memory-safe concurrent code using Rust ownership model
- Actor-based concurrency with Actix
- Zero-cost async I/O with Tokio
- Lock-free data structures (`DashMap`, `Arc<RwLock<T>>`)

### Backend Engineering
- Modular Cargo workspace design (separation of concerns across crates)
- Trait-based polymorphism for extensible endpoint patterns
- Typed ORM usage (SeaORM) with migration management
- Layered middleware pipeline with clean request/response contracts
- Pub/Sub architecture for real-time decoupling

### Security
- JWT issuance, validation, and rotation
- OAuth2 server-side token verification
- Argon2 password hashing
- IP-based threat mitigation with progressive penalties
- Secrets management via environment variable layering

### Frontend
- Reactive UI with Svelte 5 Runes (latest paradigm)
- Real-time WebSocket integration
- Payment UI flows (redirect, callback, status polling)
- Voice input (MediaRecorder API)
- Component architecture with shadcn-svelte + bits-ui

### DevOps / Infra
- Multi-stage Docker builds
- Cross-compilation (Windows → Linux)
- docker-compose orchestration
- ngrok automation scripts for webhook development

### API & Integration
- OpenAI-compatible proxy with model swapping
- Google Gemini streaming API
- Multi-gateway payment (Momo, PayPal)
- OAuth2 providers (Google, Microsoft, Apple)

---

## Contact

Open to **freelance projects** involving:
- Rust backend systems (APIs, real-time services, data pipelines)
- High-performance web backends (Actix-Web, Axum)
- Modern frontend development (Svelte, React)
- System architecture design and review
- Auth/security implementation (OAuth2, JWT, SSO)
- Payment gateway integration (international + Vietnamese market)

> Feel free to reach out if you're looking for someone who can design and build production-quality systems end-to-end.
