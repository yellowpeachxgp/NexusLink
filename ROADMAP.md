# NexusLink Development Roadmap / 开发路线图

> This document outlines the phased development plan for NexusLink.
> 本文档概述 NexusLink 的分阶段开发计划。

---

## Overview / 概览

```
Phase 0        Phase 1        Phase 2        Phase 3        Phase 4        Phase 5        Phase 6
Foundation     Identity &     P2P Mode       Community      Directory &    Release &      Advanced
基础搭建        Encryption     P2P 模式       Server         KYC            发布打磨        高级功能
               身份与加密                     社区服务器      目录与认证

[████████]───►[████████]───►[████████]───►[████████]───►[████████]───►[████████]───►[████████]
```

---

## Phase 0: Foundation / 基础搭建

**Goal / 目标**: Establish the project infrastructure, development environment, CI/CD pipeline, and protocol specifications.

建立项目基础设施、开发环境、CI/CD 流水线和协议规范。

### Tasks / 任务

| # | Task / 任务 | Details / 详细说明 |
|---|---|---|
| 0.1 | Rust workspace setup | Initialize Cargo workspace with `nexuslink-core`, `nexuslink-community-server`, `nexuslink-directory-server`, `nexuslink-cli` crates. 初始化 Cargo 工作区 |
| 0.2 | Flutter project setup | Create Flutter project with folder structure, configure `flutter_rust_bridge` FFI. 创建 Flutter 项目，配置 FFI |
| 0.3 | CI/CD pipeline | GitHub Actions: lint (clippy + dart analyze), test, build for all platforms. CI/CD 流水线配置 |
| 0.4 | Protocol specification | Write Protocol Buffer schemas for all message types (see ARCHITECTURE.md §9). 编写 Protobuf 协议定义 |
| 0.5 | Development docs | Set up mdBook or similar for protocol specification site. 搭建协议规范文档站 |
| 0.6 | Code style & linting | Configure rustfmt, clippy, dart formatter, commit hooks. 配置代码风格检查 |
| 0.7 | Rust server crate scaffold | Set up `nexuslink-community-server` crate with Axum + tokio + SQLx dependencies. Implement basic HTTP service skeleton: health check endpoint, graceful shutdown, configuration loading (TOML/env). 搭建 Rust 服务端 crate（Axum + tokio + SQLx），实现基本 HTTP 服务骨架：健康检查端点、优雅关闭、配置加载 |
| 0.8 | Database migration tooling | Configure SQLx migrate CLI and embedded migrations. Create initial PostgreSQL schema: `migrations/0001_init.sql` with core tables (`users`, `devices`, `prekeys`, `config`). Set up migration version tracking. 配置 SQLx 数据库迁移工具，创建初始 PostgreSQL schema |
| 0.9 | Docker development environment | Create `docker-compose.dev.yml` with: PostgreSQL 16 (with pgvector extension), Redis 7 (for caching/pub-sub), server binary with cargo-watch hot-reload. Shared volumes for persistent dev data. 搭建 Docker 开发环境（PostgreSQL + Redis + 服务端热重载） |
| 0.10 | Server CI pipeline | GitHub Actions workflow for server: compile check, `cargo test`, clippy lint with deny warnings, `sqlx prepare --check` for offline query verification, migration consistency check. Run against a PostgreSQL service container. 服务端 CI 流水线（编译、测试、clippy、数据库迁移检查） |

### Deliverables / 交付物
- [ ] Compiling Rust workspace with empty crate skeletons
- [ ] Flutter app that launches on iOS, Android, macOS, Windows, Linux
- [ ] `flutter_rust_bridge` FFI calling a hello-world Rust function from Flutter
- [ ] CI/CD green on all platforms
- [ ] Protocol Buffer `.proto` files for core message types
- [ ] AGPLv3 LICENSE file
- [ ] Server crate runs and responds to health check on `GET /healthz`
- [ ] PostgreSQL schema initialized via `sqlx migrate run`
- [ ] `docker-compose -f docker-compose.dev.yml up` brings up full dev environment
- [ ] Server CI pipeline passes on every PR

---

## Phase 1: Identity System & E2E Encryption Core / 身份系统与端到端加密核心

**Goal / 目标**: Implement the cryptographic identity system and the core encryption engine. No networking yet — all tested locally.

实现加密身份系统和核心加密引擎。此阶段不涉及网络——所有测试在本地进行。

### Tasks / 任务

