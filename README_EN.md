<p align="center">
  <h1 align="center">NexusLink</h1>
  <p align="center">
    <strong>Hybrid Messaging Platform -- Where Centralized Meets Decentralized</strong>
  </p>
  <p align="center">
    <a href="./README.md">中文文档</a> |
    <a href="./docs/en/architecture.md">Architecture</a> |
    <a href="./ROADMAP.md">Roadmap</a> |
    <a href="./CONTRIBUTING.md">Contributing</a>
  </p>
</p>

---

## What is NexusLink?

NexusLink is an open-source, cross-platform instant messaging application that uniquely **bridges centralized and decentralized communication** within a single app. Users choose how they connect -- through an official directory server, a self-hosted community server, or direct peer-to-peer -- without sacrificing usability or security.

**No phone number. No email. No registration.** Your identity is a cryptographic key pair generated on your device's secure hardware. You own your identity.

### The Problem

| Centralized Apps (WeChat, Telegram, Discord) | Decentralized Apps (Signal, Briar, Session) |
|---|---|
| Easy to use, feature-rich | Privacy-first, censorship-resistant |
| You surrender data sovereignty | Limited social/community features |
| Single point of failure and surveillance | Fragmented user experience |
| No choice in who hosts your data | Technical barrier for average users |

**NexusLink fills the gap** -- offering the community features of centralized platforms with the privacy and sovereignty of decentralized ones, letting each user decide their own trust level.

---

## Key Features

### Identity Without Registration
- Cryptographic key pair generated in device secure hardware (TPM / Secure Enclave / StrongBox)
- Your public key **is** your UUID -- no server involved in identity creation
- Multi-device sync via cross-signing protocol
- Identity migration via mnemonic recovery phrase (12-24 words)

### Three Connection Modes

```
+-------------------------------------------------------------+
|                    NexusLink Client                          |
|                                                              |
|  [A] Official Server     [B] Community Server    [C] P2P    |
|  - Real-name verified    - Self-hosted            - Direct   |
|  - Browse communities    - Data sovereignty       - No server|
|  - Regulated channels    - Custom rules           - E2E only |
+-------------------------------------------------------------+
```

**Mode A: Official Directory Server**
- Real-name (KYC) verified UUID required
- Browse and join registered communities
- Access official public channels
- Anti-abuse protections

**Mode B: Community Self-Hosted Server**
- Anyone can deploy with the same server codebase
- Optionally register in the official directory for discoverability
- Full data sovereignty -- the official server never sees your messages, members, or metadata
- Community-defined moderation rules

**Mode C: Peer-to-Peer**
- Direct device-to-device communication via libp2p + WebRTC
- Exchange public keys via QR code or out-of-band
- Zero server dependency (STUN for NAT traversal only)
- Maximum privacy for sensitive conversations

### Privacy by Architecture
- End-to-end encryption (Double Ratchet protocol) for all messages
- Official directory server is a **phonebook only** -- stores community listings, not messages or membership
- Community servers are **privacy shields** -- metadata never flows to the official server
- Zero-knowledge proof ready for future KYC (prove identity without revealing details)

### Cross-Platform
- iOS, Android, Windows, macOS, Linux
- Flutter UI + Rust core via FFI
- Consistent experience across all platforms

---

## Architecture Overview

```
+--------------------------------------------------------------+
|                         Client                                |
|  +--------------------+  +----------------------------------+ |
|  |   Flutter UI Layer |  |        Rust Core Library         | |
|  |                    |<-|  - Identity (Secure HW)          | |
|  |  - Chat screens    |  |  - E2E Encryption                | |
|  |  - Community view  |  |  - P2P Network (libp2p)          | |
|  |  - Settings        |  |  - Protocol codec                | |
|  |  - Multi-language  |  |  - Local encrypted storage       | |
|  +--------------------+  +----------------------------------+ |
+------------------+-------------------+-----------------------+
                   |                   |
         +---------+                   +----------+
         v                                        v
+---------------------+               +-----------------------+
|  Directory Server   |               |  Community Server     |
|  (Official)         |               |  (Self-hosted)        |
|                     |               |                       |
|  - Community index  |   register    |  - Chat hosting       |
|  - KYC service      |<-------------|  - Member management  |
|  - Abuse reporting  |   (metadata   |  - File storage       |
|                     |    only)      |  - Custom moderation  |
+---------------------+               +-----------------------+
                                               |
                                      P2P direct connection
                                               |
                                      +--------v----------+
                                      |   Another Client   |
                                      +-------------------+
```

For detailed architecture documentation, see [ARCHITECTURE.md](./docs/en/architecture.md).

---

## Server Development

This section provides a comprehensive guide for developers who want to understand, extend, or deploy the NexusLink server components.

### Server Architecture Overview

NexusLink uses a **shared codebase** model for its server infrastructure. Both the **Community Server** and the **Directory Server** are built from the same Rust workspace under `server/`, sharing common libraries for configuration, error handling, type definitions, and utilities via the `server/shared/` crate. This design ensures consistency across server types and reduces code duplication.

```
server/
+-- shared/            Shared crate (config, error types, utilities)
+-- community/         Community Server binary and modules
+-- directory/         Directory Server binary and modules
```

The two servers serve fundamentally different roles and are deployed independently, but any infrastructure code (database abstractions, middleware, serialization helpers) is written once in `shared/` and consumed by both.

### Community Server

The Community Server is the primary workhorse of the NexusLink ecosystem. It is designed to be **self-hosted by anyone** -- an individual, an organization, or a community operator. Each Community Server instance operates autonomously, storing its own data and enforcing its own policies.

#### Responsibilities

