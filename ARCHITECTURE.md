# NexusLink Architecture / 架构设计文档

> This document provides a detailed technical breakdown of NexusLink's architecture.
> 本文档提供 NexusLink 架构的详细技术说明。

---

## Table of Contents / 目录

1. [System Overview / 系统概览](#1-system-overview--系统概览)
2. [Identity System / 身份系统](#2-identity-system--身份系统)
3. [Encryption Protocol / 加密协议](#3-encryption-protocol--加密协议)
4. [Multi-Device Sync / 多设备同步](#4-multi-device-sync--多设备同步)
5. [Network Topology / 网络拓扑](#5-network-topology--网络拓扑)
6. [Directory Server / 目录服务器](#6-directory-server--目录服务器)
7. [Community Server / 社区服务器](#7-community-server--社区服务器)
8. [P2P Protocol / P2P 协议](#8-p2p-protocol--p2p-协议)
9. [Message Format / 消息格式](#9-message-format--消息格式)
10. [Storage / 存储](#10-storage--存储)
11. [Security Analysis / 安全性分析](#11-security-analysis--安全性分析)
12. [Server Shared Layer / 服务端共享层](#12-server-shared-layer--服务端共享层)
13. [Server Deployment Architecture / 服务端部署架构](#13-server-deployment-architecture--服务端部署架构)
14. [Bot System / 机器人系统](#14-bot-system--机器人系统)

---

## 1. System Overview / 系统概览

NexusLink is a three-tier hybrid messaging system:

NexusLink 是一个三层混合架构即时通讯系统：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Application Layer                            │
│                         应用层                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │ Flutter UI  │  │ Notification │  │ Media (images/voice/file) │  │
│  │ 用户界面     │  │ 通知服务      │  │ 媒体处理                   │  │
│  └──────┬──────┘  └──────┬───────┘  └────────────┬──────────────┘  │
│         └────────────────┼───────────────────────┘                  │
│                          │ FFI                                      │
├──────────────────────────┼──────────────────────────────────────────┤
│                    Core Logic Layer                                  │
│                     核心逻辑层                                       │
│  ┌───────────┐ ┌────────────┐ ┌──────────┐ ┌───────────────────┐   │
│  │ Identity  │ │ Encryption │ │ Protocol │ │ Network Manager   │   │
│  │ 身份管理   │ │ 加密引擎    │ │ 协议编码  │ │ 网络管理器         │   │
│  └───────────┘ └────────────┘ └──────────┘ └───────────────────┘   │
│  ┌───────────────────────┐ ┌────────────────────────────────────┐   │
│  │ Session Manager       │ │ Storage Engine (SQLCipher)         │   │
│  │ 会话管理器             │ │ 存储引擎                           │   │
│  └───────────────────────┘ └────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────┤
│                    Transport Layer                                   │
│                     传输层                                           │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────────────┐  │
│  │ HTTPS/TLS    │  │ WebSocket/    │  │ libp2p / WebRTC         │  │
│  │ (REST/gRPC)  │  │ QUIC          │  │ (P2P direct)            │  │
│  │ 服务器通讯    │  │ 实时消息       │  │ 点对点直连               │  │
│  └──────────────┘  └───────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.1 System Panorama / 系统全景

The following diagram shows the complete NexusLink system topology -- how the Directory Server, Community Servers, Client Apps, and P2P connections relate to each other.

下图展示了 NexusLink 完整的系统拓扑——目录服务器、社区服务器、客户端 App 和 P2P 连接之间的关系。

```
NexusLink 系统全景 / System Panorama

                    ┌─────────────────────────────┐
                    │      Directory Server        │
                    │        目录服务器              │
                    │                               │
                    │  • Community registry         │
                    │    社区注册表                   │
                    │  • Search / discovery         │
                    │    搜索 / 发现                  │
                    │  • KYC verification           │
                    │    KYC 身份验证                 │
                    │  • Official Plaza             │
                    │    官方广场（内置社区实例）       │
                    └──────┬──────────────┬─────────┘
                           │ REST/HTTPS   │ REST/HTTPS
              ┌────────────┘              └────────────┐
              ▼                                        ▼
 ┌────────────────────────┐            ┌────────────────────────┐
 │   Community Server A   │            │   Community Server B   │
 │     社区服务器 A        │            │     社区服务器 B        │
 │                        │            │                        │
 │  • WebSocket routing   │            │  • WebSocket routing   │
 │  • Prekey management   │            │  • Prekey management   │
 │  • Channels / RBAC     │            │  • Channels / RBAC     │
 │  • Media storage       │            │  • Media storage       │
 │  • Push notifications  │            │  • Push notifications  │
 └───────┬────────────────┘            └──────────────┬────────┘
         │ WebSocket                        WebSocket │
         │                                            │
         ▼                                            ▼
 ┌───────────────┐    P2P (WebRTC/mDNS)    ┌───────────────┐
 │  Client App   │◄───────────────────────►│  Client App   │
 │  客户端 App    │                         │  客户端 App    │
 │               │                         │               │
 │  Rust Core    │                         │  Rust Core    │
 │  + Flutter UI │                         │  + Flutter UI │
 └───────┬───────┘                         └───────┬───────┘
         │ REST/HTTPS                     REST/HTTPS│
         └──────────────┐      ┌────────────────────┘
                        ▼      ▼
              ┌─────────────────────────┐
              │   Community Server C    │
              │     社区服务器 C         │
              │   (or any other server) │
              │  （或任何其他服务器）      │
              └─────────────────────────┘

Key / 图例:
  ───► REST/HTTPS  : Client ↔ Directory Server (registration, search)
                     客户端 ↔ 目录服务器（注册、搜索）
  ───► WebSocket   : Client ↔ Community Server (real-time messaging)
                     客户端 ↔ 社区服务器（实时消息）
  ◄──► P2P         : Client ↔ Client (direct encrypted communication)
                     客户端 ↔ 客户端（直接加密通讯）
```

**Key design points / 关键设计要点:**

- Each Client App can simultaneously connect to the Directory Server and multiple Community Servers, plus establish P2P links with other clients.
- Community Servers are completely isolated from each other -- they never communicate directly.
- The Directory Server only knows that communities exist; it has no visibility into community membership or messages.

- 每个客户端 App 可以同时连接目录服务器和多个社区服务器，还可以与其他客户端建立 P2P 链接。
- 社区服务器之间完全隔离——它们之间从不直接通讯。
- 目录服务器只知道社区的存在；它无法看到社区成员或消息。

### 1.2 Four Roles and Their Boundaries / 四个角色的职责边界

NexusLink has exactly four types of participants. Each has a strictly defined scope of what it can and cannot do.

NexusLink 恰好有四种参与者。每种参与者都有严格定义的能力范围。

#### Client App / 客户端 App

The Client App is the user's device application, composed of two layers connected via FFI:

客户端 App 是用户的设备应用程序，由通过 FFI 连接的两层组成：

```
┌─────────────────────────────────────────────────┐
│              Client App Architecture             │
│                客户端 App 架构                     │
├─────────────────────────────────────────────────┤
│  Flutter UI (Application Layer / 应用层)         │
│  • Chat interface / 聊天界面                      │
│  • Contact management / 联系人管理                │
│  • Community browsing / 社区浏览                  │
│  • Settings / 设置                                │
├──────────────────────┬──────────────────────────┤
│                      │ FFI                       │
├──────────────────────┴──────────────────────────┤
│  Rust Core (Identity & Crypto Layer / 身份加密层) │
│  • Key generation & management / 密钥生成与管理   │
│  • E2E encryption/decryption / 端到端加解密       │
│  • Signal Protocol (Double Ratchet) / 信号协议    │
│  • UUID generation / UUID 生成                    │
│  • Local DB encryption / 本地数据库加密            │
└─────────────────────────────────────────────────┘
```

**Simultaneous connections / 同时连接:**

| Connection / 连接 | Protocol / 协议 | Purpose / 用途 |
|---|---|---|
| Directory Server / 目录服务器 | REST/HTTPS | Community discovery, KYC / 社区发现、KYC |
| Community A, B, ... / 社区 A, B, ... | WebSocket | Real-time messaging / 实时消息 |
| Other clients / 其他客户端 | WebRTC / mDNS | P2P direct chat / P2P 直连聊天 |

**Local storage / 本地存储:**

- SQLCipher encrypted database for messages, contacts, and session state / SQLCipher 加密数据库存储消息、联系人和会话状态
- OS secure storage (Keychain/Keystore) or hardware key for private key protection / 操作系统安全存储（Keychain/Keystore）或硬件密钥保护私钥

**Key principle / 核心原则:** The client is the only place in the entire system that ever sees plaintext message content. All encryption and decryption happens exclusively in the Rust Core on the user's device.

**核心原则：** 客户端是整个系统中唯一能看到明文消息内容的地方。所有加密和解密都只在用户设备上的 Rust Core 中进行。

#### Community Server / 社区服务器

A Community Server hosts one community. It is the real-time message relay and community management hub, but it is cryptographically blind to message content.

社区服务器承载一个社区。它是实时消息中继和社区管理中心，但在密码学上对消息内容完全不可见。

**What it CAN see / 它能看到的:**

- Member UUIDs and online/offline status / 成员 UUID 和在线/离线状态
- Who communicates with whom (sender UUID → receiver UUID) / 谁和谁通讯（发送者 UUID → 接收者 UUID）
- Message timing and size / 消息时间和大小
- Member IP addresses / 成员 IP 地址

**What it CANNOT see / 它不能看到的:**

- Message plaintext or decrypted file content / 消息明文或解密后的文件内容
- User private keys / 用户私钥
- Data from other Community Servers / 其他社区服务器的数据

**Core functions / 核心功能:**

1. **WebSocket message routing / WebSocket 消息路由** -- 1:1 messages, group messages, offline message queuing / 一对一消息、群组消息、离线消息排队
2. **Prekey management / 预密钥管理** -- Stores public prekey bundles so users can initiate E2E sessions even when the recipient is offline / 存储公开预密钥包，使用户即使在接收者离线时也能发起 E2E 会话
3. **Channels and RBAC / 频道与权限控制** -- Channel creation, role-based access control, moderation / 频道创建、基于角色的访问控制、内容管理
4. **Media storage / 媒体存储** -- Stores encrypted media blobs uploaded by clients / 存储客户端上传的加密媒体文件
5. **Push notifications / 推送通知** -- Sends push notification triggers (without message content) / 发送推送通知触发（不含消息内容）

**What it does NOT do / 它不做的事:**

- No inter-server communication -- Community Servers never talk to each other / 不进行服务器间通讯——社区服务器之间从不互相通讯
- No reporting to Directory Server -- it does not send activity data upstream / 不向目录服务器报告——不向上游发送活动数据
- No decryption -- it has no keys to decrypt any message / 不进行解密——它没有任何密钥来解密消息
- No real identity verification -- it only knows UUIDs, not real names / 不进行真实身份验证——它只知道 UUID，不知道真实姓名

#### Directory Server / 目录服务器

The Directory Server is the public-facing registry and discovery service for communities. It is the lightest component in terms of data access.

目录服务器是面向公众的社区注册和发现服务。它在数据访问方面是最轻量的组件。

**What it CAN see / 它能看到的:**

- Registered community names and server addresses / 已注册的社区名称和服务器地址
- KYC verification status of community operators / 社区运营者的 KYC 验证状态
- Search queries from clients / 来自客户端的搜索查询
- Client IP addresses when accessing the directory / 客户端访问目录时的 IP 地址

**What it CANNOT see / 它不能看到的:**

- Community member lists / 社区成员列表
- Any message content (plaintext or ciphertext) / 任何消息内容（明文或密文）
- Which communities a user has joined / 用户加入了哪些社区
- Who is communicating with whom / 谁在和谁通讯

**Core functions / 核心功能:**

| Function / 功能 | Description / 描述 |
|---|---|
| A. Community registration & discovery / 社区注册与发现 | Communities register their name, description, and address. Clients browse or search. / 社区注册名称、描述和地址。客户端浏览或搜索。 |
| B. Community search / 社区搜索 | Full-text search by name, tags, category / 按名称、标签、分类全文搜索 |
| C. KYC verification / KYC 身份验证 | Optional identity verification for community operators to build trust / 可选的社区运营者身份验证以建立信任 |
| D. Official Plaza / 官方广场 | A built-in Community Server instance operated alongside the Directory for public discussion / 与目录服务器一起运营的内置社区服务器实例，用于公共讨论 |

**What it does NOT do / 它不做的事:**

- No message forwarding -- it never touches any message / 不转发消息——它从不接触任何消息
- No member list storage -- it does not know who joined which community / 不存储成员列表——它不知道谁加入了哪个社区
- No key exchange participation -- it plays no role in E2E setup / 不参与密钥交换——它在 E2E 建立中没有任何角色
- No activity monitoring -- it cannot observe user behavior within communities / 不监控活动——它无法观察用户在社区内的行为

#### P2P Direct / P2P 直连

P2P Direct is the serverless communication mode. Two clients connect directly without any intermediary.

P2P 直连是无服务器通讯模式。两个客户端直接连接，没有任何中间人。

**Connection establishment / 连接建立:**

```
Client A                                          Client B
客户端 A                                           客户端 B

  │  1. Exchange public keys (out-of-band or via community)        │
  │     交换公钥（带外方式或通过社区）                                  │
  │                                                                 │
  │  2. STUN NAT traversal / STUN NAT 穿透                         │
  │────────────────────► STUN Server ◄──────────────────────────────│
  │                                                                 │
  │  3. WebRTC DataChannel established / WebRTC DataChannel 建立    │
  │◄═══════════════════════════════════════════════════════════════►│
  │                                                                 │
  │  (LAN alternative: mDNS discovery / 局域网替代方案：mDNS 发现)   │
  │◄═══════════════════════════════════════════════════════════════►│
```

**Data flow / 数据流:**

```
Plaintext → Rust Core encrypt → WebRTC DataChannel → Rust Core decrypt → Plaintext
明文     → Rust Core 加密    → WebRTC DataChannel  → Rust Core 解密   → 明文
```

**Characteristics / 特点:**

- No server = no metadata leakage. No third party knows that communication is happening. / 无服务器 = 无元数据泄露。没有第三方知道通讯正在发生。
- No offline messaging capability (unless both devices are on the same LAN via mDNS). / 无离线消息能力（除非两台设备通过 mDNS 在同一局域网内）。
- Highest privacy tier in NexusLink. / NexusLink 中最高隐私级别。

### 1.3 User Journey Map / 用户完整旅程

This section traces a new user's complete journey from installation to active communication.

本节追踪新用户从安装到活跃通讯的完整旅程。

```
User Journey / 用户旅程

 ┌──────────────────────────────────────────────────────────────────┐
 │  1. Install & First Launch (Offline) / 安装与首次启动（离线）      │
 │                                                                  │
 │     • Generate Ed25519 key pair / 生成 Ed25519 密钥对             │
 │     • Derive UUID from public key / 从公钥派生 UUID               │
 │     • Choose security tier (Easy/Standard/Hardware)              │
 │       选择安全级别（便捷/标准/硬件）                                │
 │     • Set display name (local only) / 设置显示名称（仅本地）       │
 │                                                                  │
 │     ⚡ At this point, the user has a full identity with ZERO     │
 │        server contact. / 此时用户已拥有完整身份，零服务器接触。     │
 └──────────────────────────┬───────────────────────────────────────┘
                            │
                            ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  2. Choose Connection Mode / 选择连接模式                         │
 │                                                                  │
 │     ┌─────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
 │     │ A. Browse    │  │ B. Join friend's │  │ C. Direct chat  │  │
 │     │ communities  │  │ community via    │  │ via QR/link     │  │
 │     │ 浏览社区      │  │ invite link      │  │ 通过 QR/链接    │  │
 │     │              │  │ 通过邀请链接加入   │  │ 直接聊天        │  │
 │     │              │  │ 朋友的社区        │  │                 │  │
 │     └──────┬───────┘  └────────┬─────────┘  └───────┬─────────┘  │
 └────────────┼───────────────────┼────────────────────┼────────────┘
              │                   │                    │
              ▼                   ▼                    ▼
 ┌────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
 │  Path A:           │ │  Path B:         │ │  Path C:             │
 │  Directory → Search│ │  Open invite     │ │  Scan QR / tap link  │
 │  → Pick community  │ │  link → Connect  │ │  扫描 QR / 点击链接   │
 │  → Upload prekeys  │ │  to community    │ │  → Exchange keys     │
 │  → Start chatting  │ │  server → Upload │ │    交换密钥            │
 │                    │ │  prekeys → Chat  │ │  → STUN/WebRTC       │
 │  目录 → 搜索       │ │                  │ │  → P2P chat          │
 │  → 选择社区        │ │  打开邀请链接     │ │    P2P 聊天           │
 │  → 上传预密钥      │ │  → 连接社区服务器 │ │                      │
 │  → 开始聊天        │ │  → 上传预密钥     │ │  (No server needed)  │
 │                    │ │  → 聊天           │ │  （无需服务器）        │
 └────────────────────┘ └──────────────────┘ └──────────────────────┘
```

**Note / 注意:** These paths are not mutually exclusive. A user can browse communities (Path A), join a friend's community (Path B), and have P2P chats (Path C) all at the same time. The client supports simultaneous connections to multiple communities and P2P peers.

**注意：** 这些路径不是互斥的。用户可以同时浏览社区（路径 A）、加入朋友的社区（路径 B）和进行 P2P 聊天（路径 C）。客户端支持同时连接多个社区和 P2P 对等端。

### 1.4 Cross-Mode Communication / 跨模式通讯

NexusLink's isolation-by-design means that cross-community communication requires explicit action. Here are the four most common scenarios:

NexusLink 的隔离设计意味着跨社区通讯需要明确的操作。以下是四种最常见的场景：

**Scenario 1 / 场景 1: Alice (Community A) wants to chat with Bob (Community B) / Alice（社区 A）想和 Bob（社区 B）聊天**

Communities are isolated by design. Community A's server cannot route messages to Community B's server. Solutions:

社区在设计上是隔离的。社区 A 的服务器无法将消息路由到社区 B 的服务器。解决方案：

| Solution / 解决方案 | How / 方式 |
|---|---|
| P2P Direct / P2P 直连 | Exchange keys out-of-band, connect via WebRTC / 带外交换密钥，通过 WebRTC 连接 |
| Join same community / 加入同一社区 | One of them joins the other's community / 其中一人加入对方的社区 |
| Official Plaza / 官方广场 | Both join the Directory Server's built-in community / 双方都加入目录服务器的内置社区 |

**Scenario 2 / 场景 2: Alice met Bob in a community, wants private chat / Alice 在社区中认识了 Bob，想私聊**

Two options:

两种选择：

1. **Via Community Server / 通过社区服务器** -- Use the community's 1:1 messaging. The server routes ciphertext but cannot read it. Metadata (who talks to whom, when) is visible to the community server. / 使用社区的一对一消息。服务器路由密文但无法读取。元数据（谁和谁聊、何时）对社区服务器可见。
2. **Switch to P2P / 切换到 P2P** -- Exchange keys within the community chat, then establish a direct WebRTC connection. Zero metadata leakage after switching. / 在社区聊天中交换密钥，然后建立直接 WebRTC 连接。切换后零元数据泄露。

**Scenario 3 / 场景 3: User migrates from Community A to Community B / 用户从社区 A 迁移到社区 B**

- UUID remains unchanged (it is derived from the user's key pair, not from any server). / UUID 保持不变（它是从用户的密钥对派生的，不是从任何服务器派生的）。
- Local chat history is preserved on the device (encrypted in SQLCipher). / 本地聊天记录保留在设备上（在 SQLCipher 中加密）。
- The user simply connects to Community B's server and uploads prekeys. No "account migration" needed. / 用户只需连接到社区 B 的服务器并上传预密钥。无需"账户迁移"。

**Scenario 4 / 场景 4: Community admin wants to read user chats / 社区管理员想查看用户聊天**

Impossible by design. All messages are end-to-end encrypted. The Community Server only ever handles ciphertext. Even with full server access, an admin cannot decrypt any message because they do not possess users' private keys.

在设计上不可能。所有消息都是端到端加密的。社区服务器只处理密文。即使拥有完全的服务器访问权限，管理员也无法解密任何消息，因为他们不拥有用户的私钥。

### 1.5 Data Visibility Matrix / 数据流向矩阵

This matrix shows exactly what each role in the system can and cannot see.

此矩阵精确展示系统中每个角色能看到和不能看到的内容。

| Data Item / 数据项 | Client / 客户端 | Community Server / 社区服务器 | Directory Server / 目录服务器 | P2P Peer / P2P 对等端 |
|---|:---:|:---:|:---:|:---:|
| Message plaintext / 消息明文 | ✓ | ✗ | ✗ | ✓ |
| Message ciphertext / 消息密文 | ✓ | ✓ | ✗ | ✗ |
| Sender UUID / 发送者 UUID | ✓ | ✓ | ✗ | ✓ |
| Receiver UUID / 接收者 UUID | ✓ | ✓ | ✗ | ✓ |
| Message timestamp / 消息时间戳 | ✓ | ✓ | ✗ | ✓ |
| Online status / 在线状态 | ✓ | ✓ | ✗ | ✗ |
| Community member list / 社区成员列表 | ✓ | ✓ | ✗ | ✗ |
| User IP address / 用户 IP 地址 | ✗ | ✓ | ✓ (when accessing) | ✓ |
| KYC status / KYC 状态 | ✓ | ✗ | ✓ | ✗ |
| Community name & address / 社区名称与地址 | ✓ | ✓ (own only) | ✓ | ✗ |
| Which communities joined / 加入了哪些社区 | ✓ | ✓ (own members only) | ✗ | ✗ |
| Private key / 私钥 | ✓ | ✗ | ✗ | ✗ |

**Reading the matrix / 阅读矩阵:**

- ✓ = This role has access to this data / 该角色可以访问此数据
- ✗ = This role cannot access this data by design / 该角色在设计上无法访问此数据
- The Client sees everything about its own data because it is the origin of all encryption/decryption. / 客户端能看到关于自身数据的一切，因为它是所有加密/解密的起点。
- The Private key row shows that only the Client ever holds the private key -- this is the foundation of the entire security model. / 私钥行显示只有客户端持有私钥——这是整个安全模型的基础。

---

## 2. Identity System / 身份系统

### 2.0 Design Philosophy: Choice over Mandate / 设计理念：选择而非强制

NexusLink's entire design is built on giving users choice. This extends to identity security -- users should choose their own security level, not be forced into one.

NexusLink 的整个设计都建立在给用户选择权之上。这也延伸到身份安全——用户应该自己选择安全级别，而不是被强制。

```
Identity Security Tiers / 身份安全分级：

  ┌─────────────────────────────────────────────────────────────────┐
  │                   User chooses at first launch                  │
  │                   用户在首次启动时选择                             │
  └───────────────────────────┬─────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
  ┌───────────────┐  ┌────────────────┐  ┌────────────────────┐
  │  Easy Mode    │  │ Standard Mode  │  │ Hardware Mode       │
  │  便捷模式      │  │ 标准模式        │  │ 硬件模式             │
  │               │  │                │  │                     │
  │  Software key │  │ OS secure      │  │ External hardware   │
  │  generated    │  │ storage auto-  │  │ key (YubiKey etc.)  │
  │  in app       │  │ detected &     │  │ or dedicated secure │
  │               │  │ used if avail. │  │ element             │
  │  软件密钥      │  │                │  │                     │
  │  App 内生成    │  │ 自动检测并使用  │  │ 外部硬件密钥         │
  │               │  │ OS 安全存储     │  │ 或专用安全芯片        │
  │  Door: open   │  │ Door: locked   │  │ Door: vault         │
  │  门：开着的    │  │ 门：锁着的      │  │ 门：保险库           │
  └───────────────┘  └────────────────┘  └────────────────────┘
        │                   │                    │
        │                   │                    │
        ▼                   ▼                    ▼
  Same UUID format    Same UUID format     Same UUID format
  Same E2E protocol   Same E2E protocol    Same E2E protocol
  Same user experience Same user experience Same user experience
  相同 UUID 格式      相同 UUID 格式        相同 UUID 格式
  相同 E2E 协议       相同 E2E 协议         相同 E2E 协议
  相同用户体验        相同用户体验           相同用户体验
```

All three tiers produce the same UUID, use the same encryption protocol, and interoperate seamlessly. A Hardware Mode user chatting with an Easy Mode user sees no difference -- the security level only affects how the *user's own* private key is protected.

三种级别生成相同的 UUID，使用相同的加密协议，完全无缝互操作。硬件模式用户和便捷模式用户聊天没有任何区别——安全级别只影响用户*自己的*私钥如何受保护。

### 2.1 Identity Security Tiers / 身份安全分级详述

```
Tier 1: Easy Mode / 便捷模式（默认）
──────────────────────────────────────
  Target: Casual users, first-time explorers
  目标用户：随便看看、初次体验的用户

  Key generation: Ed25519 key pair generated in software (ring / ed25519-dalek)
  密钥生成：Ed25519 密钥对在软件中生成

  Key storage: OS keychain (encrypted by OS credentials)
  密钥存储：OS 密钥链（受 OS 凭证加密保护）
    iOS      → Keychain Services (kSecAttrAccessibleWhenUnlockedThisDeviceOnly)
    Android  → Android Keystore (software-backed)
    macOS    → Keychain
    Windows  → DPAPI (Data Protection API)
    Linux    → Secret Service API / libsecret

  Mnemonic: Optional (user can skip and set up later)
  助记词：可选（用户可以跳过，以后再设置）

  Trade-off: Key is protected by OS login, but extractable by root/admin access.
             Suitable for everyday use, like how most people use messaging apps today.
  权衡：密钥受 OS 登录保护，但 root/admin 权限可提取。
        适合日常使用，类似今天大部分人使用即时通讯应用的方式。


Tier 2: Standard Mode / 标准模式（推荐）
──────────────────────────────────────────
  Target: Privacy-conscious users, community server operators
  目标用户：重视隐私的用户、社区服务器运营者

  Key generation: Ed25519 key pair, with hardware-backed protection if available
  密钥生成：Ed25519 密钥对，如果设备支持则使用硬件保护

  Key storage: Best available hardware on the device (auto-detected)
  密钥存储：设备上可用的最佳硬件（自动检测）
    iOS      → Secure Enclave (P-256 wrapping key protects Ed25519 seed)
    Android  → StrongBox if available, else TEE-backed Keystore
    macOS    → Secure Enclave (Apple Silicon / T2)
    Windows  → TPM 2.0 if available, else DPAPI
    Linux    → TPM 2.0 if available, else encrypted keyfile

  Implementation detail / 实现细节：
    Since most secure hardware natively supports P-256 but not Ed25519:
    由于大部分安全硬件原生支持 P-256 但不支持 Ed25519：

    1. Generate a P-256 "Device Wrapping Key" (DWK) inside secure hardware
       在安全硬件中生成 P-256 "设备包装密钥" (DWK)
    2. Generate Ed25519 Identity Key in software
       在软件中生成 Ed25519 身份密钥
    3. Encrypt Ed25519 private key with DWK (AES-256-GCM)
       用 DWK 加密 Ed25519 私钥（AES-256-GCM）
    4. Store encrypted blob; signing operations decrypt-in-memory, use, then zeroize
       存储加密后的密钥；签名操作时在内存中解密、使用、然后清零

  Mnemonic: Required (must write down before proceeding)
  助记词：必须（继续之前必须抄写）

  Trade-off: Private key never stored in plaintext. Hardware extraction requires
             physical attack on secure element. Slight latency for key operations.
  权衡：私钥永远不以明文存储。提取密钥需要对安全芯片进行物理攻击。
        密钥操作有轻微延迟。


Tier 3: Hardware Mode / 硬件模式（高级）
──────────────────────────────────────────
  Target: Journalists, activists, security professionals, high-value targets
  目标用户：记者、活动人士、安全专业人员、高价值目标

  Key generation: On external hardware device
  密钥生成：在外部硬件设备上

  Supported hardware / 支持的硬件：
    - FIDO2 security keys (YubiKey 5, SoloKeys, Nitrokey)
      FIDO2 安全密钥
    - Smart cards (OpenPGP card, PIV)
      智能卡
    - Custom HSM via PKCS#11
      通过 PKCS#11 的自定义 HSM

  Key storage: Key NEVER leaves the hardware device
  密钥存储：密钥永远不离开硬件设备

  Signing: All signing operations happen on the hardware device
  签名：所有签名操作在硬件设备上完成

  Mnemonic: Hardware-specific backup method (varies by device)
  助记词：硬件特定的备份方式（因设备而异）

  Algorithm note / 算法说明：
    - YubiKey 5+ supports Ed25519 natively
      YubiKey 5+ 原生支持 Ed25519
    - Older FIDO2 keys use P-256; NexusLink supports both curves for identity
      旧版 FIDO2 密钥使用 P-256；NexusLink 两种曲线都支持作为身份密钥
    - When P-256 is used, UUID derivation uses the same SHA-256(pubkey) scheme
      使用 P-256 时，UUID 推导使用相同的 SHA-256(pubkey) 方案

  Trade-off: Highest security, but requires purchasing external hardware.
             Must have hardware present to sign messages (authentication-per-use).
             Not suitable for casual users.
  权衡：最高安全级别，但需要购买外部硬件。
        每次签名都需要硬件在场（按次认证）。
        不适合普通用户。
```

```
Tier Comparison Summary / 分级对比总结：

  Dimension / 维度         Easy / 便捷    Standard / 标准   Hardware / 硬件
  ─────────────────────────────────────────────────────────────────────────
  Setup time / 设置时间     < 10 sec       < 30 sec          1-5 min
  Key extractable?          By OS admin    Physical attack   Nearly impossible
  密钥可提取？               OS 管理员可    需物理攻击         几乎不可能
  Requires purchase?        No             No                Yes ($25-60+)
  需要购买设备？             否              否                是
  Mnemonic required?        Optional       Required          Hardware backup
  助记词是否必须？           可选            必须               硬件备份方式
  Can upgrade later?        → Standard     → Hardware         -
  之后可升级？               → 标准          → 硬件            -
  Suitable for / 适用       Casual chat    Daily driver      High-risk users
                            随便聊聊       日常使用            高风险用户
```

Users can upgrade their security tier at any time without changing their UUID. The process re-wraps the existing Ed25519 private key with the new protection method.

用户随时可以升级安全级别而不改变 UUID。升级过程会用新的保护方式重新包装现有的 Ed25519 私钥。

### 2.2 Key Hierarchy / 密钥层级

```
Identity Key (IK) — 身份密钥
│  Curve: Ed25519 (signing) + X25519 (key exchange)
│  Or P-256 when using Hardware Mode with older FIDO2 keys
│  Generated once, on the first device
│  Public key = User UUID (Base58 encoded, ~44 characters)
│  Example: "nxl:8Wj3KpR2vT5mN7xQ1cY4bF6hA9dL0sE3gU8iO2pJ5"
│  Protection level depends on user's chosen security tier
│  保护级别取决于用户选择的安全分级
│
├── Device Signing Key (DSK) — 设备签名密钥
│   │  One per device
│   │  Signed by IK to prove it belongs to this identity
│   │  Storage follows the same tier as IK:
│   │  存储方式与 IK 相同的分级：
│   │    Easy     → OS keychain
│   │    Standard → Best available secure hardware (auto-detected)
│   │    Hardware → External hardware device
│   │
│   └── Ephemeral Prekeys — 临时预密钥
│       One-time use keys for X3DH key agreement
│       Uploaded to community server or exchanged via P2P
│       Replenished periodically
│
├── Self-Signing Key (SSK) — 自签名密钥
│   Signs device keys to form a trust chain
│   Derived from Identity Key
│
└── Recovery Key — 恢复密钥
    Derived from mnemonic phrase (BIP-39 compatible)
    Used to reconstruct IK when all devices are lost
    NEVER stored on any server
    In Easy Mode: setup can be deferred (with warning)
    便捷模式下：可以延迟设置（带有警告提示）
```

### 2.3 First Launch Flow / 首次启动流程

```
┌─────────────────────────────────────────────────┐
│                  First Launch                    │
│                  首次启动                         │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  "Welcome to NexusLink│
            │   Choose your         │
            │   security level"     │
            │                       │
            │  "欢迎来到 NexusLink   │
            │   选择你的安全级别"     │
            │                       │
            │  [Easy]  便捷          │
            │  Just start chatting   │
            │  直接开始聊天           │
            │                       │
            │  [Standard] 标准(推荐) │
            │  Hardware-backed keys  │
            │  硬件保护密钥           │
            │                       │
            │  [Hardware] 硬件       │
            │  External security key │
            │  外部安全密钥           │
            │                       │
            │  (What's this? 了解更多)│
            └───────────┬───────────┘
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
     Easy Mode    Standard Mode   Hardware Mode
          │             │              │
          ▼             ▼              ▼
     Generate IK   Generate DWK    Connect hardware
     in software   in secure HW    device, generate
                   wrap IK          IK on device
     软件生成 IK    安全硬件生成     连接硬件设备
                   DWK 包装 IK      在设备上生成 IK
          │             │              │
          └─────────────┼──────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Generate Device      │
            │  Signing Key (DSK)    │
            │  生成设备签名密钥       │
            └───────────┬───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  IK signs DSK         │
            │  → Device Certificate │
            │  IK 签名 DSK → 设备证书│
            └───────────┬───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Mnemonic phrase      │
            │  助记词恢复短语        │
            │                       │
            │  Easy: "Set up later" │
            │  便捷："以后再设置"     │
            │  (skip allowed)       │
            │                       │
            │  Standard: Required   │
            │  标准：必须设置         │
            │  (must write down)    │
            │                       │
            │  Hardware: Device-    │
            │  specific backup      │
            │  硬件：设备特定备份     │
            └───────────┬───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Show connection      │
            │  mode selection       │
            │  (offline, no network)│
            │  显示连接模式选择       │
            │  （离线，不联网）       │
            └───────────────────────┘

Note: "Easy" users who skip mnemonic will see periodic gentle
reminders until they set it up. Identity cannot be recovered
without a mnemonic or hardware backup.
注意：跳过助记词的"便捷"用户会看到定期温和提醒，
直到他们设置为止。没有助记词或硬件备份，身份无法恢复。
```

### 2.3 UUID Format / UUID 格式

```
Prefix:  "nxl:"          (identifies NexusLink addresses)
Body:    Base58(SHA-256(Ed25519_PublicKey)[0..32])
Length:  ~48 characters total

Example: nxl:8Wj3KpR2vT5mN7xQ1cY4bF6hA9dL0sE3gU8iO2pJ5

Display: Can be represented as QR code for easy sharing
         可通过二维码展示便于分享
```

### 2.4 Identity Recovery / 身份恢复

```
User loses all devices / 用户丢失所有设备
        │
        ▼
Enter mnemonic phrase on new device
在新设备输入助记词
        │
        ▼
Derive Identity Key from mnemonic (BIP-39 → Ed25519)
从助记词推导身份密钥
        │
        ▼
Generate new Device Signing Key on new hardware
在新硬件生成新设备签名密钥
        │
        ▼
IK signs new DSK → new Device Certificate
IK 签名新 DSK → 新设备证书
        │
        ▼
Publish revocation for old device keys
发布旧设备密钥的撤销声明
        │
        ▼
Re-upload prekeys to servers
重新上传预密钥到服务器
        │
        ▼
Identity restored. Old encrypted messages NOT recoverable
身份已恢复。旧的加密消息无法恢复（除非有加密备份）
```

---

## 3. Encryption Protocol / 加密协议

### 3.1 1:1 Messages / 一对一消息

Based on Signal Protocol (X3DH + Double Ratchet):

基于 Signal 协议（X3DH + Double Ratchet）：

```
Alice wants to message Bob (first time)
Alice 想给 Bob 发消息（首次）

Step 1: X3DH Key Agreement / X3DH 密钥协商
─────────────────────────────────────────
Alice fetches from Bob's server (or P2P exchange):
Alice 从 Bob 的服务器获取（或 P2P 交换）：
  - Bob's Identity Key (IK_B)
  - Bob's Signed Prekey (SPK_B)
  - Bob's One-Time Prekey (OPK_B)  [if available]

Alice computes:
  DH1 = DH(IK_A,  SPK_B)
  DH2 = DH(EK_A,  IK_B)     // EK_A = Alice's ephemeral key
  DH3 = DH(EK_A,  SPK_B)
  DH4 = DH(EK_A,  OPK_B)    // if OPK available

  SK = KDF(DH1 || DH2 || DH3 || DH4)
  → This shared secret initializes the Double Ratchet

Step 2: Double Ratchet / 双棘轮
─────────────────────────────────
Every message uses a unique key derived from:
每条消息使用从以下推导的唯一密钥：
  - DH ratchet (new key agreement per round-trip)
    DH 棘轮（每次往返产生新密钥协商）
  - Symmetric ratchet (new key per message)
    对称棘轮（每条消息产生新密钥）

Result: Forward secrecy + post-compromise recovery
结果：前向保密 + 入侵后恢复
```

### 3.2 Group Messages / 群组消息

Based on Sender Keys (similar to Signal Groups / Matrix Megolm):

基于发送者密钥（类似 Signal 群组 / Matrix Megolm）：

```
Group with N members / N 人群组

Each sender maintains ONE ratchet for the group
每个发送者维护一个群组棘轮

Alice sends to group:
Alice 发送到群组：
  1. Encrypt message with her Sender Key (symmetric ratchet)
     用她的发送者密钥加密消息（对称棘轮）
  2. Message sent ONCE to server (not N times)
     消息发送一次到服务器（不是 N 次）
  3. Each member decrypts with Alice's Sender Key
     每个成员用 Alice 的发送者密钥解密

Sender Key distribution:
发送者密钥分发：
  - Each member's Sender Key is encrypted with pairwise
    Double Ratchet sessions and sent to each other member
    每个成员的发送者密钥通过双棘轮会话加密后发送给其他成员

When a member leaves → all Sender Keys are rotated
当成员离开 → 所有发送者密钥轮换
```

---

## 4. Multi-Device Sync / 多设备同步

### 4.1 Adding a New Device / 添加新设备

```
Existing Device (Phone)              New Device (Laptop)
已有设备（手机）                       新设备（笔记本）

        │                                    │
        │  1. Generate DSK in secure HW      │
        │     在安全硬件中生成 DSK             │
        │                                    │
        │  2. Display pairing QR code ◄──────┤
        │     (contains new device's          │
        │      public key + challenge)        │
        │     显示配对二维码                    │
        │                                    │
        ├──► 3. Scan QR code                 │
        │     扫描二维码                       │
        │                                    │
        ├──► 4. Verify on both screens       │
        │     两个屏幕上确认验证               │
        │                                    │
        ├──► 5. IK signs new DSK             │
        │     IK 签名新 DSK                   │
        │                                    │
        ├──► 6. Encrypted key sync:          │
        │     加密密钥同步：                    │
        │     - Active session keys           │
        │     - Sender Keys for groups        │
        │     - Contact list (encrypted)      │
        │     - 活跃会话密钥                   │
        │     - 群组发送者密钥                  │
        │     - 联系人列表（加密）              │
        │                                    │
        │  7. New device is now authorized    │
        │     新设备已授权                     │
        ▼                                    ▼
```

### 4.2 Message Sync / 消息同步

```
When a message arrives for UUID-Alice:
当消息到达 UUID-Alice 时：

Server-based (Mode A/B):
基于服务器（模式 A/B）：
  Server stores encrypted message
  服务器存储加密消息
       │
       ├──► Push to Phone   (Device Key A encrypts)
       ├──► Push to Laptop  (Device Key B encrypts)
       └──► Push to Tablet  (Device Key C encrypts)

  Sender Keys (group) are shared across devices
  发送者密钥（群组）在设备间共享

  1:1 sessions: each device has its own Double Ratchet session
  一对一会话：每个设备有自己的双棘轮会话

P2P (Mode C):
  Sender encrypts for each known device of recipient
  发送者为接收者的每个已知设备加密
  Device discovery via DHT or relay
  通过 DHT 或中继发现设备
```

---

## 5. Network Topology / 网络拓扑

```
                    ┌─────────────────────┐
                    │  Official Directory │
                    │  Server             │
                    │  官方目录服务器       │
                    │                     │
                    │  Only stores:       │
                    │  仅存储：            │
                    │  - Community index  │
                    │  - KYC records      │
                    │  - Abuse reports    │
                    └──────┬──────────────┘
                           │
              ┌────────────┼─────────────┐
              │ register   │ register    │ register
              │ 注册       │ 注册        │ 注册
              ▼            ▼             ▼
     ┌──────────────┐ ┌──────────┐ ┌──────────────┐
     │ Community A  │ │ Comm. B  │ │ Community C  │
     │ 社区 A       │ │ 社区 B   │ │ 社区 C        │
     │              │ │          │ │  (unregistered│
     │ ◄──clients──►│ │◄─clients│ │   未注册)     │
     └──────────────┘ └──────────┘ └──────────────┘
           ▲                              ▲
           │          P2P direct          │
           │          P2P 直连            │
    ┌──────┴──────┐              ┌───────┴──────┐
    │   Client X  │◄────────────►│  Client Y    │
    │   客户端 X   │   (libp2p)  │  客户端 Y     │
    └─────────────┘              └──────────────┘

Key principle: Arrows show data flow direction.
关键原则：箭头显示数据流方向。
The directory server NEVER receives message data or membership info.
目录服务器永远不接收消息数据或成员信息。
```

---

## 6. Directory Server / 目录服务器

### 6.1 API Surface / API 接口

```
POST   /api/v1/community/register    - Register community (KYC required)
                                       注册社区（需实名认证）
GET    /api/v1/community/search      - Search communities
                                       搜索社区
GET    /api/v1/community/{id}        - Get community details
                                       获取社区详情
DELETE /api/v1/community/{id}        - Deregister community
                                       注销社区

POST   /api/v1/kyc/initiate         - Start KYC verification
                                       开始实名认证
POST   /api/v1/kyc/verify           - Submit KYC documents
                                       提交认证文件
GET    /api/v1/kyc/status            - Check KYC status
                                       查询认证状态

POST   /api/v1/report/abuse         - Report abuse
                                       举报滥用
```

### 6.2 Data Schema / 数据结构

```sql
-- Community registry / 社区注册表
communities (
    id              UUID PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    tags            VARCHAR[] ,
    endpoint_url    VARCHAR(500) NOT NULL,   -- community server address
    owner_uuid      VARCHAR(60) NOT NULL,    -- NexusLink UUID of owner
    public_key      BYTEA NOT NULL,          -- community server's signing key
    created_at      TIMESTAMPTZ,
    verified        BOOLEAN DEFAULT FALSE,
    member_count    INTEGER                  -- approximate, self-reported
    -- NOTE: NO member list, NO messages, NO user activity
    -- 注意：没有成员列表、没有消息、没有用户活动数据
)

-- KYC records / 实名认证记录
kyc_records (
    uuid            VARCHAR(60) PRIMARY KEY, -- NexusLink UUID
    status          ENUM('pending','approved','rejected'),
    verified_at     TIMESTAMPTZ,
    -- Actual identity documents stored encrypted,
    -- accessible only by compliance team
    -- 实际身份文件加密存储，仅合规团队可访问
)
```

### 6.3 Middleware Stack / 中间件管道

The directory server uses a tower-based middleware pipeline. Each incoming HTTP request passes through the following layers in order:

目录服务器使用基于 tower 的中间件管道。每个传入的 HTTP 请求按以下顺序通过各层：

```
Incoming Request / 传入请求
        │
        ▼
┌──────────────────────────────────────┐
│  1. Request Logging (tracing)        │
│     请求日志（tracing crate）         │
│     - Method, path, remote IP        │
│     - Request ID (UUID v7)           │
│     - Duration tracked on response   │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  2. CORS (tower-http CorsLayer)      │
│     跨域资源共享                      │
│     - Allowed origins: configurable  │
│     - Methods: GET, POST, DELETE     │
│     - Max age: 3600s                 │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  3. Rate Limiting                    │
│     速率限制                          │
│     - Per-IP sliding window          │
│     - Limits: 60 req/min general,    │
│       5 req/min for KYC endpoints    │
│     - Returns 429 with Retry-After   │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  4. Authentication                   │
│     认证                              │
│     - Bearer token (Ed25519 signed   │
│       challenge-response)            │
│     - Token contains: UUID, device   │
│       ID, expiry, signature          │
│     - Public endpoints bypass auth   │
│       (search, community details)    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  5. Compression                      │
│     响应压缩                          │
│     - gzip / brotli (Accept-Encoding)│
│     - Minimum body size: 256 bytes   │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  6. Router (axum::Router)            │
│     路由分发                          │
│     - /api/v1/community/*            │
│     - /api/v1/kyc/*                  │
│     - /api/v1/report/*               │
│     - /health                        │
└──────────────────────────────────────┘
```

Rust implementation sketch / Rust 实现概要：

```rust
use axum::Router;
use tower::ServiceBuilder;
use tower_http::{cors::CorsLayer, compression::CompressionLayer, trace::TraceLayer};

fn build_middleware_stack(app: Router, config: &AppConfig) -> Router {
    app.layer(
        ServiceBuilder::new()
            .layer(TraceLayer::new_for_http())        // 1. logging
            .layer(build_cors_layer(config))           // 2. CORS
            .layer(build_rate_limit_layer(config))     // 3. rate limiting
            .layer(AuthLayer::new(config.auth.clone()))// 4. authentication
            .layer(CompressionLayer::new())            // 5. compression
    )
}
```

### 6.4 Database Connection Pool / 数据库连接池管理

The directory server manages PostgreSQL connections through SQLx with PgPool. Connection pooling is essential for handling concurrent API requests efficiently.

目录服务器通过 SQLx 的 PgPool 管理 PostgreSQL 连接。连接池对于高效处理并发 API 请求至关重要。

```rust
use sqlx::postgres::{PgPoolOptions, PgPool};

async fn init_db_pool(config: &DatabaseConfig) -> PgPool {
    PgPoolOptions::new()
        .max_connections(config.max_connections)  // default: 20
        .min_connections(config.min_connections)  // default: 5
        .acquire_timeout(Duration::from_secs(5))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(&config.database_url)
        .await
        .expect("Failed to create database pool")
}
```

```
Connection Pool Parameters / 连接池参数：

  max_connections:   20 (default, tunable)
                     最大连接数：20（默认，可调整）
  min_connections:    5 (keeps warm connections ready)
                     最小连接数：5（保持热连接就绪）
  acquire_timeout:    5s (fail fast if pool exhausted)
                     获取超时：5秒（池耗尽时快速失败）
  idle_timeout:     600s (reclaim unused connections)
                     空闲超时：600秒（回收未使用连接）
  max_lifetime:    1800s (prevent stale connections)
                     最大存活：1800秒（防止过期连接）

Health check query / 健康检查查询：
  SELECT 1   (executed periodically on idle connections)
             （定期在空闲连接上执行）
```

### 6.5 Directory Search Implementation / 目录搜索实现

Community search relies on PostgreSQL full-text search (tsvector/tsquery) combined with tag filtering and cursor-based pagination.

社区搜索依赖 PostgreSQL 全文检索（tsvector/tsquery），并结合标签过滤与游标分页。

```sql
-- Full-text search index / 全文检索索引
ALTER TABLE communities ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(description, '')), 'B')
    ) STORED;

CREATE INDEX idx_communities_search ON communities USING GIN (search_vector);

-- Tag index (GIN for array containment queries)
-- 标签索引（GIN 用于数组包含查询）
CREATE INDEX idx_communities_tags ON communities USING GIN (tags);
```

```
Search Query Flow / 搜索查询流程：

  Client request / 客户端请求：
    GET /api/v1/community/search?q=gaming&tags=fps,open&limit=20&cursor=xxx

  Server executes / 服务端执行：
    1. Parse query text → tsquery ('gaming')
       解析查询文本 → tsquery
    2. Parse tags → array containment filter (tags @> '{fps,open}')
       解析标签 → 数组包含过滤
    3. Combine with ts_rank for relevance scoring
       结合 ts_rank 进行相关性排序
    4. Apply cursor-based pagination (WHERE created_at < cursor_timestamp)
       应用游标分页
    5. Return results with next_cursor token
       返回结果及 next_cursor 令牌
```

```sql
-- Example search query / 搜索查询示例
SELECT id, name, description, tags, member_count,
       ts_rank(search_vector, query) AS relevance
FROM communities, plainto_tsquery('english', $1) AS query
WHERE search_vector @@ query
  AND ($2::varchar[] IS NULL OR tags @> $2)
  AND verified = true
  AND ($3::timestamptz IS NULL OR created_at < $3)  -- cursor
ORDER BY relevance DESC, created_at DESC
LIMIT $4;
```

### 6.6 KYC Complete Flow / KYC 完整流程

The Know Your Customer (KYC) process is required for community registration. It ensures accountability while minimizing data exposure.

实名认证（KYC）是社区注册的前提。它在确保责任追溯的同时尽量减少数据暴露。

```
Client                     Directory Server               Compliance System
客户端                      目录服务器                       合规系统

  │                              │                              │
  │  1. POST /kyc/initiate       │                              │
  │  Body: { uuid, device_sig }  │                              │
  │─────────────────────────────►│                              │
  │                              │                              │
  │  2. Return challenge nonce   │                              │
  │     + required document list │                              │
  │◄─────────────────────────────│                              │
  │                              │                              │
  │  3. POST /kyc/verify         │                              │
  │  Body: {                     │                              │
  │    uuid,                     │                              │
  │    signed_challenge,         │                              │
  │    encrypted_documents: [    │                              │
  │      { type: "id_card",      │                              │
  │        data: <encrypted> },  │                              │
  │      { type: "selfie",       │                              │
  │        data: <encrypted> }   │                              │
  │    ]                         │                              │
  │  }                           │                              │
  │─────────────────────────────►│                              │
  │                              │  4. Verify signature         │
  │                              │     验证签名                  │
  │                              │                              │
  │                              │  5. Store encrypted docs     │
  │                              │     存储加密文件              │
  │                              │                              │
  │                              │  6. Forward to compliance    │
  │                              │     转发至合规系统            │
  │                              │─────────────────────────────►│
  │                              │                              │
  │  7. Status: "pending"        │                              │
  │◄─────────────────────────────│                              │
  │                              │                              │
  │                              │  8. Compliance reviews docs  │
  │                              │     合规团队审核文件          │
  │                              │                              │
  │                              │  9. Decision: approved /     │
  │                              │     rejected (with reason)   │
  │                              │     决定：通过/拒绝（附原因）  │
  │                              │◄─────────────────────────────│
  │                              │                              │
  │                              │  10. Update kyc_records      │
  │                              │      status = 'approved'     │
  │                              │      更新认证记录状态         │
  │                              │                              │
  │  11. GET /kyc/status         │                              │
  │─────────────────────────────►│                              │
  │                              │                              │
  │  12. Response: "approved"    │                              │
  │      (can now register       │                              │
  │       communities)           │                              │
  │      响应："已通过"            │                              │
  │      （现在可以注册社区）      │                              │
  │◄─────────────────────────────│                              │
  │                              │                              │

Document retention policy / 文件保留策略：
  - Encrypted documents deleted after 90 days post-approval
    加密文件在审核通过后 90 天删除
  - Only KYC status (approved/rejected) retained permanently
    仅永久保留认证状态（通过/拒绝）
  - Compliance team uses separate encryption keys
    合规团队使用独立加密密钥
```

### 6.7 Anti-Abuse Mechanisms / 反滥用机制

The directory server employs multiple layers of protection against automated abuse, spam registrations, and denial-of-service attacks.

目录服务器采用多层防护机制，抵御自动化滥用、垃圾注册和拒绝服务攻击。

```
Anti-Abuse Stack / 反滥用体系：

1. IP-Based Rate Limiting / 基于 IP 的速率限制
   ─────────────────────────────────────────────
   - Sliding window algorithm (token bucket per IP)
     滑动窗口算法（每 IP 令牌桶）
   - General API:     60 requests / minute
     通用 API：       60 请求/分钟
   - KYC endpoints:    5 requests / minute
     KYC 接口：        5 请求/分钟
   - Search:          30 requests / minute
     搜索：           30 请求/分钟
   - Exceeded → HTTP 429 + Retry-After header
     超限 → HTTP 429 + Retry-After 头

2. Proof-of-Work Registration / 工作量证明注册
   ─────────────────────────────────────────────
   Before registering a community, the client must solve a
   computational puzzle (Hashcash-style):
   注册社区前，客户端必须求解计算谜题（Hashcash 风格）：

   - Server issues:  challenge = SHA-256(random_nonce || timestamp)
     服务端发放：    challenge = SHA-256(随机数 || 时间戳)
   - Client finds:   nonce where SHA-256(challenge || nonce) has
                     N leading zero bits (difficulty adjustable)
     客户端求解：    使 SHA-256(challenge || nonce) 前 N 位为零的 nonce
   - Typical difficulty: ~20 bits (< 1 second on modern hardware,
     expensive for botnets at scale)
     典型难度：~20 位（现代硬件不到 1 秒，大规模僵尸网络代价高昂）
   - Difficulty auto-adjusts based on registration volume
     难度根据注册量自动调整

3. Anomaly Detection / 异常检测
   ─────────────────────────────
   - Registration spike detection (> 3x normal rate in 1 hour)
     注册高峰检测（1 小时内超过正常速率 3 倍）
   - Geographic clustering analysis (many registrations from
     same IP range)
     地理聚类分析（同一 IP 段大量注册）
   - Similarity detection for community names/descriptions
     社区名称/描述相似度检测
   - Action: increase PoW difficulty, flag for manual review
     措施：提高 PoW 难度，标记人工审核
```

### 6.8 Health Check and Monitoring / 健康检查与监控接口

The directory server exposes health check and monitoring endpoints for operational visibility.

目录服务器提供健康检查和监控端点，用于运维可观测性。

```
Health Endpoints / 健康检查端点：

GET /health
  - Returns HTTP 200 if the service is running
    服务运行中返回 HTTP 200
  - Response body:
    {
      "status": "healthy",
      "version": "1.2.3",
      "uptime_seconds": 86400
    }

GET /health/ready
  - Returns HTTP 200 only when ALL dependencies are ready:
    仅当所有依赖就绪时返回 HTTP 200：
    - Database connection pool has available connections
      数据库连接池有可用连接
    - Migrations are up to date
      数据库迁移已是最新
  - Used by load balancers to determine routing
    供负载均衡器判断路由

GET /metrics (Prometheus format)
  - nexuslink_directory_requests_total{method, path, status}
  - nexuslink_directory_request_duration_seconds{method, path}
  - nexuslink_directory_db_pool_connections{state="idle|active|waiting"}
  - nexuslink_directory_kyc_submissions_total{status}
  - nexuslink_directory_communities_registered_total
  - nexuslink_directory_search_queries_total
  - nexuslink_directory_rate_limit_rejections_total
```

---

## 7. Community Server / 社区服务器

### 7.1 Responsibilities / 职责

```
Community Server handles / 社区服务器负责：
  ✅ User prekey storage (encrypted bundles)
     用户预密钥存储（加密包）
  ✅ Message routing between members
     成员间消息路由
  ✅ Group/channel management
     群组/频道管理
  ✅ File/media storage (encrypted)
     文件/媒体存储（加密）
  ✅ Push notification relay
     推送通知中继
  ✅ Member management & moderation
     成员管理与审核
  ✅ Optional: register with directory
     可选：注册到目录服务器

Community Server CANNOT / 社区服务器不能：
  ❌ Read message content (E2E encrypted)
     阅读消息内容（端到端加密）
  ❌ Forge messages (signed by sender's device key)
     伪造消息（由发送者设备密钥签名）
  ❌ Access private keys (stored in device hardware)
     访问私钥（存储在设备硬件中）

Community Server CAN see (trade-off) / 社区服务器能看到（权衡）：
  ⚠️ Metadata: who messages whom, when, frequency
     元数据：谁给谁发消息、时间、频率
  ⚠️ IP addresses of connected clients
     已连接客户端的 IP 地址
  ⚠️ Encrypted message sizes
     加密消息的大小
```

### 7.2 One-Click Deployment / 一键部署

```yaml
# docker-compose.yml for community server
# 社区服务器的 docker-compose.yml
version: '3.8'
services:
  nexuslink-community:
    image: nexuslink/community-server:latest
    ports:
      - "443:443"
    volumes:
      - ./data:/data
      - ./certs:/certs
    environment:
      - NEXUSLINK_DOMAIN=your.community.com
      - NEXUSLINK_ADMIN_UUID=nxl:your_uuid_here
      - NEXUSLINK_MAX_MEMBERS=10000
      - NEXUSLINK_STORAGE_LIMIT=50GB
      # Optional: register with official directory
      # 可选：注册到官方目录
      - NEXUSLINK_DIRECTORY_URL=https://directory.nexuslink.org
      - NEXUSLINK_DIRECTORY_TOKEN=your_registration_token
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nexuslink
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
```

### 7.3 WebSocket Connection Management / WebSocket 连接管理架构

The community server maintains persistent WebSocket connections with all online clients. Connection management is critical for real-time message delivery and presence tracking.

社区服务器与所有在线客户端维护持久的 WebSocket 连接。连接管理对实时消息投递和在线状态追踪至关重要。

```
WebSocket Architecture / WebSocket 架构：

┌─────────────────────────────────────────────────────────────────┐
│                    Connection Manager                            │
│                    连接管理器                                     │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Connection Registry (DashMap<UUID, Vec<DeviceConn>>)    │   │
│  │  连接注册表                                               │   │
│  │                                                          │   │
│  │  UUID-Alice ──► [Device-Phone(ws1), Device-Laptop(ws2)] │   │
│  │  UUID-Bob   ──► [Device-Phone(ws3)]                     │   │
│  │  UUID-Carol ──► [Device-Tablet(ws4)]                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ Heartbeat Mgr   │  │ Reconnect Tracker│  │ Backpressure  │  │
│  │ 心跳管理器       │  │ 重连追踪器        │  │ 背压控制       │  │
│  └─────────────────┘  └──────────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────┘

Heartbeat Protocol / 心跳协议：
  - Server sends PING every 30 seconds
    服务端每 30 秒发送 PING
  - Client must respond with PONG within 10 seconds
    客户端必须在 10 秒内回复 PONG
  - 2 missed PONGs → connection considered dead, resources released
    连续 2 次未收到 PONG → 连接视为断开，释放资源

Client Reconnection Strategy / 客户端断线重连策略：
  - Exponential backoff: 1s, 2s, 4s, 8s, 16s, 30s (max)
    指数退避：1秒、2秒、4秒、8秒、16秒、30秒（最大）
  - Jitter: random +-25% to prevent thundering herd
    抖动：随机 +-25% 防止惊群效应
  - On reconnect: client sends last_received_message_id
    重连时：客户端发送 last_received_message_id
  - Server replays missed messages from queue
    服务端从队列重放未送达消息

Connection Limits / 连接限制：
  - Max connections per UUID: 8 (devices)
    每 UUID 最大连接数：8（设备）
  - Max total connections per server: configurable (default: 50,000)
    每服务器最大总连接数：可配置（默认：50,000）
  - Idle timeout (no messages, heartbeat only): 24 hours
    空闲超时（无消息，仅心跳）：24 小时
```

### 7.4 Message Routing Engine / 消息路由引擎

The routing engine is responsible for delivering encrypted messages from senders to all intended recipients across multiple devices.

路由引擎负责将加密消息从发送者投递到所有目标接收者的所有设备。

```
Message Routing Flow / 消息路由流程：

Sender ──► Community Server ──► Recipients
发送者       社区服务器           接收者

Step 1: Ingest / 消息接收
─────────────────────────
  - Sender pushes Envelope via WebSocket
    发送者通过 WebSocket 推送 Envelope
  - Server validates: signature, sender membership, timestamp freshness
    服务端验证：签名、发送者成员资格、时间戳时效性

Step 2: Route / 路由判定
────────────────────────
  For 1:1 messages / 一对一消息：
    - Lookup recipient UUID in connection registry
      在连接注册表中查找接收者 UUID
    - Deliver to ALL online devices of recipient
      投递至接收者的所有在线设备

  For group/channel messages (fan-out) / 群组/频道消息（扇出）：
    - Lookup all member UUIDs of the channel
      查找频道的所有成员 UUID
    - For each member, deliver to all online devices
      对每个成员，投递至其所有在线设备
    - Message stored ONCE, reference shared (copy-on-write semantics)
      消息存储一次，引用共享（写时复制语义）

Step 3: Multi-Device Delivery / 多设备投递
──────────────────────────────────────────
  Online devices → immediate WebSocket push
  在线设备 → 立即 WebSocket 推送
  Offline devices → enqueue to offline message queue
  离线设备 → 加入离线消息队列

Step 4: Offline Queue / 离线消息队列
──────────────────────────────────────
  - Per-device queue (not per-user) stored in PostgreSQL
    每设备队列（非每用户）存储在 PostgreSQL 中
  - Messages expire after configurable TTL (default: 30 days)
    消息在可配置 TTL 后过期（默认：30 天）
  - On reconnect: server drains queue in order
    重连时：服务端按顺序排空队列
  - Delivery receipt: client ACKs each message
    投递回执：客户端对每条消息发送 ACK
  - ACKed messages removed from queue
    已确认消息从队列移除

Fan-Out Optimization / 扇出优化：
  - Small groups (< 50 members): inline fan-out in handler task
    小群组（< 50 成员）：在处理任务中内联扇出
  - Large channels (>= 50 members): spawn dedicated fan-out task
    大频道（>= 50 成员）：创建专用扇出任务
  - Rate limit on group sends: 30 messages / 10 seconds per sender
    群组发送速率限制：每发送者 10 秒 30 条
```

### 7.5 Prekey Management Service / 预密钥管理服务

The community server stores and distributes prekey bundles needed for X3DH key agreement. This is a critical service for establishing new encrypted sessions.

社区服务器存储和分发 X3DH 密钥协商所需的预密钥包。这是建立新加密会话的关键服务。

```
Prekey Lifecycle / 预密钥生命周期：

┌───────────┐    Upload     ┌────────────────┐    Fetch     ┌───────────┐
│  Client   │──────────────►│   Prekey Store │◄────────────│  Peer     │
│  客户端    │               │   预密钥存储    │             │  对端      │
└───────────┘               └────────────────┘             └───────────┘

Storage Schema / 存储结构：
  prekey_bundles (
      uuid           VARCHAR(60),        -- owner UUID
      device_id      INTEGER,            -- device identifier
      signed_prekey  BYTEA NOT NULL,     -- Signed Prekey (rotated monthly)
      spk_signature  BYTEA NOT NULL,     -- IK signature over SPK
      one_time_keys  BYTEA[],           -- array of One-Time Prekeys
      uploaded_at    TIMESTAMPTZ,
      PRIMARY KEY (uuid, device_id)
  )

Distribution Rules / 分发规则：
  - Each fetch consumes ONE One-Time Prekey (OPK)
    每次获取消耗一个一次性预密钥（OPK）
  - OPK is atomically removed after fetch (SELECT + DELETE in transaction)
    OPK 在获取后原子性删除（事务中 SELECT + DELETE）
  - If no OPK available, return only Signed Prekey (degraded X3DH)
    如果无可用 OPK，仅返回签名预密钥（降级 X3DH）

Replenishment Notification / 补充通知：
  - Server monitors OPK count per device
    服务端监控每设备 OPK 数量
  - When count drops below threshold (default: 25):
    当数量低于阈值（默认：25）时：
    → Push notification to device: "replenish prekeys"
      向设备推送通知："补充预密钥"
    → Device generates and uploads new batch (default: 100 OPKs)
      设备生成并上传新批次（默认：100 个 OPK）

Expiration Cleanup / 过期清理：
  - Signed Prekey older than 60 days and superseded → deleted
    超过 60 天且已被替换的签名预密钥 → 删除
  - OPKs older than 90 days and unused → deleted
    超过 90 天且未使用的 OPK → 删除
  - Cleanup runs as periodic background task (every 6 hours)
    清理作为定期后台任务运行（每 6 小时）
```

### 7.6 Channel and Group Permission System / 频道与群组权限系统

The community server implements a Role-Based Access Control (RBAC) model for channels and groups. Each member is assigned a role that determines their permissions.

社区服务器为频道和群组实现了基于角色的访问控制（RBAC）模型。每个成员被分配一个角色，决定其权限。

```
Role Hierarchy / 角色层级：

  owner         (1 per community)   ── Full control, transfer ownership
  所有者         （每社区 1 个）         完全控制，可转让所有权

  admin          (unlimited)        ── Manage channels, roles, members
  管理员          （不限数量）          管理频道、角色、成员

  moderator      (unlimited)        ── Delete messages, mute/kick users
  版主            （不限数量）          删除消息、禁言/踢出用户

  member         (unlimited)        ── Send messages, read channels
  成员            （不限数量）          发送消息、阅读频道

  guest          (unlimited)        ── Read-only access to public channels
  访客            （不限数量）          公开频道只读访问

Permission Matrix / 权限矩阵：

  Action                  owner  admin  moderator  member  guest
  ──────────────────────  ─────  ─────  ─────────  ──────  ─────
  Create channel           Y      Y        N         N       N
  创建频道
  Delete channel           Y      Y        N         N       N
  删除频道
  Edit channel settings    Y      Y        N         N       N
  编辑频道设置
  Assign roles             Y      Y        N         N       N
  分配角色
  Kick member              Y      Y        Y         N       N
  踢出成员
  Mute member              Y      Y        Y         N       N
  禁言成员
  Delete others' messages  Y      Y        Y         N       N
  删除他人消息
  Send messages            Y      Y        Y         Y       N
  发送消息
  Read public channels     Y      Y        Y         Y       Y
  阅读公开频道
  Read private channels    Y      Y     (if member) (if member) N
  阅读私有频道                           （若为成员）（若为成员）
  Upload media             Y      Y        Y         Y       N
  上传媒体
  Transfer ownership       Y      N        N         N       N
  转让所有权

Channel Types / 频道类型：
  - public:   visible and readable by all members (including guests)
    公开：    所有成员可见可读（包括访客）
  - private:  visible only to explicitly added members
    私有：    仅对明确添加的成员可见
  - announce: only admins/moderators can send, all can read
    公告：    仅管理员/版主可发送，所有人可阅读
```

### 7.7 Media Upload Pipeline / 媒体上传处理管道

Media files (images, videos, documents) are uploaded through a multi-stage pipeline that handles encryption, storage, and optional post-processing.

媒体文件（图片、视频、文档）通过多阶段管道上传，处理加密、存储和可选的后处理。

```
Upload Pipeline / 上传管道：

Client                        Community Server                Storage
客户端                         社区服务器                        存储

  │                                │                              │
  │  1. Request upload slot        │                              │
  │  POST /api/v1/media/upload     │                              │
  │  { filename, size, mime_type } │                              │
  │───────────────────────────────►│                              │
  │                                │                              │
  │  2. Return upload_id +         │                              │
  │     chunk_size (4 MB)          │                              │
  │◄───────────────────────────────│                              │
  │                                │                              │
  │  3. Client-side encryption:    │                              │
  │     - Generate random AES-256  │                              │
  │       key for this file        │                              │
  │     - Encrypt file with        │                              │
  │       AES-256-GCM              │                              │
  │     - Compute SHA-256 digest   │                              │
  │     客户端加密：                │                              │
  │     - 生成随机 AES-256 密钥     │                              │
  │     - 用 AES-256-GCM 加密文件  │                              │
  │     - 计算 SHA-256 摘要        │                              │
  │                                │                              │
  │  4. Upload chunks              │                              │
  │  PUT /api/v1/media/upload      │                              │
  │      /{upload_id}/chunk/{n}    │                              │
  │  (repeat for each chunk)       │                              │
  │───────────────────────────────►│  5. Write to disk/object    │
  │                                │     storage (already         │
  │                                │     encrypted by client)     │
  │                                │─────────────────────────────►│
  │                                │                              │
  │  6. POST /api/v1/media/upload  │                              │
  │     /{upload_id}/complete      │                              │
  │  { digest: SHA-256 }           │                              │
  │───────────────────────────────►│                              │
  │                                │  7. Verify digest, assemble │
  │                                │     验证摘要，组装文件        │
  │                                │                              │
  │                                │  8. Generate thumbnail       │
  │                                │     (images only, encrypted) │
  │                                │     生成缩略图               │
  │                                │     （仅图片，加密存储）      │
  │                                │                              │
  │  9. Return media_url +         │                              │
  │     thumbnail_url              │                              │
  │◄───────────────────────────────│                              │
  │                                │                              │
  │  10. Include media_url in      │                              │
  │      encrypted message to      │                              │
  │      recipient, along with     │                              │
  │      AES key for decryption    │                              │
  │      在加密消息中包含 media_url │                              │
  │      及 AES 密钥供接收者解密    │                              │

Chunk Upload Details / 分片上传详情：
  - Chunk size: 4 MB (configurable)
    分片大小：4 MB（可配置）
  - Concurrent chunk uploads: up to 3 parallel
    并发分片上传：最多 3 个并行
  - Resume support: client can retry individual chunks
    续传支持：客户端可重试单个分片
  - Upload timeout: 1 hour (incomplete uploads auto-cleaned)
    上传超时：1 小时（未完成上传自动清理）

Thumbnail Generation / 缩略图生成：
  - Images: resize to 200x200, JPEG quality 60, then encrypt
    图片：缩放至 200x200，JPEG 质量 60，然后加密
  - Thumbnail uses SAME encryption key as original
    缩略图使用与原文件相同的加密密钥
  - Videos: no server-side thumbnail (server cannot decrypt)
    视频：无服务端缩略图（服务端无法解密）

CDN Integration (optional) / CDN 集成（可选）：
  - Encrypted blobs can be pushed to S3-compatible storage
    加密文件可推送到 S3 兼容存储
  - Signed URLs for download (time-limited, 1 hour)
    下载使用签名 URL（限时 1 小时）
  - CDN caches encrypted blobs; no privacy risk
    CDN 缓存加密文件；无隐私风险
```

### 7.8 Push Notification Architecture / 推送通知架构

Push notifications allow offline clients to be alerted about new messages without maintaining a persistent connection.

推送通知允许离线客户端在没有持久连接的情况下收到新消息提醒。

```
Push Notification Flow / 推送通知流程：

Community Server                 Push Gateway              Device
社区服务器                        推送网关                   设备

  │                                │                         │
  │  1. Message arrives for        │                         │
  │     offline device             │                         │
  │     消息到达，目标设备离线       │                         │
  │                                │                         │
  │  2. Construct push payload:    │                         │
  │     构造推送载荷：              │                         │
  │     {                          │                         │
  │       "type": "new_message",   │                         │
  │       "sender_uuid": "nxl:xx", │                         │
  │       "channel_id": "...",     │                         │
  │       // NO message content    │                         │
  │       // 不包含消息内容         │                         │
  │     }                          │                         │
  │                                │                         │
  │  3. Route to appropriate       │                         │
  │     push service:              │                         │
  │     路由至对应推送服务：         │                         │
  │─────────────────────────────►  │                         │
  │  Android → FCM (Firebase)      │  4. Deliver to device  │
  │  iOS     → APNs (Apple)        │─────────────────────── ►│
  │  Fallback → UnifiedPush        │     投递至设备           │
  │                                │                         │
  │                                │                         │
  │                                │         5. Device wakes │
  │                                │         app, connects   │
  │                                │         via WebSocket,  │
  │                                │         fetches messages│
  │                                │         设备唤醒应用，   │
  │                                │         建立 WebSocket， │
  │                                │◄────────fetch 消息      │
  │                                │                         │

Push Service Integration / 推送服务集成：

  Platform    Service        Token Storage
  平台        服务            令牌存储
  ────────    ─────────      ──────────────
  Android     FCM            push_tokens table
  iOS         APNs           push_tokens table
  Self-host   UnifiedPush    push_tokens table (distributor URL)
  Desktop     WebSocket      (always connected, no push needed)
  桌面端      WebSocket      （始终连接，无需推送）

Notification Queue / 通知队列：
  - Notifications batched per device (max 5 per batch)
    通知按设备批量处理（每批最多 5 条）
  - Coalescing: multiple messages from same channel → 1 notification
    合并：同一频道的多条消息 → 1 条通知
  - Retry on failure: 3 attempts with exponential backoff
    失败重试：3 次尝试，指数退避
  - Token invalidation: if push service reports invalid token,
    remove from push_tokens table
    令牌失效：如果推送服务报告令牌无效，从 push_tokens 表中移除

Silent Push / 静默推送：
  - Used for prekey replenishment notifications
    用于预密钥补充通知
  - Used for key rotation notifications
    用于密钥轮换通知
  - Does not display user-visible notification
    不显示用户可见通知
  - Triggers background fetch on the device
    触发设备后台获取
```

### 7.9 Server-Side Performance Optimization / 服务端性能优化

The community server employs several strategies to handle high concurrency and throughput efficiently.

社区服务器采用多种策略高效处理高并发和高吞吐场景。

```
Performance Optimization Stack / 性能优化体系：

1. Connection Multiplexing / 连接复用
   ─────────────────────────────────────
   - tokio async runtime with multi-threaded scheduler
     tokio 异步运行时，多线程调度器
   - Each WebSocket handled by a lightweight tokio task (not OS thread)
     每个 WebSocket 由轻量 tokio 任务处理（非 OS 线程）
   - Cost per connection: ~8 KB memory (stack + buffers)
     每连接开销：约 8 KB 内存（栈 + 缓冲区）
   - 50,000 concurrent connections ≈ 400 MB memory
     50,000 并发连接约占 400 MB 内存

2. Message Batching / 消息批量处理
   ────────────────────────────────
   - Incoming messages buffered and flushed to DB in batches
     传入消息缓冲后批量刷入数据库
   - Batch size: 50 messages or 100ms elapsed (whichever first)
     批次大小：50 条消息或 100 毫秒（以先到者为准）
   - Reduces PostgreSQL round-trips by ~10x under load
     高负载下减少约 10 倍 PostgreSQL 往返
   - Fan-out also batched: group sends coalesced before delivery
     扇出也做批量处理：群组发送合并后再投递

3. Database Query Optimization / 数据库查询优化
   ────────────────────────────────────────────
   - Prepared statements cached per connection (sqlx automatic)
     预处理语句按连接缓存（sqlx 自动）
   - Indexes on hot paths:
     热路径索引：
       - message_queue(recipient_uuid, device_id, created_at)
       - members(uuid) with INCLUDE (public_key, role)
       - channels(community_id, channel_type)
   - Partitioned message_queue by month (for efficient expiration)
     message_queue 按月分区（高效过期）
   - VACUUM scheduled during low-traffic hours
     VACUUM 安排在低流量时段

4. Caching Layer / 缓存层
   ──────────────────────
   - In-process LRU cache (moka crate):
     进程内 LRU 缓存（moka crate）：
       - Member roles and permissions: TTL 5 minutes
         成员角色和权限：TTL 5 分钟
       - Channel metadata: TTL 10 minutes
         频道元数据：TTL 10 分钟
       - Prekey bundle existence checks: TTL 1 minute
         预密钥包存在性检查：TTL 1 分钟
   - Optional Redis for multi-instance deployments (see Chapter 13)
     多实例部署可选 Redis（见第 13 章）
```

### 7.10 Operations and Monitoring / 运维与监控

Comprehensive observability is built into the community server to support production operations.

社区服务器内建全面的可观测性，支持生产环境运维。

```
Observability Stack / 可观测性体系：

1. Prometheus Metrics Export / Prometheus 指标导出
   ──────────────────────────────────────────────
   Endpoint: GET /metrics (Prometheus text format)

   Key Metrics / 关键指标：
   - nexuslink_ws_connections_active          (gauge)
     当前活跃 WebSocket 连接数
   - nexuslink_ws_connections_total           (counter)
     WebSocket 连接总数
   - nexuslink_messages_routed_total          (counter, labels: type)
     路由消息总数
   - nexuslink_messages_queued_offline        (gauge)
     离线消息队列深度
   - nexuslink_message_delivery_duration_seconds (histogram)
     消息投递延迟
   - nexuslink_prekey_pool_size               (gauge, labels: uuid)
     预密钥池大小
   - nexuslink_media_upload_bytes_total       (counter)
     媒体上传总字节数
   - nexuslink_push_notifications_sent_total  (counter, labels: platform, status)
     推送通知发送总数
   - nexuslink_db_pool_connections            (gauge, labels: state)
     数据库连接池状态
   - nexuslink_db_query_duration_seconds      (histogram, labels: query)
     数据库查询延迟

2. Structured Logging / 结构化日志
   ────────────────────────────────
   Uses the tracing crate ecosystem (tracing + tracing-subscriber)
   使用 tracing crate 生态系统

   Log format: JSON (for log aggregation systems)
   日志格式：JSON（供日志聚合系统使用）

   Example output / 示例输出：
   {
     "timestamp": "2025-06-15T10:23:45.123Z",
     "level": "INFO",
     "target": "nexuslink_community::router",
     "span": { "request_id": "01H5..." },
     "fields": {
       "message": "message routed",
       "sender": "nxl:8Wj3...",
       "channel": "general",
       "recipients": 42,
       "duration_ms": 3
     }
   }

   Log levels / 日志级别：
     ERROR  → service failures, unrecoverable errors
              服务故障、不可恢复错误
     WARN   → degraded operations, rate limit hits
              降级运行、速率限制触发
     INFO   → request/response, connection events
              请求/响应、连接事件
     DEBUG  → message routing details (disabled in production)
              消息路由细节（生产环境禁用）
     TRACE  → protocol-level details (development only)
              协议级细节（仅开发环境）

3. Automated Backup / 自动备份
   ───────────────────────────
   - PostgreSQL: pg_dump daily, WAL archiving for point-in-time recovery
     PostgreSQL：每日 pg_dump，WAL 归档用于时间点恢复
   - Media storage: incremental backup to secondary storage
     媒体存储：增量备份到二级存储
   - Backup encryption: AES-256 with server-side backup key
     备份加密：AES-256，服务端备份密钥
   - Retention: 30 days of daily backups, 12 months of monthly backups
     保留策略：30 天每日备份，12 个月每月备份
   - Restore test: automated monthly restore to staging environment
     恢复测试：每月自动恢复到预发布环境
```

---

## 8. P2P Protocol / P2P 协议

### 8.1 Connection Establishment / 连接建立

```
Alice (behind NAT)                    Bob (behind NAT)
Alice（NAT 后）                        Bob（NAT 后）

Step 1: Key Exchange (out-of-band) / 密钥交换（带外）
──────────────────────────────────────────────────
  Alice shows QR code containing:
  Alice 展示包含以下信息的二维码：
    - Her NexusLink UUID (public key)
    - A temporary libp2p PeerID
    - Optional: relay node address

  Bob scans → both parties know each other's identity
  Bob 扫描 → 双方知道对方身份

Step 2: Discovery / 发现
──────────────────────────
  Option A: Both online, use DHT to find each other
  选项 A：双方在线，使用 DHT 互相发现

  Option B: Use public relay/rendezvous node
  选项 B：使用公共中继/汇合节点

  Option C: Same LAN, use mDNS
  选项 C：同一局域网，使用 mDNS

Step 3: NAT Traversal / NAT 穿透
──────────────────────────────────
  1. Both query STUN server → learn public IP:port
     双方查询 STUN 服务器 → 获取公网 IP:端口
  2. Exchange candidates via relay (signaling)
     通过中继交换候选地址（信令）
  3. ICE tries direct connection (UDP hole punching)
     ICE 尝试直连（UDP 打洞）
  4. If direct fails → use TURN relay (still E2E encrypted)
     如果直连失败 → 使用 TURN 中继（仍端到端加密）

Step 4: Secure Channel / 安全通道
──────────────────────────────────
  Noise Protocol handshake (XX pattern) over libp2p
  通过 libp2p 进行 Noise 协议握手（XX 模式）
  → Authenticated, encrypted, multiplexed connection
  → 认证的、加密的、多路复用的连接
```

### 8.2 Offline Message Delivery / 离线消息投递

```
P2P mode challenge: What if Bob is offline?
P2P 模式挑战：如果 Bob 不在线怎么办？

Solution: Optional message relay / 解决方案：可选消息中继

  1. Alice encrypts message with Bob's public key
     Alice 用 Bob 的公钥加密消息
  2. Stores on a relay node (any community server or public relay)
     存储在中继节点（任意社区服务器或公共中继）
  3. When Bob comes online, fetches from relay
     Bob 上线后从中继获取
  4. Relay cannot read content (E2E encrypted)
     中继无法读取内容（端到端加密）
  5. Messages auto-expire after configurable TTL
     消息在可配置的 TTL 后自动过期
```

---

## 9. Message Format / 消息格式

```protobuf
// NexusLink Wire Protocol (Protocol Buffers)
// NexusLink 传输协议（Protocol Buffers）

syntax = "proto3";
package nexuslink.protocol;

message Envelope {
    string sender_uuid = 1;          // sender's NexusLink UUID
    string recipient_uuid = 2;       // recipient's UUID (or group ID)
    uint32 sender_device_id = 3;     // which device sent this
    MessageType type = 4;
    bytes  encrypted_payload = 5;    // E2E encrypted content
    bytes  sender_signature = 6;     // Ed25519 signature
    uint64 timestamp = 7;           // Unix timestamp (ms)
    string message_id = 8;          // unique message ID (UUID v7)
}

enum MessageType {
    TEXT = 0;
    IMAGE = 1;
    FILE = 2;
    VOICE = 3;
    VIDEO = 4;
    KEY_EXCHANGE = 10;              // X3DH initial message
    PREKEY_BUNDLE = 11;             // prekey upload
    DEVICE_ANNOUNCEMENT = 12;       // new device added
    DEVICE_REVOCATION = 13;         // device removed
    GROUP_SENDER_KEY = 20;          // sender key distribution
    GROUP_MEMBER_CHANGE = 21;       // member added/removed
    READ_RECEIPT = 30;
    TYPING_INDICATOR = 31;
    SYSTEM = 99;
}

message DecryptedContent {
    string text = 1;                 // text body
    repeated Attachment attachments = 2;
    QuotedMessage quote = 3;         // reply to another message
    map<string, string> metadata = 10;
}

message Attachment {
    string filename = 1;
    string mime_type = 2;
    uint64 size = 3;
    bytes  encryption_key = 4;       // per-attachment encryption key
    string download_url = 5;         // URL to fetch encrypted blob
    bytes  digest = 6;               // SHA-256 of encrypted blob
}
```

---

## 10. Storage / 存储

### 10.1 Client-Side / 客户端

```
Local encrypted database: SQLCipher (SQLite + AES-256)
本地加密数据库：SQLCipher（SQLite + AES-256）

Database key derived from:
数据库密钥来源：
  HKDF(device_key, "nexuslink-local-db", device_id)

Tables / 表：
  - messages        (decrypted message cache)      消息缓存
  - conversations   (chat list metadata)           会话列表
  - contacts        (known UUIDs + display names)  联系人
  - sessions        (Double Ratchet state)         棘轮状态
  - prekeys         (local prekey pool)            预密钥池
  - devices         (multi-device registry)        多设备注册
  - settings        (user preferences)             用户设置
```

### 10.2 Server-Side / 服务端

```
Community Server database / 社区服务器数据库：
  PostgreSQL (recommended) or SQLite (small deployments)
  PostgreSQL（推荐）或 SQLite（小规模部署）

Tables / 表：
  - members         (UUID, public key, prekey bundles)
  - channels        (channel metadata, permissions)
  - message_queue   (encrypted message blobs, pending delivery)
  - media_store     (encrypted file references)
  - audit_log       (moderation actions)

Directory Server database / 目录服务器数据库：
  PostgreSQL only
  仅 PostgreSQL

Tables / 表：
  - communities     (see Section 6.2)
  - kyc_records     (see Section 6.2)
  - abuse_reports   (reports, actions taken)
```

---

## 11. Security Analysis / 安全性分析

### 11.1 Encryption Strength / 加密强度

| Component | Algorithm | Security Level |
|-----------|-----------|---------------|
| Identity Key | Ed25519 | ~128-bit |
| Key Exchange | X25519 (X3DH) | ~128-bit |
| Message Encryption | AES-256-GCM (via Double Ratchet) | 256-bit |
| Group Encryption | AES-256-GCM (via Sender Keys) | 256-bit |
| Local Storage | AES-256-CBC (SQLCipher) | 256-bit |
| P2P Transport | Noise XX + ChaCha20-Poly1305 | 256-bit |
| Key Derivation | HKDF-SHA-256 | 256-bit |

### 11.2 Forward Secrecy / 前向保密

```
1:1 chats:    ✅ Full forward secrecy (Double Ratchet)
              每条消息使用不同密钥，密钥被删除后无法回溯解密

Group chats:  ⚠️ Partial (Sender Key ratchet is symmetric-only)
              发送者密钥仅使用对称棘轮，不如 Double Ratchet 强
              Mitigated by periodic Sender Key rotation
              通过定期轮换发送者密钥缓解

P2P:          ✅ Full forward secrecy (Noise XX + ratchet)
```

### 11.3 Post-Compromise Security / 入侵后安全

```
If an attacker compromises a session key:
如果攻击者获取了某个会话密钥：

1:1 chats:    ✅ Self-healing — next DH ratchet step generates
              new keys; attacker loses access
              自愈 — 下一次 DH 棘轮步骤生成新密钥；攻击者失去访问

Device compromised:
设备被入侵：
  1. User revokes device from another device
     用户从另一设备撤销该设备
  2. All contacts notified of device revocation
     所有联系人收到设备撤销通知
  3. New sessions established, old keys invalidated
     建立新会话，旧密钥失效
```

### 11.4 Metadata Exposure Matrix / 元数据暴露矩阵

| Data Point | Mode A (Official) | Mode B (Community) | Mode C (P2P) |
|-----------|-------------------|-------------------|-------------|
| Message content | Hidden | Hidden | Hidden |
| Who talks to whom | Hidden from directory | Visible to community server | Hidden (STUN sees IPs) |
| When | Hidden from directory | Visible to community server | Hidden |
| IP address | Hidden from directory | Visible to community server | Visible to STUN/peer |
| Group membership | Hidden from directory | Visible to community server | N/A |
| UUID existence | Known (if KYC'd) | Known to server | Known to peer only |

---

## 12. Server Shared Layer / 服务端共享层

Both the directory server and community server share common infrastructure code. This chapter documents the shared patterns and utilities.

目录服务器和社区服务器共享公共基础设施代码。本章记录共享的模式和工具。

### 12.1 Error Handling System / 错误处理体系

A unified error system ensures consistent error responses across all server APIs and simplifies debugging.

统一的错误体系确保所有服务端 API 返回一致的错误响应，并简化调试。

```rust
// Unified error type / 统一错误类型
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    // 4xx Client Errors / 客户端错误
    #[error("bad request: {message}")]
    BadRequest { message: String, code: ErrorCode },

    #[error("unauthorized")]
    Unauthorized { code: ErrorCode },

    #[error("forbidden: {message}")]
    Forbidden { message: String, code: ErrorCode },

    #[error("not found: {resource}")]
    NotFound { resource: String, code: ErrorCode },

    #[error("rate limited")]
    RateLimited { retry_after_secs: u32, code: ErrorCode },

    #[error("conflict: {message}")]
    Conflict { message: String, code: ErrorCode },

    // 5xx Server Errors / 服务端错误
    #[error("internal server error")]
    Internal { source: anyhow::Error, code: ErrorCode },

    #[error("database error")]
    Database { source: sqlx::Error, code: ErrorCode },

    #[error("service unavailable")]
    Unavailable { message: String, code: ErrorCode },
}
```

```
Error Code Specification / 错误码规范：

  Format: DOMAIN_CATEGORY_DETAIL
  格式：  域_类别_详情

  Domain prefixes / 域前缀：
    DIR_  → Directory server errors   目录服务器错误
    COM_  → Community server errors   社区服务器错误
    AUTH_ → Authentication errors     认证错误
    KYC_  → KYC-related errors        KYC 相关错误

  Examples / 示例：
    AUTH_TOKEN_EXPIRED        → Authentication token has expired
                                 认证令牌已过期
    AUTH_SIGNATURE_INVALID    → Ed25519 signature verification failed
                                 Ed25519 签名验证失败
    DIR_COMMUNITY_NOT_FOUND   → Community ID does not exist
                                 社区 ID 不存在
    DIR_KYC_REQUIRED          → KYC verification required for this action
                                 此操作需要 KYC 验证
    COM_CHANNEL_FORBIDDEN     → User lacks permission for this channel
                                 用户缺少此频道的权限
    COM_PREKEY_EXHAUSTED      → No one-time prekeys available
                                 无可用一次性预密钥
    COM_UPLOAD_TOO_LARGE      → File exceeds size limit
                                 文件超出大小限制
```

```json
// Client-friendly error response / 客户端友好错误响应
// HTTP 403 Forbidden
{
  "error": {
    "code": "COM_CHANNEL_FORBIDDEN",
    "message": "You do not have permission to send messages in this channel.",
    "details": {
      "channel_id": "01H5...",
      "required_role": "member",
      "current_role": "guest"
    },
    "request_id": "01H5...",
    "timestamp": "2025-06-15T10:23:45Z"
  }
}
```

### 12.2 Configuration Management / 配置管理

Servers are configured through a layered system: TOML config files, environment variable overrides, and limited runtime hot-reloading.

服务端通过分层系统配置：TOML 配置文件、环境变量覆盖，以及有限的运行时热更新。

```toml
# nexuslink-server.toml (example configuration)
# nexuslink-server.toml（配置示例）

[server]
bind_address = "0.0.0.0"
port = 443
tls_cert_path = "/certs/fullchain.pem"
tls_key_path = "/certs/privkey.pem"
max_connections = 50000

[database]
url = "postgresql://nexuslink:password@localhost:5432/nexuslink"
max_connections = 20
min_connections = 5

[rate_limit]
general_rpm = 60           # requests per minute
kyc_rpm = 5
search_rpm = 30

[push]
fcm_credentials_path = "/secrets/fcm.json"
apns_cert_path = "/secrets/apns.p8"
apns_team_id = "XXXXXXXXXX"

[media]
storage_path = "/data/media"
max_file_size = "100MB"
chunk_size = "4MB"

[backup]
enabled = true
schedule = "0 3 * * *"     # daily at 03:00 UTC
retention_days = 30

[logging]
level = "info"
format = "json"
```

```
Configuration Layer Priority (highest → lowest) /
配置层优先级（最高 → 最低）：

  1. Environment variables (NEXUSLINK_SERVER_PORT=8443)
     环境变量
     Convention: NEXUSLINK_ prefix + section_key in SCREAMING_SNAKE_CASE
     约定：NEXUSLINK_ 前缀 + section_key 大写蛇形

  2. TOML config file (nexuslink-server.toml)
     TOML 配置文件

  3. Compiled-in defaults
     编译时默认值

Runtime Hot-Reload (partial) / 运行时热更新（部分）：
  The server watches the config file for changes (via notify crate).
  服务端监控配置文件变化（通过 notify crate）。

  Hot-reloadable settings / 可热更新配置：
    - rate_limit.*          (rate limiting thresholds)
      速率限制阈值
    - logging.level         (log verbosity)
      日志详细级别
    - media.max_file_size   (upload size limit)
      上传大小限制

  Requires restart / 需重启：
    - server.bind_address, server.port   (network binding)
      网络绑定
    - database.*                          (connection pool)
      数据库连接池
    - server.tls_*                        (TLS certificates)
      TLS 证书
    - push.*                              (push service credentials)
      推送服务凭证
```

### 12.3 Database Migration / 数据库迁移

Database schema evolution is managed through SQLx's built-in migration system, ensuring reproducible and versioned schema changes.

数据库模式演进通过 SQLx 内置迁移系统管理，确保可复现和版本化的模式变更。

```
Migration Directory Structure / 迁移目录结构：

  migrations/
  ├── 20250101000000_initial_schema.sql
  ├── 20250115000000_add_push_tokens.sql
  ├── 20250201000000_add_channel_types.sql
  ├── 20250215000000_add_message_queue_partitioning.sql
  └── 20250301000000_add_prekey_expiry_index.sql

Migration Workflow / 迁移工作流：

  Development / 开发：
    $ sqlx migrate add <description>
    → Creates timestamped .sql file in migrations/
      在 migrations/ 中创建带时间戳的 .sql 文件

  Deployment / 部署：
    $ sqlx migrate run
    → Applies all pending migrations in order
      按顺序应用所有待执行迁移
    → Records applied migrations in _sqlx_migrations table
      在 _sqlx_migrations 表中记录已应用迁移

  Verification / 验证：
    $ sqlx migrate info
    → Shows status of all migrations (applied / pending)
      显示所有迁移的状态（已应用/待执行）

  Server Startup / 服务启动：
    On boot, the server automatically runs pending migrations
    before accepting connections.
    启动时，服务端在接受连接前自动运行待执行迁移。

Migration Rules / 迁移规则：
  - Migrations are forward-only (no automatic rollback)
    迁移仅向前（无自动回滚）
  - Rollback by creating a new migration that reverses changes
    回滚通过创建新迁移来反转变更
  - All migrations run within a transaction
    所有迁移在事务中运行
  - Breaking changes require two-phase migration:
    破坏性变更需要两阶段迁移：
      Phase 1: Add new column/table (backward compatible)
      阶段 1：添加新列/表（向后兼容）
      Phase 2: Remove old column/table (after code deployed)
      阶段 2：移除旧列/表（代码部署后）
```

### 12.4 Common Type Definitions / 公共类型定义

Shared types used across both servers to ensure consistency in API contracts and data handling.

两个服务端共享的类型，确保 API 契约和数据处理的一致性。

```rust
// UUID type / UUID 类型
// NexusLink UUIDs use the "nxl:" prefix format
// NexusLink UUID 使用 "nxl:" 前缀格式
pub struct NxlUuid(String);

impl NxlUuid {
    pub fn from_public_key(pk: &Ed25519PublicKey) -> Self {
        let hash = sha256(pk.as_bytes());
        let encoded = bs58::encode(&hash[..32]).into_string();
        Self(format!("nxl:{}", encoded))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// Timestamp / 时间戳
// All timestamps are UTC, stored as milliseconds since Unix epoch
// 所有时间戳为 UTC，存储为 Unix 纪元后的毫秒数
pub type Timestamp = i64;

pub fn now_ms() -> Timestamp {
    chrono::Utc::now().timestamp_millis()
}

// Pagination parameters / 分页参数
// Cursor-based pagination for all list endpoints
// 所有列表端点使用游标分页
#[derive(Deserialize)]
pub struct PaginationParams {
    /// Maximum number of items to return (1-100, default 20)
    /// 最大返回条数（1-100，默认 20）
    pub limit: Option<u32>,

    /// Opaque cursor from previous response
    /// 上次响应中的不透明游标
    pub cursor: Option<String>,
}

impl PaginationParams {
    pub fn effective_limit(&self) -> u32 {
        self.limit.unwrap_or(20).min(100).max(1)
    }
}

// API response wrapper / API 响应包装
// All successful responses follow this structure
// 所有成功响应遵循此结构
#[derive(Serialize)]
pub struct ApiResponse<T: Serialize> {
    pub data: T,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub pagination: Option<PaginationInfo>,
}

#[derive(Serialize)]
pub struct PaginationInfo {
    /// Cursor to fetch the next page
    /// 获取下一页的游标
    pub next_cursor: Option<String>,

    /// Whether more items exist beyond this page
    /// 是否还有更多条目
    pub has_more: bool,
}

// Usage example / 使用示例：
// GET /api/v1/community/search?q=gaming&limit=20&cursor=abc123
//
// Response:
// {
//   "data": [
//     { "id": "...", "name": "...", ... },
//     { "id": "...", "name": "...", ... }
//   ],
//   "pagination": {
//     "next_cursor": "def456",
//     "has_more": true
//   }
// }
```

---

## 13. Server Deployment Architecture / 服务端部署架构

This chapter covers deployment patterns for both small self-hosted communities and larger-scale production deployments.

本章涵盖从小型自托管社区到大规模生产部署的部署模式。

### 13.1 Single-Node Deployment / 单机部署方案

The simplest deployment: everything runs on one machine using Docker Compose. Suitable for small to medium communities (up to ~5,000 members).

最简单的部署方案：所有服务在一台机器上通过 Docker Compose 运行。适用于中小型社区（最多约 5,000 成员）。

```yaml
# docker-compose.yml (single-node deployment)
# docker-compose.yml（单机部署）
version: '3.8'

services:
  nexuslink-community:
    image: nexuslink/community-server:latest
    ports:
      - "443:443"
    volumes:
      - ./config:/etc/nexuslink
      - media_data:/data/media
      - certs:/certs
    environment:
      - NEXUSLINK_DATABASE_URL=postgresql://nexuslink:${DB_PASSWORD}@postgres:5432/nexuslink
      - NEXUSLINK_SERVER_BIND_ADDRESS=0.0.0.0
      - NEXUSLINK_SERVER_PORT=443
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G

  postgres:
    image: postgres:16-alpine
    volumes:
      - pg_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nexuslink
      - POSTGRES_USER=nexuslink
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nexuslink"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M

  # Optional: Redis for caching and rate limiting
  # 可选：用于缓存和速率限制的 Redis
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 192M

volumes:
  pg_data:
  media_data:
  redis_data:
  certs:
```

```
Single-Node Architecture / 单机架构：

┌─────────────────────────────────────────────────────┐
│                   Host Machine                       │
│                   宿主机                              │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │             Docker Compose                     │  │
│  │                                                │  │
│  │  ┌──────────────────┐  ┌────────────────────┐ │  │
│  │  │  Community Server│  │  PostgreSQL 16      │ │  │
│  │  │  社区服务器       │──│  数据库              │ │  │
│  │  │  Port: 443       │  │  Port: 5432         │ │  │
│  │  └──────────────────┘  └────────────────────┘ │  │
│  │           │                                    │  │
│  │           │            ┌────────────────────┐ │  │
│  │           └────────────│  Redis (optional)  │ │  │
│  │                        │  Port: 6379        │ │  │
│  │                        └────────────────────┘ │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  /data/media    (encrypted media files)        │  │
│  │  /data/pgdata   (PostgreSQL data)              │  │
│  │  /data/certs    (TLS certificates)             │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 13.2 Cluster Deployment / 集群部署方案

For larger communities or high-availability requirements: multiple community server instances behind a load balancer, with shared PostgreSQL and Redis for cross-instance coordination.

适用于大型社区或高可用性需求：多个社区服务器实例位于负载均衡器之后，共享 PostgreSQL 和 Redis 用于跨实例协调。

```
Cluster Architecture / 集群架构：

                    ┌──────────────────────┐
                    │   Load Balancer      │
                    │   负载均衡器          │
                    │   (HAProxy / nginx)  │
                    │                      │
                    │   TLS termination    │
                    │   TLS 终止           │
                    │   (or passthrough)   │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
     ┌────────────────┐ ┌──────────────┐ ┌──────────────┐
     │ Community Srv 1│ │ Comm. Srv 2  │ │ Comm. Srv 3  │
     │ 社区服务器 1    │ │ 社区服务器 2  │ │ 社区服务器 3  │
     └───────┬────────┘ └──────┬───────┘ └──────┬───────┘
             │                 │                │
             └────────┬────────┘                │
                      │                         │
              ┌───────▼───────┐                 │
              │    Redis      │◄────────────────┘
              │  (pub/sub +   │
              │   cache)      │
              │  发布/订阅+缓存│
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │  PostgreSQL   │
              │  (primary +   │
              │   replicas)   │
              │  主库 + 副本   │
              └───────────────┘

Cross-Instance Message Routing / 跨实例消息路由：
  Problem: Sender on Instance 1, Recipient on Instance 3
  问题：发送者在实例 1，接收者在实例 3

  Solution: Redis pub/sub / 解决方案：Redis 发布/订阅
    1. Instance 1 receives message from sender
       实例 1 从发送者接收消息
    2. Instance 1 checks local connection registry
       实例 1 检查本地连接注册表
    3. Recipient NOT found locally → publish to Redis channel
       接收者不在本地 → 发布到 Redis 频道
       PUBLISH nexuslink:route:{recipient_uuid} <envelope>
    4. Instance 3 (subscribed) receives and delivers to recipient
       实例 3（已订阅）接收并投递给接收者

Load Balancer Configuration / 负载均衡器配置：
  - WebSocket: sticky sessions (by client UUID cookie)
    WebSocket：粘性会话（按客户端 UUID cookie）
  - Health check: GET /health/ready every 10 seconds
    健康检查：每 10 秒 GET /health/ready
  - Draining: 30-second graceful shutdown on instance removal
    排空：实例移除时 30 秒优雅关闭
  - Algorithm: least-connections (best for WebSocket workloads)
    算法：最少连接数（最适合 WebSocket 负载）
```

### 13.3 TLS Certificate Management / TLS 证书管理

TLS is mandatory for all client-server communication. Certificate provisioning is automated through the ACME protocol (Let's Encrypt).

所有客户端-服务端通信强制使用 TLS。证书配置通过 ACME 协议（Let's Encrypt）自动化。

```
TLS Certificate Lifecycle / TLS 证书生命周期：

  Initial Provisioning / 初始配置：
  ──────────────────────────────────
  Option A: Built-in ACME client (recommended for single-node)
  选项 A：内置 ACME 客户端（单机推荐）
    - Server uses rustls-acme to automatically obtain certificates
      服务端使用 rustls-acme 自动获取证书
    - HTTP-01 challenge: server temporarily listens on port 80
      HTTP-01 挑战：服务端临时监听 80 端口
    - Certificate stored in /certs/ directory
      证书存储在 /certs/ 目录

  Option B: External certificate manager (recommended for clusters)
  选项 B：外部证书管理器（集群推荐）
    - certbot / acme.sh running as separate service
      certbot / acme.sh 作为独立服务运行
    - Load balancer handles TLS termination
      负载均衡器处理 TLS 终止
    - Certificates shared via mounted volume or secret manager
      证书通过挂载卷或密钥管理器共享

  Auto-Renewal / 自动续期：
  ──────────────────────────
  - Certificates checked daily for approaching expiry
    每日检查证书是否临近过期
  - Renewal triggered 30 days before expiry
    过期前 30 天触发续期
  - New certificate hot-swapped without restart (rustls supports this)
    新证书热替换无需重启（rustls 支持）
  - Failure alerts: notification sent if renewal fails 3 times
    失败告警：续期连续失败 3 次时发送通知

  TLS Configuration / TLS 配置：
  ─────────────────────────────
  - Minimum version: TLS 1.2 (TLS 1.3 preferred)
    最低版本：TLS 1.2（优先 TLS 1.3）
  - Cipher suites (TLS 1.3):
    TLS_AES_256_GCM_SHA384
    TLS_AES_128_GCM_SHA256
    TLS_CHACHA20_POLY1305_SHA256
  - HSTS header: max-age=63072000; includeSubDomains
  - OCSP stapling: enabled
    OCSP 装订：启用
```

### 13.4 Data Backup and Recovery / 数据备份与恢复策略

A comprehensive backup strategy protects against data loss from hardware failures, software bugs, or operational errors.

全面的备份策略防止因硬件故障、软件缺陷或运维失误导致的数据丢失。

```
Backup Strategy / 备份策略：

1. PostgreSQL Backup / PostgreSQL 备份
   ────────────────────────────────────
   Daily full backup / 每日全量备份：
     - pg_dump with --format=custom (compressed)
       pg_dump 使用 --format=custom（压缩）
     - Scheduled at 03:00 UTC (low-traffic window)
       安排在 UTC 03:00（低流量窗口）
     - Encrypted with AES-256 before storage
       存储前用 AES-256 加密

   Continuous WAL archiving / 持续 WAL 归档：
     - PostgreSQL WAL files archived to backup storage
       PostgreSQL WAL 文件归档到备份存储
     - Enables point-in-time recovery (PITR)
       支持时间点恢复（PITR）
     - RPO (Recovery Point Objective): < 5 minutes
       RPO（恢复点目标）：< 5 分钟

2. Media Backup / 媒体备份
   ────────────────────────
   - Incremental sync to secondary storage (rsync or rclone)
     增量同步到二级存储（rsync 或 rclone）
   - Media files already encrypted by clients; no additional
     encryption needed for backup
     媒体文件已由客户端加密；备份无需额外加密
   - Frequency: every 6 hours
     频率：每 6 小时

3. Configuration Backup / 配置备份
   ────────────────────────────────
   - Config files and secrets backed up separately
     配置文件和密钥独立备份
   - Version-controlled in private git repository (recommended)
     建议在私有 git 仓库中版本控制
   - Secrets encrypted with age or sops
     密钥用 age 或 sops 加密

Retention Policy / 保留策略：
  - Daily backups: retained for 30 days
    每日备份：保留 30 天
  - Weekly backups: retained for 3 months
    每周备份：保留 3 个月
  - Monthly backups: retained for 12 months
    每月备份：保留 12 个月

Recovery Procedures / 恢复流程：
  Scenario 1: Database corruption / 数据库损坏
    1. Stop community server
       停止社区服务器
    2. Restore latest pg_dump
       恢复最新 pg_dump
    3. Replay WAL to desired point in time
       重放 WAL 到目标时间点
    4. Verify data integrity
       验证数据完整性
    5. Restart community server
       重启社区服务器
    RTO (Recovery Time Objective): < 30 minutes
    RTO（恢复时间目标）：< 30 分钟

  Scenario 2: Full server loss / 服务器完全丢失
    1. Provision new server
       配置新服务器
    2. Deploy via Docker Compose
       通过 Docker Compose 部署
    3. Restore config from backup
       从备份恢复配置
    4. Restore PostgreSQL from backup
       从备份恢复 PostgreSQL
    5. Restore media from backup storage
       从备份存储恢复媒体
    6. Update DNS records
       更新 DNS 记录
    RTO: < 2 hours
```

### 13.5 Hardware Recommendations / 硬件推荐配置

Hardware requirements vary significantly based on community size. Below are tested recommendations.

硬件需求因社区规模而异。以下是经过测试的推荐配置。

```
Small Community (< 500 members, < 50 concurrent) /
小型社区（< 500 成员，< 50 并发）：
─────────────────────────────────────────────────
  CPU:     2 cores (ARM or x86_64)
  RAM:     2 GB
  Storage: 20 GB SSD (OS + DB + media)
  Network: 10 Mbps
  Cost:    ~$5-10/month VPS
  Examples: Raspberry Pi 4, entry-level VPS

  Can run on / 可运行于：
    - Raspberry Pi 4 (4 GB model)
    - Hetzner Cloud CX22
    - DigitalOcean Basic Droplet (2 GB)
    - Home server / NAS

Medium Community (500-5,000 members, 50-500 concurrent) /
中型社区（500-5,000 成员，50-500 并发）：
───────────────────────────────────────────────────────
  CPU:     4 cores
  RAM:     8 GB
  Storage: 100 GB SSD (OS + DB), 500 GB HDD/object storage (media)
  Network: 100 Mbps
  Cost:    ~$20-40/month VPS

Large Community (5,000-50,000 members, 500-5,000 concurrent) /
大型社区（5,000-50,000 成员，500-5,000 并发）：
─────────────────────────────────────────────────────────────
  CPU:     8+ cores
  RAM:     32 GB
  Storage: 500 GB NVMe SSD (DB), separate object storage (media)
  Network: 1 Gbps
  Cost:    ~$100-200/month dedicated server or multi-VPS

  Recommended: cluster deployment (see Section 13.2)
  推荐：集群部署（见第 13.2 节）
    - 2-3 community server instances (4 cores, 8 GB each)
      2-3 个社区服务器实例（各 4 核 8 GB）
    - 1 PostgreSQL primary + 1 replica (8 cores, 16 GB each)
      1 个 PostgreSQL 主库 + 1 个副本（各 8 核 16 GB）
    - 1 Redis instance (2 cores, 4 GB)
      1 个 Redis 实例（2 核 4 GB）
    - Load balancer (2 cores, 2 GB)
      负载均衡器（2 核 2 GB）

Key Scaling Bottlenecks / 关键扩展瓶颈：
  1. WebSocket connections: limited by RAM (~8 KB per connection)
     WebSocket 连接：受 RAM 限制（每连接约 8 KB）
  2. Message fan-out: CPU-bound for large channels
     消息扇出：大频道时受 CPU 限制
  3. Media storage: disk I/O and capacity
     媒体存储：磁盘 I/O 和容量
  4. Database queries: connection pool and query complexity
     数据库查询：连接池和查询复杂度
```

---

## 14. Bot System / 机器人系统

### 14.0 Design Philosophy / 设计理念

In Discord, bots run server-side with full access to plaintext messages — essentially "God Mode." In NexusLink, a Bot is a **Headless Client**: it has its own Ed25519 key pair and UUID, connects to a Community Server using the same WebSocket protocol as regular clients, and must participate in E2E key exchange to read any message.

在 Discord 中，Bot 运行在服务端，拥有对明文消息的完全访问权——本质上是「上帝模式」。在 NexusLink 中，Bot 是一个**无头客户端（Headless Client）**：它拥有自己的 Ed25519 密钥对和 UUID，使用与普通客户端相同的 WebSocket 协议连接社区服务器，并且必须参与 E2E 密钥交换才能读取任何消息。

**Core principles / 核心原则:**

- **Bot is a cryptographic participant, not an observer / Bot 是加密参与者，不是旁观者** — A Bot must be invited into an E2E session and receive session keys to read messages. / Bot 必须被邀请进入 E2E 会话并获得会话密钥才能读取消息。
- **Bot presence is always visible / Bot 存在始终可见** — When a Bot is in a channel, all members see a Bot badge. No invisible surveillance. / 当 Bot 在频道中时，所有成员都能看到 Bot 标识。不存在隐形监听。
- **Permissions controlled by community RBAC / 权限由社区 RBAC 控制** — Admins decide which channels a Bot can join and what actions it can perform. / 管理员决定 Bot 能进哪些频道、能执行什么操作。
- **Bot SDK built on Rust Core / Bot SDK 基于 Rust Core** — Reuses existing crypto and protocol layers. Bot developers never implement cryptography themselves. / 复用现有的加密和协议层。Bot 开发者无需自己实现密码学。

### 14.1 Bot Types / 机器人类型

| Type / 类型 | Runtime Location / 运行位置 | Typical Use Cases / 典型用途 |
|---|---|---|
| **Community Bot / 社区机器人** | Community admin's server / 社区管理员的服务器 | Auto-moderation, welcome messages, channel management, scheduled announcements / 自动审核、欢迎新人、频道管理、定时公告 |
| **Personal Bot / 个人机器人** | User's own device or VPS / 用户自己的设备或 VPS | Message forwarding, auto-reply, account hosting, multi-device sync proxy / 消息转发、自动回复、账户托管、多设备同步代理 |
| **Third-party Bot / 第三方机器人** | Third-party developer hosted / 第三方开发者托管 | Translation, RSS feeds, games, polls, AI assistants / 翻译、RSS 推送、游戏、投票、AI 助手 |


### 14.2 Architecture / 架构

```
Bot Architecture / Bot 架构

┌─────────────────────────────────────────────────────┐
│  Bot Process (Headless) / 机器人进程（无头）           │
│                                                       │
│  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  Bot Logic        │  │ NexusLink Bot SDK (Rust) │  │
│  │  机器人逻辑        │  │                          │  │
│  │                    │  │  • Crypto (E2E)          │  │
│  │  - Command handler │◄►│  • WebSocket client      │  │
│  │  - Event listener  │  │  • Prekey management     │  │
│  │  - Response sender │  │  • Message encode/decode │  │
│  │  - State storage   │  │  • Permission checker    │  │
│  └──────────────────┘  └────────────┬─────────────┘  │
└─────────────────────────────────────┼─────────────────┘
                                      │ WSS (same protocol as regular client)
                                      │ 与普通客户端相同的协议
                                      ▼
                        ┌──────────────────────────┐
                        │   Community Server        │
                        │   社区服务器               │
                        │                           │
                        │   Bot = just another      │
                        │   member with a UUID      │
                        │   and a Bot badge         │
                        │                           │
                        │   Bot = 只是另一个有       │
                        │   UUID 和 Bot 标识的成员   │
                        └──────────────────────────┘
```

**Key point / 关键点:** The Community Server does not distinguish between a Bot and a human client at the protocol level. Both use the same WebSocket connection, the same message format, and the same E2E encryption. The only difference is a `bot: true` flag in the member profile, which triggers the visible Bot badge in other clients' UI.

**关键点：** 社区服务器在协议层面不区分 Bot 和人类客户端。两者使用相同的 WebSocket 连接、相同的消息格式和相同的 E2E 加密。唯一的区别是成员资料中的 `bot: true` 标志，它会在其他客户端的 UI 中触发可见的 Bot 标识。


### 14.3 Bot Lifecycle / 机器人生命周期

```
Bot Lifecycle / Bot 生命周期

  Developer                    Community Admin              Community Server
  开发者                        社区管理员                     社区服务器

     │                              │                            │
     │  1. Create Bot               │                            │
     │     Generate key pair         │                            │
     │     生成密钥对                 │                            │
     │                              │                            │
     │  2. Register Bot ────────────►│                            │
     │     Submit UUID + public key  │                            │
     │     提交 UUID + 公钥          │                            │
     │                              │                            │
     │                              │  3. Review & Approve       │
     │                              │     审核并批准              │
     │                              │                            │
     │                              │  4. Assign permissions ───►│
     │                              │     Channels + RBAC roles  │
     │                              │     分配频道 + RBAC 角色    │
     │                              │                            │
     │  5. Connect ─────────────────────────────────────────────►│
     │     Upload prekeys            │                            │
     │     上传预密钥                 │                            │
     │                              │                            │
     │  6. Active ◄─────────────────────────────────────────────►│
     │     Receive events, send responses                        │
     │     接收事件，发送响应                                      │
     │                              │                            │
     │                              │  7. Revoke (anytime)       │
     │                              │     随时可撤销              │
     │                              │     → Key rotation triggered│
     │                              │       触发密钥轮换          │
```

**On removal / 移除时:** When a Bot is removed from a channel or community, a key rotation is triggered for all remaining participants. This ensures Forward Secrecy — the removed Bot cannot decrypt any future messages even if it retained the old session keys.

**移除时：** 当 Bot 从频道或社区中移除时，会为所有剩余参与者触发密钥轮换。这确保了前向保密——被移除的 Bot 即使保留了旧的会话密钥也无法解密任何未来的消息。


### 14.4 Privacy Comparison with Discord / 与 Discord 的隐私对比

| Aspect / 方面 | Discord | NexusLink |
|---|---|---|
| Bot access to messages / Bot 对消息的访问 | All messages in permitted channels (plaintext) / 允许频道中的所有消息（明文） | Only channels where Bot participates in E2E key exchange / 仅 Bot 参与 E2E 密钥交换的频道 |
| Invisible bots / 隐形 Bot | Possible with certain permissions / 某些权限下可能 | Impossible by design — Bot badge always visible / 设计上不可能——Bot 标识始终可见 |
| Server-side access / 服务端访问 | Bot runs server-side with API token / Bot 使用 API token 在服务端运行 | Bot is a client — server never sees plaintext / Bot 是客户端——服务器永远看不到明文 |
| Removal impact / 移除影响 | Bot loses API access / Bot 失去 API 访问 | Key rotation ensures Forward Secrecy / 密钥轮换确保前向保密 |
| Data retention / 数据留存 | Bot can store all historical messages / Bot 可存储所有历史消息 | Bot only has messages from its active participation period / Bot 仅拥有其活跃参与期间的消息 |


### 14.5 Bot SDK Overview / Bot SDK 概览

The Bot SDK is a Rust crate (`nexuslink-bot-sdk`) that wraps the core library and provides a developer-friendly API:

Bot SDK 是一个 Rust crate（`nexuslink-bot-sdk`），封装核心库并提供开发者友好的 API：

```rust
// Conceptual API — not final implementation
// 概念性 API——非最终实现

use nexuslink_bot_sdk::{Bot, Event, Context};

#[tokio::main]
async fn main() {
    let bot = Bot::builder()
        .key_path("bot_keys.enc")       // Encrypted key storage / 加密密钥存储
        .server("wss://community.example.com")
        .build()
        .await
        .unwrap();

    bot.on_event(|event: Event, ctx: Context| async move {
        match event {
            Event::Message(msg) => {
                if msg.content.starts_with("/hello") {
                    ctx.reply("Hello from Bot!").await?;
                }
            }
            Event::MemberJoin(member) => {
                ctx.send(format!("Welcome, {}!", member.display_name)).await?;
            }
            _ => {}
        }
        Ok(())
    });

    bot.run().await;
}
```


**SDK responsibilities / SDK 职责:**

| Layer / 层 | Handled by SDK / SDK 处理 | Developer implements / 开发者实现 |
|---|---|---|
| Key generation & storage / 密钥生成与存储 | ✓ | |
| E2E encryption & decryption / E2E 加解密 | ✓ | |
| WebSocket connection & reconnection / WebSocket 连接与重连 | ✓ | |
| Prekey upload & rotation / 预密钥上传与轮换 | ✓ | |
| Message serialization (Protobuf) / 消息序列化 | ✓ | |
| Permission checking / 权限检查 | ✓ | |
| Command parsing / 命令解析 | | ✓ |
| Business logic / 业务逻辑 | | ✓ |
| State management / 状态管理 | | ✓ |
| External API calls / 外部 API 调用 | | ✓ |

### 14.6 Use Case Examples / 使用场景示例

| Scenario / 场景 | Bot Type / Bot 类型 | How it works / 工作方式 |
|---|---|---|
| Auto-reply when offline / 离线时自动回复 | Personal Bot / 个人机器人 | Hosts user's account on a VPS, responds with preset messages / 在 VPS 上托管用户账户，使用预设消息回复 |
| Community translation / 社区翻译 | Community Bot / 社区机器人 | Listens to messages, calls translation API, replies with translated text / 监听消息，调用翻译 API，回复翻译文本 |
| Poll system / 投票系统 | Community Bot / 社区机器人 | Responds to `/vote` commands, tracks votes, announces results / 响应 `/vote` 命令，统计投票，公布结果 |
| AI chat assistant / AI 聊天助手 | Third-party Bot / 第三方机器人 | Forwards decrypted messages to LLM API, returns AI response / 将解密消息转发到 LLM API，返回 AI 响应 |
| RSS feed / RSS 推送 | Community Bot / 社区机器人 | Periodically fetches RSS feeds, posts new items to designated channel / 定期获取 RSS 源，将新条目发布到指定频道 |
| Moderation / 内容审核 | Community Bot / 社区机器人 | Scans messages against rules, warns or mutes violators / 根据规则扫描消息，警告或禁言违规者 |

---

## Appendix: Cryptographic Library Dependencies / 附录：密码学库依赖

```toml
# Cargo.toml dependencies for nexuslink-core
[dependencies]
# Secure hardware abstraction / 安全硬件抽象
nmshd-rust-crypto = "x.x"         # Cross-platform secure element

# Cryptographic primitives / 密码学原语
ed25519-dalek = "2.x"             # Ed25519 signing
x25519-dalek = "2.x"              # X25519 key exchange
aes-gcm = "0.10"                  # AES-256-GCM encryption
chacha20poly1305 = "0.10"         # ChaCha20-Poly1305
hkdf = "0.12"                     # HKDF key derivation
sha2 = "0.10"                     # SHA-256
rand = "0.8"                      # Secure random generation

# Double Ratchet / 双棘轮
# (custom implementation or adapted from signal-protocol crate)

# P2P networking / P2P 网络
libp2p = { version = "0.53", features = ["webrtc", "noise", "quic"] }

# Serialization / 序列化
prost = "0.12"                    # Protocol Buffers
prost-types = "0.12"

# Local storage / 本地存储
rusqlite = { version = "0.31", features = ["bundled-sqlcipher"] }

# FFI / 外部函数接口
flutter_rust_bridge = "2.x"
```