| # | Task / 任务 | Details / 详细说明 |
|---|---|---|
| 1.1 | Secure hardware abstraction | Integrate `nmshd/rust-crypto` (or build abstraction). Key generation on Secure Enclave / TPM / StrongBox with software fallback. 安全硬件抽象层 |
| 1.2 | Identity Key generation | Ed25519 key pair generation. Derive UUID from public key (Base58 encoding). 身份密钥生成 |
| 1.3 | Mnemonic recovery phrase | BIP-39 compatible mnemonic generation. Derive Ed25519 seed from mnemonic via PBKDF2/Argon2. 助记词恢复短语 |
| 1.4 | Device Signing Key | Per-device key in secure hardware. IK signs DSK → device certificate. 设备签名密钥 |
| 1.5 | X3DH key agreement | Implement X3DH: prekey generation, bundle creation, shared secret derivation. X3DH 密钥协商 |
| 1.6 | Double Ratchet | Implement Double Ratchet for 1:1 message encryption/decryption. Forward secrecy & post-compromise security. 双棘轮实现 |
| 1.7 | Sender Keys | Implement Sender Key distribution and symmetric ratchet for group encryption. 发送者密钥（群组加密） |
| 1.8 | Local encrypted storage | SQLCipher integration. Store sessions, keys, messages locally. 本地加密存储 |
| 1.9 | Cross-signing | IK → SSK → DSK signing chain. Device authorization and revocation. 交叉签名 |
| 1.10 | Flutter identity UI | First-launch screen: key generation animation, mnemonic display/verify, connection mode selection. Flutter 身份界面 |
| 1.11 | Server UUID auth module | Implement server-side challenge-response authentication: server issues random nonce, client signs with device key, server verifies signature against registered public key. Token issuance (JWT or opaque token) with configurable expiry. Session management and refresh flow. 服务端 UUID 认证模块：挑战-应答协议实现，令牌签发与会话管理 |
| 1.12 | Prekey storage service | REST API endpoints for prekey bundle management: `POST /v1/prekeys` (upload bundle), `GET /v1/prekeys/{uuid}` (fetch bundle), `DELETE /v1/prekeys/{uuid}/{key_id}` (consume one-time prekey). Auto-replenish threshold notification. Prekey count monitoring endpoint. 预密钥存储服务：接收和提供 prekey bundles 的 REST API |
| 1.13 | Server shared layer foundation | Build shared server infrastructure: unified error handling with structured error codes (`AppError` enum mapped to HTTP status), configuration management (`nexuslink.toml` + environment variable overrides), standardized JSON API response envelope (`{"ok": bool, "data": ..., "error": ...}`), request ID tracing propagation. 服务端共享层基础：统一错误处理、配置管理、API 响应格式、请求追踪 |

### Deliverables / 交付物
- [ ] `nexuslink-core` crate with identity and crypto modules, 90%+ test coverage
- [ ] Can generate identity, display UUID, show mnemonic
- [ ] Can perform X3DH + Double Ratchet between two local instances
- [ ] Can encrypt/decrypt group messages with Sender Keys
- [ ] Flutter app: first-launch onboarding flow complete
- [ ] Benchmark: encryption/decryption throughput on target devices
- [ ] Server authenticates clients via UUID challenge-response protocol
- [ ] Prekey bundle REST API functional with integration tests
- [ ] Unified error handling and API response format adopted across all server endpoints

### Key Risks / 关键风险
- Secure hardware API differences across platforms may cause edge cases
- BIP-39 → Ed25519 derivation path needs careful security review
- Double Ratchet state serialization must be bullet-proof (corruption = lost messages)

---

## Phase 2: P2P Messaging (Mode C) / P2P 消息传递（模式 C）

**Goal / 目标**: Implement direct peer-to-peer messaging with NAT traversal. This is the most self-contained mode and validates the core architecture.

实现带 NAT 穿透的点对点直接消息传递。这是最独立的模式，可验证核心架构。

### Tasks / 任务