- Host real-time messaging for all channels and groups within the community
- Manage member registration, roles, permissions, and moderation actions
- Store and serve uploaded files (images, documents, media)
- Maintain prekey bundles for offline message delivery (X3DH protocol)
- Deliver push notifications to offline clients
- Optionally register with the official Directory Server for discoverability

#### Core Modules

| Module | Description |
|--------|-------------|
| **WebSocket Real-Time Messaging** | Persistent WebSocket connections for bidirectional message delivery. Supports connection multiplexing, automatic reconnection signaling, and per-channel message routing. |
| **Message Queue** | Internal message queue for reliable delivery. Handles offline message buffering, delivery acknowledgments, and retry logic. Messages are queued per-recipient and flushed upon reconnection. |
| **Prekey Management** | Stores X3DH prekey bundles uploaded by clients. Serves one-time prekeys to initiators of new conversations and manages prekey rotation and replenishment signals. |
| **Channel and Group Management** | Full lifecycle management for channels (public, private, announcement) and groups. Includes creation, archival, topic management, pinned messages, and permission inheritance. |
| **File Storage** | Handles file upload, storage, and retrieval. Supports configurable storage backends (local filesystem, S3-compatible object storage). Files are referenced by content hash for deduplication. |
| **Push Notifications** | Integrates with platform push services (APNs, FCM) to notify offline clients of new messages. Push payloads contain only opaque notification identifiers -- no message content. |
| **Member Management and Moderation** | Role-based access control (owner, admin, moderator, member, guest). Supports bans, mutes, invite-only channels, and customizable moderation policies per community. |
| **Rate Limiting** | Per-client and per-endpoint rate limiting using the token bucket algorithm. Configurable thresholds to prevent abuse without impacting normal usage. Implemented via tower middleware. |
| **Health Checks** | Liveness and readiness probe endpoints for container orchestration platforms. Reports database connectivity, WebSocket listener status, and background task health. |

#### API Overview

The Community Server exposes two transport interfaces:

**REST API (HTTPS)**

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/auth/register` | Register a new member (public key + signed challenge) |
| POST | `/api/v1/auth/challenge` | Request an authentication challenge |
| GET | `/api/v1/channels` | List available channels |
| POST | `/api/v1/channels` | Create a new channel (admin+) |
| GET | `/api/v1/channels/{id}/messages` | Fetch message history (paginated) |
| POST | `/api/v1/messages` | Send a message (REST fallback) |
| POST | `/api/v1/files/upload` | Upload a file |
| GET | `/api/v1/files/{hash}` | Download a file by content hash |
| GET | `/api/v1/members` | List community members |
| PUT | `/api/v1/members/{id}/role` | Update a member's role (admin+) |
| POST | `/api/v1/prekeys` | Upload prekey bundle |
| GET | `/api/v1/prekeys/{user_id}` | Fetch a prekey bundle for a user |
| GET | `/health/live` | Liveness probe |
| GET | `/health/ready` | Readiness probe |

**WebSocket API (WSS)**

| Event | Direction | Description |
|-------|-----------|-------------|
| `message.new` | Server -> Client | New message in a subscribed channel |
| `message.send` | Client -> Server | Send a message to a channel |
| `message.ack` | Server -> Client | Delivery acknowledgment |
| `typing.start` | Bidirectional | Typing indicator start |
| `typing.stop` | Bidirectional | Typing indicator stop |
| `presence.update` | Server -> Client | Member online/offline status change |
| `channel.update` | Server -> Client | Channel metadata changed |

#### Database Design

The Community Server uses PostgreSQL as its primary datastore. Key tables include:

```
members
  - id (UUID, primary key)
  - public_key (BYTEA, unique)
  - display_name (TEXT)
  - role (ENUM: owner, admin, moderator, member, guest)
  - joined_at (TIMESTAMPTZ)
  - banned (BOOLEAN)

channels
  - id (UUID, primary key)
  - name (TEXT)
  - channel_type (ENUM: public, private, announcement)
  - created_by (UUID, FK -> members)
  - created_at (TIMESTAMPTZ)
  - archived (BOOLEAN)

messages
  - id (UUID, primary key)
  - channel_id (UUID, FK -> channels)
  - sender_id (UUID, FK -> members)
  - ciphertext (BYTEA)        -- E2E encrypted payload
  - timestamp (TIMESTAMPTZ)
  - message_type (ENUM: text, file, system)

prekeys
  - id (BIGSERIAL, primary key)
  - user_id (UUID, FK -> members)
  - prekey_id (INTEGER)
  - public_key (BYTEA)
  - is_one_time (BOOLEAN)
  - uploaded_at (TIMESTAMPTZ)

files
  - content_hash (TEXT, primary key)
  - uploader_id (UUID, FK -> members)
  - file_size (BIGINT)
  - mime_type (TEXT)
  - storage_path (TEXT)
  - uploaded_at (TIMESTAMPTZ)
```

All message content stored in the `messages` table is the E2E-encrypted ciphertext. The Community Server never possesses the plaintext or the decryption keys.

### Directory Server

The Directory Server is the **official, centrally operated** component of the NexusLink network. Its scope is deliberately minimal: it functions as a **phonebook** for discovering communities, not as a message relay or storage backend.

#### Responsibilities

- Maintain a searchable index of registered communities (name, description, endpoint URL, public key)
- Provide KYC (Know Your Customer) identity verification for users who wish to participate in the official directory ecosystem
- Accept and process community registration applications
- Handle abuse reports and enforce network-wide policies
- Serve as a trust anchor for community server identity verification

#### API Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/communities` | Search and browse registered communities |
| GET | `/api/v1/communities/{id}` | Get community details |
| POST | `/api/v1/communities/register` | Register a new community (server operator) |
| PUT | `/api/v1/communities/{id}` | Update community listing (verified operator) |
| DELETE | `/api/v1/communities/{id}` | Delist a community |
| POST | `/api/v1/kyc/initiate` | Start KYC verification flow |
| POST | `/api/v1/kyc/verify` | Submit KYC verification documents |
| GET | `/api/v1/kyc/status` | Check KYC verification status |
| POST | `/api/v1/reports` | Submit an abuse report |
| GET | `/health/live` | Liveness probe |
| GET | `/health/ready` | Readiness probe |

