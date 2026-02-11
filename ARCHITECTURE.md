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

---

## 2. Identity System / 身份系统

### 2.1 Key Hierarchy / 密钥层级

```
Identity Key (IK) — 身份密钥
│  Curve: Ed25519 (signing) + X25519 (key exchange)
│  Generated once, on the first device
│  Public key = User UUID (Base58 encoded, ~44 characters)
│  Example: "nxl:8Wj3KpR2vT5mN7xQ1cY4bF6hA9dL0sE3gU8iO2pJ5"
│
├── Device Signing Key (DSK) — 设备签名密钥
│   │  One per device, generated in secure hardware
│   │  Signed by IK to prove it belongs to this identity
│   │  Platform mapping:
│   │    iOS     → Secure Enclave (P-256 / Ed25519)
│   │    Android → StrongBox / TEE (P-256 / Ed25519)
│   │    macOS   → Secure Enclave (Apple Silicon / T2)
│   │    Windows → TPM 2.0 (RSA-2048 / P-256)
│   │    Linux   → TPM 2.0, or software fallback (encrypted keyfile)
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
```

### 2.2 First Launch Flow / 首次启动流程

```
┌─────────────────────────────────────────────────┐
│                  First Launch                    │
│                  首次启动                         │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Generate IK in       │
            │  secure hardware      │
            │  在安全硬件中生成 IK    │
            └───────────┬───────────┘
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
            │  Display mnemonic     │
            │  recovery phrase      │
            │  (user MUST write     │
            │   down securely)      │
            │  显示助记词恢复短语     │
            │  （用户必须安全抄写）   │
            └───────────┬───────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  Verify: user re-     │
            │  enters phrase        │
            │  验证：用户重新输入     │
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