| # | Task / 任务 | Details / 详细说明 |
|---|---|---|
| 2.1 | libp2p integration | Integrate libp2p with Noise XX handshake, QUIC transport. libp2p 集成 |
| 2.2 | Peer discovery | Implement mDNS (LAN), DHT (Kademlia), and relay-based discovery. 节点发现 |
| 2.3 | NAT traversal | STUN integration, ICE candidate exchange, UDP hole punching via WebRTC. NAT 穿透 |
| 2.4 | QR code key exchange | Generate/scan QR codes containing UUID + PeerID + relay hints. 二维码密钥交换 |
| 2.5 | Message delivery | Send/receive encrypted messages over libp2p streams. 消息投递 |
| 2.6 | Offline message relay | Optional relay node stores encrypted blobs for offline peers (configurable TTL). 离线消息中继 |
| 2.7 | Typing indicators & receipts | Real-time presence over P2P connection. 输入指示器与回执 |
| 2.8 | Flutter chat UI | Basic chat interface: message list, input bar, read receipts, connection status. Flutter 聊天界面 |
| 2.9 | Contact management | Add/remove contacts, nickname, notes. Stored locally encrypted. 联系人管理 |
| 2.10 | STUN/TURN relay deployment | Configure and package STUN/TURN relay service deployment: bundled coturn configuration templates, Docker image for one-command TURN server deployment, credential provisioning (static or time-limited via HMAC), bandwidth and connection limits. Provide deployment guide for both self-hosted and cloud environments. STUN/TURN 中继服务部署配置：coturn 配置模板、Docker 镜像、凭据管理 |
| 2.11 | Offline message relay service | Server-side encrypted message staging service: accept and store encrypted blobs (max 256KB per message, configurable) for offline recipients, TTL-based automatic expiration and cleanup (cron job or tokio background task), delivery confirmation and blob deletion upon recipient retrieval, storage quota per user to prevent abuse. 离线消息中继服务：接收加密消息暂存、TTL 自动过期清理、投递确认、用户配额管理 |

### Deliverables / 交付物
- [ ] Two NexusLink clients can chat over P2P (same LAN and across Internet)
- [ ] QR code scanning works on mobile
- [ ] Messages delivered when both online; queued at relay when offline
- [ ] Chat UI functional on all 5 platforms
- [ ] NAT traversal success rate > 80% in testing
- [ ] STUN/TURN relay deployable via Docker with documented configuration
- [ ] Offline relay stores and delivers messages with TTL enforcement

### Key Risks / 关键风险
- NAT traversal unreliable on symmetric NATs (carrier-grade NAT) — need TURN fallback
- libp2p Rust + Flutter FFI may have performance overhead on mobile
- Battery/network impact of maintaining P2P connections on mobile

---

## Phase 3: Community Server (Mode B) / 社区服务器（模式 B）

**Goal / 目标**: Build the self-hostable community server and client integration.

构建可自托管的社区服务器及客户端集成。

### Tasks / 任务