#### KYC Flow

```
Client                      Directory Server              KYC Provider
  |                                |                           |
  |  POST /kyc/initiate           |                           |
  |------------------------------->|                           |
  |  { challenge, session_id }    |                           |
  |<-------------------------------|                           |
  |                                |                           |
  |  POST /kyc/verify             |                           |
  |  { signed_challenge, docs }   |                           |
  |------------------------------->|                           |
  |                                |  verify documents         |
  |                                |-------------------------->|
  |                                |  verification result      |
  |                                |<--------------------------|
  |                                |                           |
  |  { status: verified/rejected } |                           |
  |<-------------------------------|                           |
```

KYC verification binds a real-world identity to a UUID. The binding is stored as a one-way hash -- the Directory Server can confirm that a UUID has been verified without storing the original identity documents. This supports future zero-knowledge proof integration.

#### Community Registration Flow

```
Community Server              Directory Server
  |                                |
  |  POST /communities/register   |
  |  { name, desc, endpoint,      |
  |    server_public_key,          |
  |    operator_uuid (KYC'd) }    |
  |------------------------------->|
  |                                |  verify operator KYC status
  |                                |  verify server reachability
  |                                |  verify TLS certificate
  |                                |
  |  { community_id, status }     |
  |<-------------------------------|
```

Requirements for registration:
- The operator must have a KYC-verified UUID
- The community server must be reachable at the declared endpoint
- The community server must present a valid TLS certificate
- The server's public key must match the one submitted during registration

#### Anti-Abuse Mechanisms

- **Rate limiting**: All Directory Server endpoints enforce strict rate limits per UUID and per IP address
- **Reputation scoring**: Communities receive reputation scores based on report volume and verified activity
- **Automated delisting**: Communities with sustained high abuse-report ratios are automatically delisted pending review
- **Proof-of-work challenges**: Optional computational challenges for registration to deter automated spam
- **IP-based throttling**: Progressive backoff for repeated failed requests from the same origin

### Deployment

#### Docker One-Click Deploy

The recommended deployment method for production environments:

```bash
# Community Server
docker pull nexuslink/community-server:latest
docker run -d \
  --name nexuslink-community \
  -p 443:443 \
  -p 8443:8443 \
  -v /path/to/data:/data \
  -v /path/to/config.toml:/etc/nexuslink/config.toml \
  -e NEXUSLINK_DOMAIN=your.community.domain.com \
  -e DATABASE_URL=postgres://user:pass@db:5432/nexuslink \
  nexuslink/community-server:latest

# Directory Server (official deployment only)
docker pull nexuslink/directory-server:latest
docker run -d \
  --name nexuslink-directory \
  -p 443:443 \
  -v /path/to/data:/data \
  -v /path/to/config.toml:/etc/nexuslink/config.toml \
  -e DATABASE_URL=postgres://user:pass@db:5432/nexuslink_dir \
  nexuslink/directory-server:latest
```

A `docker-compose.yml` is provided for deploying a Community Server alongside PostgreSQL and Redis:

```bash
cd server/community
docker compose up -d
```

#### Build from Source

```bash
# Clone the repository
git clone https://github.com/yellowpeachxgp/NexusLink.git
cd NexusLink

# Build Community Server
cargo build --release -p nexuslink-community-server

# Build Directory Server
cargo build --release -p nexuslink-directory-server

# Run with a configuration file
./target/release/nexuslink-community-server --config config.toml
```

#### Configuration

Server configuration is managed via TOML files. Example `config.toml` for the Community Server:

```toml
[server]
bind_address = "0.0.0.0"
https_port = 443
ws_port = 8443
domain = "your.community.domain.com"

[database]
url = "postgres://user:password@localhost:5432/nexuslink"
max_connections = 50
min_connections = 5

[redis]
url = "redis://localhost:6379"    # Optional: enables caching and pub/sub
enabled = false

[storage]
backend = "local"                 # "local" or "s3"
local_path = "/data/files"
max_file_size = "50MB"

[tls]
cert_path = "/etc/nexuslink/cert.pem"
key_path = "/etc/nexuslink/key.pem"

[rate_limit]
requests_per_second = 100
burst_size = 200

[push]
apns_key_path = "/etc/nexuslink/apns.p8"
fcm_credentials_path = "/etc/nexuslink/fcm.json"

[directory]
enabled = false                   # Set to true to register with the Directory Server
directory_url = "https://directory.nexuslink.org"
server_public_key_path = "/etc/nexuslink/server_key.pub"
```

Environment variables override TOML values. All configuration keys map to environment variables with the `NEXUSLINK_` prefix (e.g., `NEXUSLINK_DATABASE_URL`).

### Server Tech Stack

| Component | Technology | Role |
|-----------|-----------|------|
| **HTTP Framework** | Axum | Async HTTP/WebSocket server built on hyper and tokio |
| **Async Runtime** | tokio | Multi-threaded async executor for all I/O operations |
| **Database Driver** | SQLx | Compile-time checked SQL queries with async PostgreSQL support |
| **Cache** | Redis (optional) | Session caching, rate limit counters, pub/sub for multi-instance WebSocket fan-out |
| **Middleware** | tower | Composable middleware stack: rate limiting, logging, authentication, CORS |
| **WebSocket** | tokio-tungstenite | High-performance async WebSocket implementation |
| **Serialization** | serde + serde_json | JSON serialization for REST API; Protocol Buffers for wire format |
| **TLS** | rustls | Pure-Rust TLS implementation for secure connections |
| **Logging** | tracing + tracing-subscriber | Structured, async-aware logging with span context |
| **Metrics** | prometheus-client | Prometheus-compatible metrics exposition |
| **Configuration** | config-rs | Layered configuration from files, environment variables, and defaults |
| **Migration** | SQLx migrations | Version-controlled database schema migrations |

### Server Monitoring and Operations

#### Health Check Endpoints

Both servers expose standardized health check endpoints compatible with Kubernetes and other orchestration platforms:

- `GET /health/live` -- Returns `200 OK` if the process is running. Used for liveness probes.
- `GET /health/ready` -- Returns `200 OK` only when the server is fully operational (database connected, background tasks running, WebSocket listener active). Used for readiness probes.

Response body example:

```json
{
  "status": "healthy",
  "version": "0.1.0",
  "uptime_seconds": 86400,
  "checks": {
    "database": "connected",
    "websocket_listener": "active",
    "background_tasks": "running",
    "redis": "connected"
  }
}
```

#### Prometheus Metrics

The server exposes a `/metrics` endpoint in Prometheus exposition format. Key metrics include:

| Metric | Type | Description |
|--------|------|-------------|
| `nexuslink_http_requests_total` | Counter | Total HTTP requests by method, path, and status code |
| `nexuslink_http_request_duration_seconds` | Histogram | Request latency distribution |
| `nexuslink_ws_connections_active` | Gauge | Currently active WebSocket connections |
| `nexuslink_ws_messages_total` | Counter | Total WebSocket messages sent and received |
| `nexuslink_db_query_duration_seconds` | Histogram | Database query latency |
| `nexuslink_db_pool_connections_active` | Gauge | Active database pool connections |
| `nexuslink_messages_queued` | Gauge | Messages waiting in the offline delivery queue |
| `nexuslink_file_uploads_total` | Counter | Total file uploads |
| `nexuslink_file_storage_bytes` | Gauge | Total file storage usage in bytes |

#### Logging

Structured JSON logging is used in production. Log levels are configurable per module at runtime via the `RUST_LOG` environment variable:

```bash
# Example: info-level for most modules, debug for the chat module
RUST_LOG="info,nexuslink_community::chat=debug" ./nexuslink-community-server
```

All log entries include request IDs, user IDs (when authenticated), and span context for distributed tracing compatibility.

#### Backup Strategy

- **Database**: Use `pg_dump` for logical backups or configure PostgreSQL continuous archiving (WAL) for point-in-time recovery. Recommended schedule: full backup daily, WAL archiving continuous.
- **File storage**: If using local storage, back up the configured `local_path` directory. If using S3-compatible storage, configure bucket versioning and cross-region replication.
- **Configuration**: Store `config.toml` and TLS certificates in version control or a secrets manager. Never store secrets in plaintext files in production.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **UI** | Flutter (Dart) | Cross-platform native UI |
| **Core** | Rust | Cryptography, networking, protocol logic |
| **Bridge** | flutter_rust_bridge / FFI | Connect Flutter to Rust |
| **Identity** | nmshd/rust-crypto | Secure hardware key management |
| **Encryption** | Double Ratchet + X3DH | E2E message encryption |
| **P2P** | libp2p + WebRTC | Direct peer connections |
| **Server Framework** | Axum | Async HTTP and WebSocket server |
| **Async Runtime** | tokio | Multi-threaded async I/O executor |
| **Database Driver** | SQLx | Compile-time verified async SQL |
| **Server Cache** | Redis (optional) | Caching, rate limiting, pub/sub fan-out |
| **Middleware** | tower | Composable server middleware stack |
| **Server TLS** | rustls | Pure-Rust TLS termination |
| **Server Database** | PostgreSQL | Server-side persistent storage |
| **Client Database** | SQLCipher | Encrypted local client storage |
| **Serialization** | Protocol Buffers / FlatBuffers | Wire format |
| **Metrics** | prometheus-client | Server observability |
| **Logging** | tracing | Structured async-aware logging |
| **CI/CD** | GitHub Actions | Automated testing and releases |

---

## Project Structure