| # | Task / 任务 | Details / 详细说明 |
|---|---|---|
| 3.1 | Server framework | Axum-based HTTP/WebSocket server with TLS. Build the core server binary. Detailed: define Axum router with versioned API prefix (`/v1/`), tower middleware stack including `tower-http` layers for structured logging (`TraceLayer`), authentication extraction, rate limiting (`governor`), CORS policy, request body size limits. Implement `AppError` → `IntoResponse` for consistent error JSON. Graceful shutdown via `tokio::signal`. 服务器框架：Axum 路由、tower 中间件栈（日志、认证、限流、CORS）、统一错误处理、优雅关闭 |
| 3.2 | Authentication | UUID-based auth: client signs challenge with device key, server verifies. UUID 认证 |
| 3.3 | Prekey management | Store/serve prekey bundles for members. Auto-replenish notifications. 预密钥管理 |
| 3.4 | Message routing | WebSocket-based real-time message delivery. Queue for offline users. Detailed: WebSocket connection lifecycle management with configurable heartbeat interval (ping/pong, default 30s) and idle timeout, automatic reconnection protocol with exponential backoff on client side, message fan-out to all registered devices of a recipient, offline message queue implemented via PostgreSQL `NOTIFY`/`LISTEN` for single-node or Redis pub/sub for multi-node deployments, message ordering guarantees with server-assigned sequence IDs per channel. 消息路由：WebSocket 连接管理（心跳、断线重连）、消息扇出到多设备、离线消息队列（PostgreSQL NOTIFY 或 Redis pub/sub）、消息排序 |
| 3.5 | Channels & groups | Create/manage channels (text). Permissions: admin, moderator, member. 频道与群组 |
| 3.6 | Media handling | Encrypted file upload/download. Thumbnail generation. Size limits. 媒体处理 |
| 3.7 | Moderation tools | Ban, mute, delete (metadata only — can't read content). Moderation log. 管理工具 |
| 3.8 | Docker packaging | Dockerfile, docker-compose.yml, one-command deploy. Docker 打包 |
| 3.9 | Admin CLI | `nexuslink-cli` for server administration: user management, stats, config. 管理 CLI |
| 3.10 | Flutter server UI | Server browser, channel list, member list, channel chat view. Flutter 服务器界面 |
| 3.11 | Push notifications | FCM (Android), APNs (iOS) integration via community server. 推送通知 |
| 3.12 | Multi-device message fan-out | Server encrypts and delivers to all registered devices of a user. 多设备消息分发 |
| 3.13 | Database design & implementation | Complete PostgreSQL schema design and implementation. Core tables: `members` (uuid, display_name, avatar_hash, joined_at, role, status), `devices` (device_id, member_uuid, public_key, device_name, last_seen), `channels` (id, name, type, topic, created_by, created_at, sort_order), `channel_members` (channel_id, member_uuid, permissions, joined_at), `messages` (id, channel_id, sender_uuid, encrypted_payload, sequence_id, sent_at, edited_at), `media` (id, message_id, encrypted_blob_path, mime_type, size_bytes, thumbnail_path), `prekeys` (id, member_uuid, key_data, uploaded_at, consumed), `audit_log` (id, actor_uuid, action, target_type, target_id, metadata_json, timestamp), `server_config` (key, value, updated_at). Indexes on foreign keys, composite indexes for common query patterns (e.g., messages by channel + sequence_id). Proper use of PostgreSQL features: ULID/UUIDv7 for ordered primary keys, JSONB for flexible metadata, `timestamptz` for all timestamps. 数据库设计与实现：完整的 PostgreSQL schema，包括 members、channels、messages、media、audit_log 等所有表定义，索引优化 |
| 3.14 | Redis cache layer | Implement Redis-backed caching and real-time data layer: online presence tracking (member UUID → last heartbeat timestamp, with configurable TTL for "online" threshold), session cache (auth tokens → session data, avoiding repeated DB lookups), rate limiting counters (sliding window via Redis sorted sets or `INCR`+`EXPIRE`), channel unread counts cache, typing indicator pub/sub. Use `deadpool-redis` connection pool. Define clear cache invalidation strategy and key naming conventions (`nexuslink:{entity}:{id}:{field}`). Redis 缓存层：在线状态追踪、会话缓存、速率限制计数器、未读消息计数、输入指示器 pub/sub |
| 3.15 | Server performance testing | Establish performance baseline and optimization: load testing with wrk (HTTP endpoints) and k6 (WebSocket scenarios), target benchmarks: 10,000 concurrent WebSocket connections per node, < 50ms p99 message delivery latency, 1,000 messages/second throughput per channel. Database query profiling with `EXPLAIN ANALYZE`, index tuning, connection pool sizing (`sqlx::PgPoolOptions`). Identify and resolve N+1 query patterns. Memory profiling with `jemalloc` + `heaptrack`. Document performance characteristics and capacity planning guidelines. 服务端性能测试：wrk/k6 压测、WebSocket 并发测试、数据库查询优化、内存分析、容量规划 |
| 3.16 | Server security hardening | Comprehensive server-side security measures: TLS 1.3 configuration (via `rustls`, reject TLS < 1.2), strict request validation (content-type checking, payload size limits, malformed JSON rejection), SQL injection prevention verified (all queries via SQLx compile-time checked parameterized queries), input sanitization and validation layer (`validator` crate for struct field constraints), HTTP security headers (HSTS, X-Content-Type-Options, X-Frame-Options), authentication token rotation and revocation, brute-force protection on auth endpoints (exponential backoff + lockout), dependency audit via `cargo audit` in CI. 服务端安全加固：TLS 配置、请求验证、SQL 注入防护（SQLx 参数化查询）、输入校验、安全响应头、认证令牌轮转、暴力破解防护 |

### Deliverables / 交付物
- [ ] Community server deployable via `docker-compose up`
- [ ] Clients can register, join channels, send/receive messages
- [ ] File sharing works (images, documents, voice messages)
- [ ] Admin CLI can manage server
- [ ] Push notifications on mobile
- [ ] Load test: 1000 concurrent users on single community server
- [ ] Complete PostgreSQL schema with all tables, indexes, and migration files
- [ ] Redis cache layer operational with documented key schema
- [ ] Performance test suite with reproducible benchmarks
- [ ] Security hardening checklist fully implemented and verified

### Key Risks / 关键风险
- WebSocket scaling under load — may need connection pooling
- Push notification requires Firebase/APNs accounts (infrastructure dependency)
- Encrypted media storage can grow quickly — need storage management
- Redis as a dependency increases operational complexity for self-hosters — provide embedded fallback or clear documentation

---

## Phase 4: Directory Server & KYC (Mode A) / 目录服务器与实名认证（模式 A）

**Goal / 目标**: Build the official directory server, KYC system, and community registration.

构建官方目录服务器、实名认证系统和社区注册。

### Tasks / 任务

| # | Task / 任务 | Details / 详细说明 |
|---|---|---|
| 4.1 | Directory server | REST API for community registration, search, deregistration. Detailed: full CRUD endpoints (`POST /v1/communities`, `GET /v1/communities`, `GET /v1/communities/{id}`, `PATCH /v1/communities/{id}`, `DELETE /v1/communities/{id}`), PostgreSQL full-text search using `tsvector`/`tsquery` on community name, description, and tags (with `pg_trgm` for fuzzy matching), paginated responses with cursor-based pagination (keyset pagination for stable ordering), sorting by relevance, member count, creation date, or last active time. OpenAPI/Swagger spec auto-generated via `utoipa`. 目录服务器：REST API 完整实现、PostgreSQL 全文搜索（tsvector + pg_trgm 模糊匹配）、游标分页、多维度排序、OpenAPI 文档 |
| 4.2 | KYC integration | Identity verification service. Options: integrate third-party KYC provider or build simple document-based verification. 实名认证集成 |
| 4.3 | Community registration flow | Community server → directory server: submit name, description, endpoint, owner UUID. 社区注册流程 |
| 4.4 | Community search | Full-text search, tag-based filtering, sorting. Client-side community browser. 社区搜索 |
| 4.5 | Abuse reporting | Report mechanism from client → directory server. Review queue for admins. 滥用举报 |
| 4.6 | Official channels | Host official channels on directory server's built-in community server instance. 官方频道 |
| 4.7 | Rate limiting & anti-spam | Request rate limits, proof-of-work for registration, anomaly detection. 限流与反垃圾 |
| 4.8 | Flutter directory UI | Community discovery screen, KYC flow, official channel access. Flutter 目录界面 |
| 4.9 | KYC data security | Secure handling of identity verification data: encrypt all KYC documents at rest (AES-256-GCM with per-document keys, master key in HSM or vault), comprehensive access audit trail (who accessed which document, when, why), data retention policy (auto-delete verified documents after configurable period, default 90 days, retain only verification status), GDPR compliance considerations (right to erasure implementation, data portability export, explicit consent tracking, DPO contact endpoint), data minimization (store only what is strictly necessary for verification). KYC 数据安全：加密存储身份文件、访问审计、数据保留策略、GDPR 合规考虑、数据最小化 |
| 4.10 | Directory server high availability | Production-grade availability for the directory server: PostgreSQL primary-replica replication (streaming replication with `pg_basebackup`), read-only replicas for search queries (route via SQLx `PgPoolOptions` with separate read/write pools), DDoS protection layer (Cloudflare or similar reverse proxy with WAF rules, geographic rate limiting), horizontal scaling considerations (stateless API servers behind load balancer), automated failover for database (Patroni or similar), uptime target: 99.9%. 目录服务器高可用：数据库主从复制、只读副本、DDoS 防护、水平扩展、自动故障转移 |
| 4.11 | Community server health monitoring | Directory server actively monitors registered communities: periodic health checks (HTTP `GET /healthz` every 5 minutes, configurable), track response latency and success rate over rolling window, automatic status marking (online / degraded / offline) based on health check results, notify community owners via stored contact method when status changes, display real-time status indicators in client community browser, deregister communities that have been offline beyond a configurable threshold (default 30 days) with owner notification grace period. 社区服务器健康监测：定期检查已注册社区的可用性，标记下线社区，通知管理员，自动清理长期离线社区 |

### Deliverables / 交付物
- [ ] Directory server deployed and running
- [ ] KYC flow: submit → review → approved/rejected
- [ ] Community servers can register and appear in search
- [ ] Client can browse, search, and join communities via directory
- [ ] Abuse reporting works end-to-end
- [ ] Rate limiting prevents spam registration
- [ ] KYC documents encrypted at rest with full access audit trail
- [ ] Directory server running with read replicas and DDoS protection
- [ ] Health monitoring active for all registered communities

### Key Risks / 关键风险
- KYC requires handling sensitive personal data — legal and security implications
- Directory server is a centralization point — needs DDoS protection
- KYC review process needs human reviewers or AI-assisted verification
- GDPR and data protection regulation compliance across jurisdictions requires legal review

---

## Phase 5: Cross-Platform Release & Polish / 全平台发布与打磨

**Goal / 目标**: Polish the app for public release, optimize performance, and prepare distribution.

打磨应用以准备公开发布，优化性能，准备分发。

### Tasks / 任务

| # | Task / 任务 | Details / 详细说明 |
|---|---|---|
| 5.1 | UI/UX polish | Design system, animations, dark mode, accessibility. UI/UX 打磨 |
| 5.2 | Internationalization | i18n: English, Chinese (Simplified), Chinese (Traditional). More languages via community. 国际化 |
| 5.3 | Performance optimization | Profile and optimize: startup time, message throughput, battery usage, memory. 性能优化 |
| 5.4 | Security audit | Engage third-party security firm for cryptographic protocol and implementation audit. 安全审计 |
| 5.5 | App store preparation | iOS App Store, Google Play Store, F-Droid, Microsoft Store, Snapcraft/Flatpak. 应用商店准备 |
| 5.6 | Documentation | User guide, server deployment guide, FAQ, troubleshooting. 文档完善 |
| 5.7 | Beta testing program | Closed beta → open beta. Feedback collection and bug tracking. Beta 测试 |
| 5.8 | Landing page & website | Project website with download links, docs, and community links. 官网 |
| 5.9 | Server production deployment docs | Comprehensive server deployment documentation: single-node deployment guide (systemd unit file, environment configuration, reverse proxy with nginx/caddy), multi-node cluster deployment guide (load balancer setup, shared PostgreSQL, Redis cluster, session affinity considerations), TLS certificate management (Let's Encrypt auto-renewal via certbot or caddy, manual certificate installation guide, certificate pinning considerations for clients), hardware sizing recommendations by user count tier (100 / 1,000 / 10,000 users). 服务端生产部署文档：单机部署指南、集群部署指南、TLS 证书管理、硬件规模建议 |
| 5.10 | Automated operations scripts | Server operations automation toolkit: database backup script (pg_dump with compression, scheduled via cron, upload to S3-compatible storage with retention rotation), database restore and point-in-time recovery procedure, zero-downtime upgrade script (rolling restart for multi-node, database migration pre-check), log rotation configuration (logrotate for file-based logs, or structured JSON logs to stdout for container environments), disk space monitoring and media storage cleanup for expired/orphaned files. 自动化运维脚本：备份、恢复、升级、日志轮转、磁盘监控 |
| 5.11 | Monitoring and alerting | Production observability stack: expose Prometheus metrics from server (`/metrics` endpoint via `metrics` + `metrics-exporter-prometheus` crates), key metrics: active WebSocket connections, message throughput, request latency histograms, database connection pool usage, error rates by type, queue depths. Grafana dashboard templates: server overview, per-channel activity, system resources, alert history. Alerting rules: high error rate, database connection exhaustion, disk space low, certificate expiry approaching, health check failures. 监控告警搭建：Prometheus 指标暴露 + Grafana 仪表盘模板 + 告警规则 |

### Deliverables / 交付物
- [ ] App published on all major stores/platforms
- [ ] Security audit report (no critical/high findings)
- [ ] < 3 second cold start on mid-range devices
- [ ] User documentation complete in 2 languages
- [ ] Beta community of 100+ testers
- [ ] Server deployment guide covering single-node and cluster topologies
- [ ] Automated backup/restore/upgrade scripts tested and documented
- [ ] Grafana dashboard importable via JSON template, alerting rules ready for production

---

## Phase 6: Advanced Features / 高级功能

**Goal / 目标**: Extend NexusLink with advanced capabilities based on user feedback and community needs.

根据用户反馈和社区需求扩展 NexusLink 的高级功能。

### Planned Features / 计划功能

| Feature / 功能 | Description / 描述 | Priority / 优先级 |
|---|---|---|
| **Voice Messages** | Record, encrypt, send voice clips. 语音消息 | High |
| **Voice/Video Calls** | 1:1 E2E encrypted calls via WebRTC. 语音/视频通话 | High |
| **Group Voice Channels** | Discord-style voice rooms in communities. 群组语音频道 | Medium |
| **ZKP-based KYC** | Zero-knowledge proof: prove identity without revealing details. 零知识证明认证 | Medium |
| **Message search** | Client-side full-text search over encrypted message database. 消息搜索 | Medium |
| **Bots & Integrations** | Bot API for community automation (RSS, notifications, moderation). 机器人与集成 | Medium |
| **Encrypted backup** | Cloud-encrypted backup of messages (user holds key). 加密备份 | Medium |
| **WebSocket cluster** | Multi-node WebSocket message synchronization: shared message bus (Redis Streams or NATS), sticky sessions with failover, consistent hashing for channel-to-node assignment, zero-message-loss node migration during rolling deployments. WebSocket 集群（多节点消息同步） | Medium |
| **Server plugin system** | Extensible server-side plugin architecture: plugin trait definition (`NexusPlugin` with lifecycle hooks: `on_load`, `on_message`, `on_member_join`, etc.), WASM-based sandboxed execution for third-party plugins, built-in plugins for common needs (welcome messages, auto-moderation, webhook relay), plugin marketplace metadata format. 服务端插件系统（WASM 沙箱执行、生命周期钩子、插件市场） | Medium |
| **GraphQL API** | Optional GraphQL endpoint alongside REST: schema stitching community server data, subscription support for real-time updates (backed by WebSocket), introspection for developer tooling, query complexity analysis and depth limiting to prevent abuse. GraphQL API（可选，实时订阅、查询复杂度限制） | Low |
| **S3-compatible object storage** | Pluggable media storage backend: support S3-compatible APIs (AWS S3, MinIO, Cloudflare R2) for encrypted media blobs, configurable storage backend (local filesystem / S3), automatic migration tool between storage backends, content-addressable storage for deduplication of identical encrypted blobs. S3 兼容对象存储（可插拔存储后端、去重） | Low |
| **Disappearing messages** | Auto-delete messages after configurable time. 阅后即焚 | Low |
| **Reactions & threads** | Emoji reactions and threaded replies. 表情回应与消息线程 | Low |
| **Custom themes** | Community-specific themes and branding. 自定义主题 | Low |
| **Bridge to other protocols** | Matrix, XMPP, IRC bridges. 协议桥接 | Low |
| **PIR directory queries** | Private Information Retrieval for directory lookups. 私密目录查询 | Research |
| **Post-quantum crypto** | Hybrid key exchange with ML-KEM (Kyber). 后量子密码学 | Research |

---

## Practical Analysis / 实际情况分析

### Target Users / 目标用户

| User Segment / 用户群体 | Why NexusLink / 为什么选 NexusLink | Mode / 使用模式 |
|---|---|---|
| **Open-source communities** 开源社区 | Self-host, own their data, avoid Discord/Slack lock-in 自托管，数据自主 | Mode B (community server) |
| **Privacy-conscious individuals** 隐私敏感个人用户 | No registration, P2P option, choose trust level 无需注册，P2P 可选 | Mode C (P2P) + Mode B |
| **Small organizations / teams** 小型组织/团队 | Self-hosted, compliant, no third-party data exposure 合规自建，数据不外流 | Mode B |
| **Journalists / activists** 记者/活动人士 | Censorship-resistant P2P, no phone number required 抗审查 P2P 通讯 | Mode C (P2P) |
| **General users** 普通用户 | Easy onboarding via official server, community discovery 通过官方服务器便捷入门 | Mode A (official) |
| **Regulated industries** 合规行业 | KYC gate + self-hosted server = privacy + compliance 实名认证 + 自建 = 隐私 + 合规 | Mode A + Mode B |

### Competitive Advantages / 竞争优势

```
1. Unique Hybrid Architecture / 独特的混合架构
   No other app lets users seamlessly switch between
   centralized, federated, and P2P in one interface.
   没有其他应用能让用户在同一界面无缝切换三种模式。

2. Zero-Registration Identity / 零注册身份
   Hardware-backed cryptographic identity.
   No email, no phone, no account creation.
   硬件级加密身份，无需任何注册信息。

3. Graduated Trust Model / 渐进式信任模型
   Users choose their trust level:
   用户自主选择信任级别：
     P2P → trust nobody but yourself
     Community → trust a specific server operator
     Official → trust the NexusLink organization + KYC

4. Community Data Sovereignty / 社区数据主权
   Community operators own their data.
   Official server cannot read, access, or subpoena it.
   社区运营者拥有自己的数据。官方服务器无法读取或传唤。

5. One Codebase, Two Roles / 一套代码，两种角色
   Same server binary runs as community server or directory server.
   Lower maintenance, consistent quality.
   同一服务端二进制文件可用作社区服务器或目录服务器。
```

### Challenges & Mitigations / 挑战与应对

| Challenge / 挑战 | Impact / 影响 | Mitigation / 应对 |
|---|---|---|
| **Network effects** 网络效应 | Hard to attract users without existing users 没有用户就难以吸引用户 | Start with open-source communities that value self-hosting. Offer easy migration from Discord/Matrix. 从重视自托管的开源社区入手，提供 Discord/Matrix 迁移工具 |
| **Secure hardware fragmentation** 安全硬件碎片化 | Different platforms, different APIs 不同平台不同 API | Use `nmshd/rust-crypto` abstraction + software fallback. Extensive device testing matrix. 使用统一抽象层 + 软件回退方案 |
| **NAT traversal reliability** NAT 穿透可靠性 | P2P fails on strict NATs P2P 在严格 NAT 下失败 | TURN relay fallback (still E2E encrypted). Community servers as natural relay points. TURN 中继回退，社区服务器作为天然中继点 |
| **KYC legal complexity** 实名认证法律复杂性 | Different jurisdictions, different requirements 不同司法管辖区不同要求 | Start with a simple document-based flow. Partner with established KYC providers later. 先用简单的文档认证，后续对接专业 KYC 服务商 |
| **Cryptographic complexity** 密码学复杂性 | Bugs in crypto = catastrophic failures 密码学 bug = 灾难性后果 | Use well-audited libraries (dalek, RustCrypto). Hire external security audit before launch. 使用经过审计的库，上线前做外部安全审计 |
| **User experience** 用户体验 | Three modes can confuse users 三种模式可能让用户困惑 | Smart defaults: official mode for new users, unlock other modes in settings. 智能默认值：新用户默认官方模式，其他模式在设置中解锁 |
| **Funding / sustainability** 资金/可持续性 | Open-source projects need funding 开源项目需要资金 | Community donations, optional premium features for large deployments, grants from privacy foundations. 社区捐赠、大规模部署的可选付费功能、隐私基金会拨款 |

### Revenue Model Options (Open Source Compatible) / 收入模式（兼容开源）

```
1. Donations & Sponsorship / 捐赠与赞助
   GitHub Sponsors, Open Collective, etc.

2. Managed Hosting / 托管服务
   Offer paid managed community server hosting
   for non-technical organizations.
   为非技术组织提供付费的社区服务器托管。

3. Enterprise Support / 企业支持
   Paid support contracts, SLA, custom deployment.
   付费支持合同、SLA、定制部署。

4. Premium Directory Features / 高级目录功能
   Promoted community listings, verified badges.
   推广社区排名、认证徽章。

5. Grants / 基金拨款
   Apply to NLnet, Open Technology Fund, etc.
   申请 NLnet、开放技术基金等。
```

---

## Version Milestones / 版本里程碑

| Version | Content / 内容 | Server Content / 服务端内容 | Corresponds to |
|---------|---|---|---|
| `v0.1.0-alpha` | Identity + encryption + local test | Server crate scaffold, DB migrations, dev Docker, UUID auth module, prekey API, shared layer | Phase 0 + 1 |
| `v0.2.0-alpha` | P2P messaging works | STUN/TURN relay deployment, offline message relay service | Phase 2 |
| `v0.3.0-alpha` | Community server functional | Full DB schema, Redis cache layer, WebSocket message routing, middleware stack, performance baseline, security hardening | Phase 3 |
| `v0.4.0-beta` | Directory server + KYC + all 3 modes | Full-text search API, KYC data encryption, high availability setup, community health monitoring | Phase 4 |
| `v1.0.0` | First stable public release | Production deployment docs, backup/restore/upgrade scripts, Prometheus + Grafana monitoring | Phase 5 |
| `v1.x` | Voice, video, ZKP, bridges | WebSocket cluster, plugin system, GraphQL API (optional), S3-compatible object storage | Phase 6 |

---

## How to Contribute to Roadmap / 如何参与路线图

The roadmap is a living document. We welcome community input:

路线图是一个动态文档，欢迎社区参与：

1. **Feature requests**: Open a GitHub Issue with the `feature-request` label
2. **Priority feedback**: Comment on existing roadmap issues
3. **Implementation**: Pick a task and submit a PR (see CONTRIBUTING.md)
4. **Security review**: Review protocol specs and report concerns

---

<p align="center">
  <em>Last updated / 最后更新: 2026-02</em>
</p>