```
NexusLink/
|-- client/                          # Client applications
|   |-- flutter/                     # Flutter UI (cross-platform)
|   |   |-- lib/
|   |   |   |-- screens/            # Page screens
|   |   |   |-- widgets/            # Reusable UI components
|   |   |   |-- services/           # Business logic services
|   |   |   |-- models/             # Data models
|   |   |   |-- providers/          # State management
|   |   |   |-- utils/              # UI utilities
|   |   |   +-- l10n/               # Internationalization (i18n)
|   |   |-- assets/                  # Icons, images, fonts
|   |   +-- test/                    # Widget and integration tests
|   |
|   +-- core/                        # Rust core library (shared logic)
|       |-- src/
|       |   |-- identity/            # Key generation, cross-signing, recovery
|       |   |-- crypto/              # E2E encryption, Double Ratchet, key exchange
|       |   |-- network/             # libp2p, WebRTC, NAT traversal
|       |   |-- protocol/            # Message encoding/decoding, wire format
|       |   |-- storage/             # Local encrypted database (SQLCipher)
|       |   +-- ffi/                 # FFI bindings for Flutter
|       |-- tests/                   # Integration tests
|       +-- benches/                 # Performance benchmarks
|
|-- server/                          # Server applications (shared Rust workspace)
|   |-- directory/                   # Official Directory Server
|   |   |-- src/
|   |   |   |-- api/                 # REST endpoints (community listing, KYC, reports)
|   |   |   |-- auth/                # KYC verification, UUID authentication, challenge-response
|   |   |   |-- db/                  # Database queries and connection pool management
|   |   |   |-- models/              # Data models (community listing, KYC record, report)
|   |   |   +-- services/            # Business logic (registration review, abuse detection, reputation)
|   |   +-- migrations/              # SQLx database migrations (version-controlled schema)
|   |
|   |-- community/                   # Community Server (self-hostable)
|   |   |-- src/
|   |   |   |-- api/                 # REST and WebSocket endpoints (messages, channels, files, members)
|   |   |   |-- chat/                # Real-time messaging engine (WebSocket handler, message routing,
|   |   |   |                        #   offline queue, typing indicators, presence tracking)
|   |   |   |-- db/                  # Database queries, connection pool, transaction management
|   |   |   |-- models/              # Data models (member, channel, message, prekey, file metadata)
|   |   |   |-- services/            # Business logic (member management, moderation, push notifications,
|   |   |   |                        #   prekey management, file storage, rate limiting)
|   |   |   +-- federation/          # Directory Server registration and heartbeat communication
|   |   |-- migrations/              # SQLx database migrations (version-controlled schema)
|   |   +-- docker-compose.yml       # One-click deployment with PostgreSQL and Redis
|   |
|   +-- shared/                      # Shared server crate (used by both directory and community)
|       +-- src/
|           |-- config/              # Layered configuration management (TOML + env vars)
|           |-- error/               # Unified error types and HTTP error response mapping
|           |-- types/               # Shared type definitions (UUID, pagination, timestamps)
|           +-- utils/               # Common utilities (hashing, validation, middleware helpers)
|
|-- protocol/                        # Protocol specifications
|   |-- specs/                       # Human-readable protocol documentation
|   +-- schemas/                     # Protobuf / FlatBuffers schema definitions
|
|-- docs/                            # Documentation
|   |-- en/                          # English documentation
|   |-- cn/                          # Chinese documentation
|   |-- diagrams/                    # Architecture diagrams
|   +-- api/                         # API reference (auto-generated)
|
|-- tools/                           # Developer tools
|   |-- cli/                         # CLI management tool
|   |   +-- src/
|   +-- scripts/                     # Build, deploy, test scripts
|
|-- .github/                         # GitHub configuration
|   |-- workflows/                   # CI/CD pipelines
|   +-- ISSUE_TEMPLATE/              # Issue templates
|
|-- README.md                        # Chinese README (primary)
|-- README_EN.md                     # English README (this file)
|-- ROADMAP.md                       # Development roadmap
|-- ARCHITECTURE.md                  # Architecture deep-dive
|-- CONTRIBUTING.md                  # Contribution guide
|-- LICENSE                          # AGPLv3
+-- Cargo.toml                       # Rust workspace root
```

---

## Quick Start

> **Note:** NexusLink is in early development. The following instructions will be fully functional once Phase 1 is complete.

### Prerequisites

- Rust 1.75+ (with `cargo`)
- Flutter 3.19+ (with `dart`)
- PostgreSQL 15+ (for server)
- Protobuf compiler (`protoc`)
- Docker and Docker Compose (optional, for containerized deployment)

### Client Development

```bash
# Clone the repository
git clone https://github.com/yellowpeachxgp/NexusLink.git
cd NexusLink

# Build the Rust core library
cargo build --release -p nexuslink-core

# Generate FFI bindings
cargo run --release -p nexuslink-codegen

# Run the Flutter client
cd client/flutter
flutter pub get
flutter run
```

To run client tests:

```bash
# Rust core unit and integration tests
cargo test -p nexuslink-core

# Flutter widget tests
cd client/flutter
flutter test
```

### Server Development

```bash
# Clone the repository
git clone https://github.com/yellowpeachxgp/NexusLink.git
cd NexusLink

# Start a local PostgreSQL instance (if not already running)
# Option A: Using Docker
docker run -d --name nexuslink-pg \
  -p 5432:5432 \
  -e POSTGRES_USER=nexuslink \
  -e POSTGRES_PASSWORD=dev_password \
  -e POSTGRES_DB=nexuslink_dev \
  postgres:15

# Option B: Use docker-compose for the full stack
cd server/community
docker compose up -d
cd ../..

# Run database migrations
DATABASE_URL="postgres://nexuslink:dev_password@localhost:5432/nexuslink_dev" \
  cargo run -p nexuslink-community-server -- migrate

# Build and run the Community Server in development mode
cargo run -p nexuslink-community-server -- --config server/community/config.dev.toml

# Build and run the Directory Server in development mode
cargo run -p nexuslink-directory-server -- --config server/directory/config.dev.toml
```

To run server tests:

```bash
# All server tests (requires a running PostgreSQL instance)
DATABASE_URL="postgres://nexuslink:dev_password@localhost:5432/nexuslink_test" \
  cargo test -p nexuslink-community-server -p nexuslink-directory-server -p nexuslink-shared

# Run with logging enabled for debugging
RUST_LOG=debug DATABASE_URL="..." cargo test -p nexuslink-community-server -- --nocapture
```

---

## Roadmap

See [ROADMAP.md](./ROADMAP.md) for the full development plan.

| Phase | Focus | Status |
|-------|-------|--------|
| Phase 0 | Project setup, CI/CD, protocol specs | Planned |
| Phase 1 | Identity system, E2E encryption core | Planned |
| Phase 2 | P2P messaging (Mode C) | Planned |
| Phase 3 | Community server (Mode B) | Planned |
| Phase 4 | Directory server + KYC (Mode A) | Planned |
| Phase 5 | Cross-platform release and polish | Planned |
| Phase 6 | Advanced features (voice, file, ZKP) | Planned |

---

## Product Positioning & Competitive Analysis

### Product Positioning

NexusLink's core philosophy is **user choice**: instead of deciding the balance between privacy and convenience for users, let them decide for themselves.

```
Market Positioning Matrix:

                  High Privacy
                       │
          Session      │   NexusLink (Mode C / P2P)
            Briar      │   NexusLink (Mode B / Community)
                       │
  Decentralized ───────┼────────── Centralized
                       │
           XMPP        │   NexusLink (Mode A / Official)
                       │   Signal
                       │   Telegram    Discord
                  Low Privacy

NexusLink is not a single point on this grid -- it's a sliding scale.
Users choose their own position within one app.
```

**One-line positioning**: NexusLink gives users the *choice* of their privacy level -- with Signal's encryption strength, Matrix's self-hosting capability, and Discord's community experience, without forcing you to pick just one.

### Deep Dive: NexusLink vs Matrix

Matrix is the closest existing project to NexusLink's vision. Both support self-hosting and pursue openness and decentralization. But they differ fundamentally at the architectural level:

#### Design Philosophy

| Dimension | Matrix | NexusLink |
|-----------|--------|-----------|
| Core idea | Open federation -- any server can talk to any server | Hierarchical isolation -- servers don't communicate directly; discovery via directory |
| Data model | DAG (Directed Acyclic Graph) -- room state fully replicated across all participating servers | Message routing -- messages only exist on the sender's server, never replicated across servers |
| Design goal | Become the "Email of messaging" (open interconnection) | Become the "VPN + address book of messaging" (privacy isolation + discoverability) |

#### Identity Model

```
Matrix Identity:
  @alice:matrix.org
  ├── Bound to a specific server (matrix.org)
  ├── Requires account registration (username + password)
  ├── Migrating servers = changing identity (must rebuild all trust)
  └── Server admin can reset password = server controls identity

NexusLink Identity:
  nxl:8Wj3kR9... (Ed25519 public key hash)
  ├── Generated locally in device secure hardware
  ├── Zero registration -- no server dependency
  ├── Freely portable across any server, identity unchanged
  └── Mnemonic recovery -- only the user can restore identity
```

This means:
- If a Matrix homeserver shuts down, your identity disappears; you must rebuild all contacts
- NexusLink identities belong to users permanently; if a server shuts down, just join another community

#### Federation vs Hierarchical Isolation

This is the most critical difference.

```
Matrix Federation Model:

  User A on Server 1 sends message to Room X
  Room X has members from Server 1, 2, 3

  Server 1 ───── full room state ─────→ Server 2
      │                                     │
      └───── full room state ─────→ Server 3

  Result:
  - 3 servers each hold a full copy of Room X
  - Any server admin can see the complete member list
  - Message timestamps, membership changes, presence -- all visible
  - This is Matrix's known "metadata leakage" problem

NexusLink Hierarchical Model:

  Directory Server (official)
  │
  ├── Stores ONLY: community name, description, endpoint URL
  │   Does NOT store: member lists, messages, presence
  │
  └── Community A Server        Community B Server
      ├── Member data stays here  ├── Completely independent
      ├── Messages stay here      ├── Cannot see any of Community A's data
      └── Never reported upstream └── Even if the same user is in both

  Result:
  - Directory server is just a "phone book" -- not involved in messaging
  - Community servers are fully isolated, no state replication
  - The official org cannot know anything about your activity in community servers
  - Switching communities = entering an independent world; privacy isolated by design
```

#### Metadata Privacy

This is the core concern of Matrix users and privacy researchers.

| Metadata Type | Matrix Server Visibility | NexusLink Community Server | NexusLink Directory Server |
|--------------|-------------------------|---------------------------|---------------------------|
| Message content | Not visible (if E2E enabled) | Not visible (E2E mandatory) | Not visible |
| Who talks to whom | **All participating homeservers** | Only that community server | **Not visible** |
| Communication timing | **All participating homeservers** | Only that community server | **Not visible** |
| Presence/online status | **All participating homeservers** | Only that community server | **Not visible** |
| Group member lists | **All participating homeservers** | Only that community server | **Not visible** |
| User IP address | User's homeserver | Only that community server | Only when accessing directory |

**Key difference**: In Matrix, a room spanning N servers exposes metadata to N administrators. In NexusLink, no matter how many communities a user joins, each community's metadata is only visible to that community's admin and never aggregated.

#### Encryption Strategy

| Dimension | Matrix | NexusLink |
|-----------|--------|-----------|
| 1:1 encryption | Olm (Double Ratchet variant), optional | Signal Protocol (X3DH + Double Ratchet), **mandatory** |
| Group encryption | Megolm (Sender Key variant), optional | Sender Keys + Signal, **mandatory** |
| Default state | New rooms default to encrypted (post-2023), but old/public rooms often unencrypted | **No unencrypted option** -- all messages are E2E encrypted |
| Key verification | Cross-signing + emoji verification | Cross-signing + QR code / safety numbers |
| Key backup | Server-side encrypted backup (SSSS) | Client-side mnemonic, no server dependency |

#### P2P Capability

| Dimension | Matrix | NexusLink |
|-----------|--------|-----------|
| Status | Experimental (Pinecone project) | First-class citizen (Mode C) |
| Implementation | Custom Pinecone overlay network | libp2p + WebRTC (mature industrial stack) |
| LAN discovery | Research stage | mDNS auto-discovery (no server needed) |
| NAT traversal | Unresolved | STUN/ICE + TURN fallback |
| Production ready | No | Designed to be usable from Phase 2 |
| Offline messaging | Not supported | Supported on LAN (via mDNS) |

#### Server Resources

| Scale | Matrix (Synapse) | Matrix (Conduit) | NexusLink |
|-------|-----------------|------------------|-----------|
| Small (< 500 users) | 2-4 GB RAM, 2 cores | 512 MB RAM, 1 core | 2 GB RAM, 2 cores (incl. DB) |
| Medium (~5000 users) | 8-16 GB RAM, 4 cores | 2-4 GB RAM, 2 cores | 8 GB RAM, 4 cores (incl. DB) |
| Startup time | Synapse: Python, slow | Conduit: Rust, fast | Rust (Axum), fast |
| Maintenance | High (state resolution, federation sync, media store) | Moderate | Low (no federation sync, no DAG) |

**Why NexusLink is lighter**: No DAG state resolution, no cross-server state sync, no federation queries. Each community server handles only its own members and messages.

#### Where Matrix Still Leads (honest assessment)

Matrix remains ahead in these areas:

- **Maturity**: Matrix development started in 2014 (10+ years); NexusLink is just beginning
- **Ecosystem**: Matrix has Element, FluffyChat, SchildiChat, and many bridges (Telegram, Discord, IRC, Slack)
- **Specification completeness**: Matrix Spec is a formal document refined over years
- **Open federation flexibility**: Matrix enables cross-organization collaboration; NexusLink's isolation model requires P2P or shared communities for cross-community messaging
- **Government/enterprise adoption**: French government (Tchap), German military (BwMessenger), and others have deployed Matrix

What NexusLink learned from Matrix:
- Cross-signing approach for multi-device verification
- E2E encryption practices for group scenarios
- Federation metadata leakage lessons (the direct reason NexusLink chose hierarchical isolation)
- Importance of community governance and open specifications

### Comparison with Other Solutions

#### NexusLink vs Signal

| Dimension | Signal | NexusLink |
|-----------|--------|-----------|
| Architecture | Purely centralized (Signal servers) | Hybrid (three modes) |
| Registration | Phone number required | Zero registration |
| Self-hosting | Impossible (code is open but server doesn't support federation) | Core feature |
| Communities | Basic groups (max 1000) | Full community features (channels, permissions, roles) |
| Encryption protocol | Signal Protocol | Signal Protocol (same) |
| Anonymity | Low (phone number required) | High (zero registration, P2P mode) |

**Borrowed**: NexusLink directly adopts Signal Protocol as its encryption foundation -- the most widely audited E2E protocol available.
**Differentiation**: NexusLink provides self-hosting and community features that Signal cannot offer.

#### NexusLink vs Telegram

| Dimension | Telegram | NexusLink |
|-----------|----------|-----------|
| Encryption | E2E only in "secret chats" (MTProto); regular chats visible to server | **All messages E2E mandatory** |
| Server | Closed-source, fully controlled by Telegram Inc. | Fully open-source (AGPLv3), self-hostable |
| Group features | Very mature (channels, supergroups, bots) | Planned (channels, permissions, RBAC) |
| Privacy | Server holds plaintext for most messages | Server can **never** see message plaintext |
| Registration | Phone number required | Zero registration |

**Borrowed**: Telegram's channel/supergroup/bot ecosystem is a key reference for community features.
**Differentiation**: Telegram's fundamental issue is that encryption is not default; most users' messages are visible to the server. NexusLink has no unencrypted option.

#### NexusLink vs Discord

| Dimension | Discord | NexusLink |
|-----------|---------|-----------|
| Encryption | **No E2E encryption** | Mandatory E2E |
| Open source | No (commercial, closed) | Yes (AGPLv3) |
| Self-hosting | Impossible | Core feature |
| Community features | Extremely rich (channels, voice, threads, roles, bots) | Planned, incremental rollout |
| Business model | Nitro subscription, server boosts | Open-source free, optional managed hosting |
| Privacy | Low (all data visible to Discord, subject to subpoena) | High (E2E + community sovereignty) |

**Borrowed**: Discord's community experience (channel categories, roles, permissions, voice channels) is the UX benchmark for NexusLink communities.
**Differentiation**: Discord has zero E2E encryption; all messages are fully visible to Discord Inc. NexusLink community admins cannot see message content either.

#### NexusLink vs Session

| Dimension | Session | NexusLink |
|-----------|---------|-----------|
| Architecture | Purely decentralized (Oxen network / onion routing) | Hybrid (centralized + decentralized + P2P) |
| Registration | None (key generation) | None (secure hardware key) |
| Speed | Slow (high onion routing latency) | Mode A/B: fast (direct server); Mode C: moderate |
| Community features | Limited | Full community features (planned) |
| Groups | Max ~100, basic | Designed for large-scale communities |
| Identity security | Software key | **Hardware secure element** (higher security tier) |

**Borrowed**: Session proved that "no registration" identity models are viable.
**Differentiation**: Session's pure decentralization makes it slow and feature-limited. NexusLink lets users choose community mode when speed and features matter.

#### NexusLink vs XMPP (Jabber)

| Dimension | XMPP | NexusLink |
|-----------|------|-----------|
| Protocol age | 1999 (26 years) | 2025 (brand new) |
| Federation | Yes (open federation, similar to Matrix) | Hierarchical isolation (not federated) |
| Encryption | OMEMO (optional, XEP-0384) | Signal Protocol (mandatory) |
| Client consistency | Poor (different clients, different features) | Unified (official client, Flutter cross-platform) |
| Modern features | Depends on XEP extensions, inconsistent | Natively integrated (voice, files, push) |

**Borrowed**: XMPP proves open protocols can have longevity, but also exposes fragmentation risks.
**Differentiation**: NexusLink learned from XMPP's fragmentation -- provide a unified official client and clear protocol spec, avoiding "protocol supports it but no client implements it."

### Comparison Overview Table

| Feature | NexusLink | Matrix | Signal | Telegram | Discord | Session | XMPP |
|---------|-----------|--------|--------|----------|---------|---------|------|
| Open source | Full (AGPLv3) | Full (Apache) | Client+Server | Client only | No | Full (GPL) | Open protocol |
| No registration | **Yes** | No | No | No | No | **Yes** | No |
| Default E2E | **Mandatory** | Optional | **Mandatory** | No | No | **Mandatory** | Optional |
| Self-hosted server | **Yes** | Yes | No | No | No | No | Yes |
| P2P direct | **Mode C** | Experimental | No | No | No | Onion routing | No |
| Metadata isolation | **Architectural** | No | Partial | No | No | Yes | No |
| Community features | Planned | Mature | Basic | Mature | **Richest** | Limited | Fragmented |
| Trust levels | **3 levels** | 1 level | 1 level | 1 level | 1 level | 1 level | 1 level |
| Identity security | **HW secure element** | Password | PIN | Phone number | Password | Software key | Password |
| Protocol maturity | New (in dev) | Mature (10y) | Mature (10y) | Mature | Mature | Moderate (5y) | Mature (26y) |

### What NexusLink Borrows from Each Project

```
Reference Map:

  Signal Protocol ──────────→ Encryption (X3DH + Double Ratchet)
                               Adopted directly; not reinvented

  Matrix ───────────────────→ Cross-signing multi-device verification
                               Federation metadata leakage lessons
                               → direct reason for hierarchical isolation
                               Community governance and open specs

  Session ──────────────────→ Zero-registration identity model validation
                               Mnemonic recovery mechanism

  Discord ──────────────────→ Community UX benchmark (channels, roles, permissions, voice)
                               Server discovery/recommendation

  Telegram ─────────────────→ Channel/supergroup/bot ecosystem reference
                               Message sync and multi-device experience

  XMPP ─────────────────────→ Long-term viability of open protocols
                               Fragmentation lessons → unified client

  libp2p ───────────────────→ P2P networking stack (not custom-built)
                               NAT traversal and peer discovery

  Briar ─────────────────────→ Offline P2P messaging scenarios
                               Tor integration possibilities
```

---

## Security Model

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| Server compromise | E2E encryption -- server never has plaintext messages |
| Identity theft | Private key in secure hardware -- cannot be exported |
| Metadata surveillance (official) | Directory server has no access to communication data |
| Metadata surveillance (community) | Community server sees metadata but not content; users choose which server to trust |
| Device loss | Mnemonic recovery phrase restores identity on new device |
| Man-in-the-middle | Public key verification via QR code / safety numbers |

### What NexusLink Does NOT Protect Against
- A compromised community server can see communication metadata (who talks to whom, when) -- this is a conscious trade-off for usability
- If a user's device is compromised (malware, root access), the attacker has access to decrypted messages
- P2P mode requires STUN servers for NAT traversal; these servers can see IP addresses (but not message content)

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for detailed guidelines.

We welcome contributions in all areas:
- Protocol design and security review
- Rust core library development
- Flutter UI/UX design and implementation
- Server development (Community Server and Directory Server)
- Documentation and translation
- Security auditing

---

## License

NexusLink is licensed under the [GNU Affero General Public License v3.0](./LICENSE).

This means:
- You can freely use, modify, and distribute NexusLink
- Any modifications must also be open-sourced under AGPLv3
- If you run a modified version as a network service, you must make the source code available to users
- This ensures NexusLink remains open and free for everyone

---

## Acknowledgments

NexusLink builds upon the research and work of many open-source projects and protocols. We respect each referenced project and have clearly documented what NexusLink learned from them and where it diverges:

- [Signal Protocol](https://signal.org/docs/) -- NexusLink's encryption foundation (X3DH + Double Ratchet), adopted directly rather than reinvented
- [Matrix](https://matrix.org/) -- Pioneer of federated messaging; NexusLink drew inspiration from Matrix's cross-signing for multi-device verification and learned from its metadata leakage problems to design hierarchical isolation
- [Session](https://getsession.org/) -- Validated the viability of zero-registration identity models; NexusLink adopted a similar mnemonic recovery mechanism
- [Discord](https://discord.com/) -- UX benchmark for community features (channels, roles, permissions); NexusLink's community features reference Discord's experience
- [Telegram](https://telegram.org/) -- Reference for channel, supergroup, and bot ecosystems
- [XMPP](https://xmpp.org/) -- Proof that open protocols have long-term viability, and also a cautionary tale about fragmentation
- [libp2p](https://libp2p.io/) -- Peer-to-peer networking stack
- [Briar](https://briarproject.org/) -- Reference for offline P2P messaging scenarios
- [nmshd/rust-crypto](https://github.com/nmshd/rust-crypto) -- Cross-platform secure hardware abstraction
- [Axum](https://github.com/tokio-rs/axum) -- Ergonomic and modular web framework for Rust
- [tokio](https://tokio.rs/) -- Asynchronous runtime for Rust

---

<p align="center">
  <strong>Your messages. Your identity. Your choice.</strong>
</p>
