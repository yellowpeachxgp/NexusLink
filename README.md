<p align="center">
  <h1 align="center">NexusLink</h1>
  <p align="center">
    <strong>混合架构即时通讯平台 -- 中心化与去中心化的桥梁</strong>
  </p>
  <p align="center">
    <a href="./README_EN.md">English</a> |
    <a href="./ARCHITECTURE.md">架构设计</a> |
    <a href="./ROADMAP.md">开发路线图</a> |
    <a href="./CONTRIBUTING.md">参与贡献</a>
  </p>
</p>

---

## 目录

- [NexusLink 是什么](#nexuslink-是什么)
- [核心特性](#核心特性)
- [架构概览](#架构概览)
- [服务端开发](#服务端开发)
- [技术栈](#技术栈)
- [快速开始](#快速开始)
- [项目结构](#项目结构)
- [与现有方案的对比](#与现有方案的对比)
- [安全模型](#安全模型)
- [开发路线图](#开发路线图)
- [参与贡献](#参与贡献)
- [许可证](#许可证)
- [致谢](#致谢)

---

## NexusLink 是什么

NexusLink 是一个开源、跨平台的即时通讯应用，独特地在**同一个 App 中融合了中心化与去中心化通讯**。用户自主选择连接方式——官方目录服务器、自建社区服务器或点对点直连——无需在易用性和安全性之间妥协。

**无需手机号。无需邮箱。无需注册。** 你的身份是由设备安全硬件生成的加密密钥对。身份属于你自己。

### 当前痛点

| 中心化应用（微信、Telegram、Discord） | 去中心化应用（Signal、Briar、Session） |
|---|---|
| 易用、功能丰富 | 隐私优先、抗审查 |
| 用户交出数据主权 | 社交/社区功能受限 |
| 单点故障与监控风险 | 用户体验割裂 |
| 无法选择数据托管方 | 普通用户技术门槛高 |

**NexusLink 填补这个空白** -- 兼具中心化平台的社区功能与去中心化的隐私和数据主权，让每个用户自己决定信任级别。

---

## 核心特性

### 无需注册的身份系统
- 设备安全硬件（TPM / Secure Enclave / StrongBox）生成加密密钥对
- 公钥 **即** UUID -- 身份创建完全不依赖服务器
- 通过交叉签名协议实现多设备同步
- 助记词恢复短语（12-24 个单词）支持身份迁移

### 三种连接模式

```
+--------------------------------------------------------------+
|                     NexusLink 客户端                          |
|                                                              |
|  [A] 官方服务器        [B] 社区服务器         [C] P2P 直连    |
|  - 实名认证           - 自建自管             - 设备直连       |
|  - 浏览社区目录       - 数据主权             - 无需服务器     |
|  - 官方监管频道       - 自定义规则           - 端到端加密     |
+--------------------------------------------------------------+
```

**模式 A：官方目录服务器**
- 需要 UUID 实名认证（KYC）
- 浏览和加入已注册的社区
- 访问官方公共频道
- 防滥用保护机制

**模式 B：社区自建服务器**
- 任何人都可以用同一套服务端代码部署
- 可选择注册到官方目录以获得可发现性
- 完全的数据主权 -- 官方服务器永远看不到你的消息、成员或元数据
- 社区自定义管理规则

**模式 C：点对点直连**
- 通过 libp2p + WebRTC 实现设备间直接通讯
- 通过二维码或线下方式交换公钥
- 零服务器依赖（仅需 STUN 进行 NAT 穿透）
- 敏感对话的最高隐私级别

### 架构级隐私保护
- 所有消息端到端加密（Double Ratchet 协议）
- 官方目录服务器**仅是电话簿** -- 只存储社区列表，不存储消息和成员信息
- 社区服务器是**隐私隔离层** -- 元数据不会流向官方服务器
- 为未来的零知识证明 KYC 预留接口（证明身份而不泄露细节）

### 全平台支持
- iOS、Android、Windows、macOS、Linux
- Flutter UI + Rust 核心，通过 FFI 桥接
- 所有平台体验一致

---

## 架构概览

```
+--------------------------------------------------------------+
|                          客户端                               |
|  +--------------------+  +----------------------------------+|
|  |   Flutter UI 层    |  |       Rust 核心库                ||
|  |                    |<-|  - 身份管理（安全硬件）            ||
|  |  - 聊天界面        |  |  - 端到端加密                     ||
|  |  - 社区浏览        |  |  - P2P 网络（libp2p）             ||
|  |  - 设置            |  |  - 协议编解码                     ||
|  |  - 多语言支持      |  |  - 本地加密存储                   ||
|  +--------------------+  +----------------------------------+|
+------------------+-------------------+-----------------------+
                   |                   |
         +---------+                   +----------+
         v                                        v
+---------------------+               +-----------------------+
|   目录服务器（官方） |               |   社区服务器（自建）   |
|                     |               |                       |
|  - 社区索引          |    注册       |  - 聊天托管           |
|  - 实名认证          |<-------------|  - 成员管理           |
|  - 滥用举报          |  （仅元信息） |  - 文件存储           |
|                     |               |  - 自定义管理         |
+---------------------+               +-----------------------+
                                               |
                                      P2P 直连通道
                                               |
                                      +--------v----------+
                                      |    其他客户端       |
                                      +-------------------+
```

详细架构文档请参阅 [架构设计文档](./ARCHITECTURE.md)。

---

## 服务端开发

NexusLink 的服务端由两个角色组成：**社区服务器**（Community Server）和**目录服务器**（Directory Server）。它们共享同一套代码基础（`server/shared/` 模块），通过不同的配置和功能模块实现各自的职责。同一个服务端二进制文件可以通过启动参数切换角色，降低维护成本并保持一致的代码质量。

```
+-----------------------------------------------------------------------+
|                        NexusLink 服务端架构                             |
|                                                                       |
|  +---------------------------+    +-------------------------------+   |
|  |      目录服务器（官方）     |    |       社区服务器（自建）       |   |
|  |  - 社区索引与搜索          |    |  - WebSocket 实时消息路由     |   |
|  |  - KYC 实名认证            |    |  - 预密钥管理                |   |
|  |  - 滥用举报处理            |    |  - 频道/群组管理             |   |
|  |  - 反垃圾与限流            |    |  - 加密文件存储              |   |
|  +-------------+-------------+    |  - 推送通知中继              |   |
|                |                  |  - 成员管理与审核            |   |
|                |   注册（元信息）  |  - 速率限制                  |   |
|                +----------------->|  - 健康检查                  |   |
|                                   +-------------------------------+   |
|                                                                       |
|  +---------------------------------------------------------------+   |
|  |                  共享代码层 (server/shared/)                    |   |
|  |  - 配置管理        - 错误类型        - 公共类型定义             |   |
|  |  - 工具函数        - 中间件          - 数据库连接池             |   |
|  +---------------------------------------------------------------+   |
+-----------------------------------------------------------------------+
```

### 社区服务器

社区服务器是 NexusLink 架构中的核心服务组件。任何人都可以使用同一套开源代码部署自己的社区服务器，实现完全的数据主权。

#### 职责

- 托管社区内的所有通讯（消息路由、文件存储）
- 管理社区成员的预密钥包（用于 X3DH 密钥协商）
- 维护频道、群组及其权限体系
- 处理加密文件的上传和下载
- 向离线客户端推送通知（FCM / APNs）
- 提供成员管理和内容审核工具
- 可选：向官方目录服务器注册以获取可发现性

#### 核心模块

**WebSocket 实时通讯**

社区服务器使用 WebSocket 长连接实现消息的实时推送。客户端连接后通过设备密钥签名进行身份验证，服务器维护活跃连接池并负责消息的扇出分发（同一用户的多设备同步）。

```
客户端 A --[WebSocket]--> 社区服务器 --[WebSocket]--> 客户端 B
                              |
                              +--[WebSocket]--> 客户端 B（设备 2）
                              |
                              +--[消息队列]--> 离线客户端 C（上线后投递）
```

**消息队列**

对于离线用户，服务器将加密消息暂存于消息队列中。消息存储为不透明的加密数据块，服务器无法解读内容。每条消息设置可配置的 TTL（生存时间），过期后自动清理。

**预密钥管理**

服务器存储每个成员的预密钥包（Prekey Bundle），供其他用户发起 X3DH 密钥协商时获取。当预密钥库存不足时，服务器通过推送通知提醒客户端补充。

**频道与群组管理**

支持多种频道类型（文本频道、公告频道等），提供细粒度的权限控制：
- 管理员（Admin）：完全控制权
- 版主（Moderator）：内容审核、成员管理
- 普通成员（Member）：发言、阅读

**文件存储**

所有上传的文件均为客户端加密后的密文。服务器仅负责存储和分发，无法解密文件内容。支持缩略图生成（针对加密后的元数据）、文件大小限制和存储配额管理。

**推送通知**

集成 FCM（Android）和 APNs（iOS）推送服务。推送内容不包含消息明文，仅通知客户端有新消息需要拉取。

**成员管理与审核**

提供完整的审核工具集：禁言、封禁、消息删除（仅删除元数据引用，无法读取消息内容）、审核日志等。

**速率限制**

基于 tower 中间件的多层速率限制策略：
- 每 IP 请求速率限制
- 每用户 API 调用频率限制
- WebSocket 消息发送频率限制
- 文件上传速率和大小限制

**健康检查**

提供 `/health` 端点用于负载均衡和容器编排平台的健康状态探测，返回服务器运行状态、数据库连接状态和关键指标。

#### API 概览

```
WebSocket  /ws                          - 实时消息通道（认证后升级）

POST   /api/v1/auth/challenge          - 请求认证挑战
POST   /api/v1/auth/verify             - 提交签名验证身份

GET    /api/v1/prekeys/{uuid}          - 获取用户预密钥包
POST   /api/v1/prekeys                 - 上传预密钥

POST   /api/v1/messages/send           - 发送加密消息
GET    /api/v1/messages/pending        - 拉取待接收消息
DELETE /api/v1/messages/{id}           - 确认消息已接收（从队列移除）

POST   /api/v1/channels                - 创建频道
GET    /api/v1/channels                - 获取频道列表
GET    /api/v1/channels/{id}           - 获取频道详情
PUT    /api/v1/channels/{id}           - 更新频道设置
DELETE /api/v1/channels/{id}           - 删除频道

GET    /api/v1/members                 - 获取成员列表
POST   /api/v1/members/join            - 加入社区
DELETE /api/v1/members/{uuid}          - 移除成员
PUT    /api/v1/members/{uuid}/role     - 修改成员角色

POST   /api/v1/media/upload            - 上传加密文件
GET    /api/v1/media/{id}              - 下载加密文件

POST   /api/v1/devices/register        - 注册设备（多设备支持）
DELETE /api/v1/devices/{device_id}     - 注销设备

GET    /health                          - 健康检查
GET    /metrics                         - Prometheus 指标
```

#### 数据库设计

社区服务器使用 PostgreSQL（推荐）或 SQLite（小规模部署）作为持久化存储：

```sql
-- 社区成员表：存储成员公钥和预密钥包
members (
    uuid            VARCHAR(60) PRIMARY KEY,   -- NexusLink UUID
    public_key      BYTEA NOT NULL,            -- 成员身份公钥
    display_name    VARCHAR(100),              -- 显示名称（可选）
    joined_at       TIMESTAMPTZ NOT NULL,
    role            VARCHAR(20) DEFAULT 'member',
    is_banned       BOOLEAN DEFAULT FALSE
)

-- 设备表：支持多设备
devices (
    device_id       SERIAL PRIMARY KEY,
    member_uuid     VARCHAR(60) REFERENCES members(uuid),
    device_public_key BYTEA NOT NULL,          -- 设备签名公钥
    device_cert     BYTEA NOT NULL,            -- IK 签发的设备证书
    push_token      VARCHAR(500),              -- FCM / APNs 推送令牌
    registered_at   TIMESTAMPTZ NOT NULL
)

-- 预密钥表：X3DH 密钥协商所需
prekeys (
    id              SERIAL PRIMARY KEY,
    member_uuid     VARCHAR(60) REFERENCES members(uuid),
    prekey_id       INTEGER NOT NULL,
    public_key      BYTEA NOT NULL,            -- 一次性预密钥公钥
    is_used         BOOLEAN DEFAULT FALSE,
    uploaded_at     TIMESTAMPTZ NOT NULL
)

-- 签名预密钥表
signed_prekeys (
    id              SERIAL PRIMARY KEY,
    member_uuid     VARCHAR(60) REFERENCES members(uuid),
    key_id          INTEGER NOT NULL,
    public_key      BYTEA NOT NULL,
    signature       BYTEA NOT NULL,            -- IK 的签名
    uploaded_at     TIMESTAMPTZ NOT NULL
)

-- 频道表
channels (
    id              UUID PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(20) DEFAULT 'text', -- text / announcement / voice
    description     TEXT,
    created_by      VARCHAR(60) REFERENCES members(uuid),
    created_at      TIMESTAMPTZ NOT NULL,
    is_archived     BOOLEAN DEFAULT FALSE
)

-- 频道权限表
channel_permissions (
    channel_id      UUID REFERENCES channels(id),
    role            VARCHAR(20),               -- admin / moderator / member
    can_read        BOOLEAN DEFAULT TRUE,
    can_write       BOOLEAN DEFAULT TRUE,
    can_manage      BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (channel_id, role)
)

-- 消息队列表：暂存待投递的加密消息
message_queue (
    id              BIGSERIAL PRIMARY KEY,
    recipient_uuid  VARCHAR(60) NOT NULL,
    device_id       INTEGER,                   -- NULL 表示发送到所有设备
    encrypted_blob  BYTEA NOT NULL,            -- 不透明加密数据
    channel_id      UUID REFERENCES channels(id),
    created_at      TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL       -- TTL 过期时间
)

-- 加密媒体文件引用表
media_store (
    id              UUID PRIMARY KEY,
    uploader_uuid   VARCHAR(60) REFERENCES members(uuid),
    encrypted_size  BIGINT NOT NULL,           -- 加密后文件大小
    mime_type_hint  VARCHAR(100),              -- 客户端提供的类型提示
    storage_path    VARCHAR(500) NOT NULL,     -- 服务器本地存储路径
    digest          BYTEA NOT NULL,            -- 加密数据块的 SHA-256
    uploaded_at     TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ                -- 可选过期时间
)

-- 审核日志
audit_log (
    id              BIGSERIAL PRIMARY KEY,
    actor_uuid      VARCHAR(60),               -- 操作者
    action          VARCHAR(50) NOT NULL,       -- ban / mute / delete / role_change
    target_uuid     VARCHAR(60),               -- 目标用户
    target_channel  UUID,                      -- 目标频道
    details         JSONB,                     -- 操作详情
    created_at      TIMESTAMPTZ NOT NULL
)
```

### 目录服务器

目录服务器是 NexusLink 架构中唯一的中心化组件，其角色被严格限制为**电话簿**：仅维护社区的索引信息，不参与任何消息传递。

#### 职责

- 维护已注册社区的公开索引（名称、描述、标签、连接地址）
- 提供社区搜索和发现服务
- 管理 UUID 实名认证（KYC）流程
- 处理滥用举报
- 为整个生态提供信任锚点

目录服务器**不**存储任何消息内容、成员列表、用户活动数据或通讯元数据。

#### API 概览

```
POST   /api/v1/community/register      - 注册社区（需 KYC 认证）
GET    /api/v1/community/search        - 搜索社区（全文搜索 + 标签过滤）
GET    /api/v1/community/{id}          - 获取社区详情
PUT    /api/v1/community/{id}          - 更新社区信息
DELETE /api/v1/community/{id}          - 注销社区

POST   /api/v1/kyc/initiate           - 发起实名认证
POST   /api/v1/kyc/verify             - 提交认证材料
GET    /api/v1/kyc/status             - 查询认证状态

POST   /api/v1/report/abuse           - 提交滥用举报
GET    /api/v1/report/{id}            - 查询举报处理状态

GET    /health                          - 健康检查
GET    /metrics                         - Prometheus 指标
```

#### KYC 流程

NexusLink 的实名认证是**可选**的，仅在用户需要使用官方目录服务时才需要。认证流程如下：

```
1. 用户在客户端发起 KYC 请求
   客户端使用设备密钥对请求签名
       |
       v
2. 目录服务器返回认证挑战
   包含随机 nonce 和所需材料清单
       |
       v
3. 用户提交认证材料
   身份文件加密上传，仅合规审核团队可解密
       |
       v
4. 审核处理
   当前阶段：人工审核
   未来规划：ZKP（零知识证明）-- 证明身份而不泄露细节
       |
       v
5. 认证结果
   approved（通过） / rejected（拒绝）
   通过后 UUID 获得"已认证"标记
```

#### 社区注册流程

```
1. 社区管理员在社区服务器上执行注册命令
       |
       v
2. 社区服务器向目录服务器提交注册请求
   包含：社区名称、描述、标签、连接端点 URL、
         管理员 UUID、社区服务器公钥
       |
       v
3. 目录服务器验证
   - 管理员 UUID 是否已通过 KYC
   - 端点 URL 是否可达
   - 社区服务器公钥签名是否有效
       |
       v
4. 注册完成
   社区出现在目录搜索结果中
   目录定期探测社区端点以确认可用性
```

#### 反滥用机制

- **请求速率限制**：基于 IP 和 UUID 的多层限流
- **注册工作量证明**：社区注册需要完成计算密集型挑战，防止批量注册
- **滥用举报与审核**：用户可举报社区，达到阈值后自动暂停并进入人工审核队列
- **异常检测**：监控注册模式、搜索模式等行为特征，识别自动化攻击
- **社区信誉评分**：基于运行时间、举报数量、成员活跃度等指标计算社区可信度

### 服务端部署

#### Docker 一键部署（推荐）

部署社区服务器：

```bash
# 创建工作目录
mkdir nexuslink-community && cd nexuslink-community

# 下载 docker-compose 配置
curl -O https://raw.githubusercontent.com/yellowpeachxgp/NexusLink/main/server/docker-compose.yml

# 编辑环境变量
cp .env.example .env
# 根据实际情况编辑 .env 文件

# 启动服务
docker compose up -d
```

docker-compose.yml 参考配置：

```yaml
version: '3.8'
services:
  nexuslink-community:
    image: nexuslink/community-server:latest
    ports:
      - "443:443"
      - "8080:8080"    # 管理接口
    volumes:
      - ./data:/data
      - ./certs:/certs
    environment:
      - NEXUSLINK_DOMAIN=your.community.com
      - NEXUSLINK_ADMIN_UUID=nxl:your_uuid_here
      - NEXUSLINK_MAX_MEMBERS=10000
      - NEXUSLINK_STORAGE_LIMIT=50GB
      - NEXUSLINK_DB_URL=postgres://nexuslink:password@postgres:5432/nexuslink
      - NEXUSLINK_REDIS_URL=redis://redis:6379       # 可选
      # 可选：注册到官方目录
      - NEXUSLINK_DIRECTORY_URL=https://directory.nexuslink.org
      - NEXUSLINK_DIRECTORY_TOKEN=your_registration_token
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=nexuslink
      - POSTGRES_USER=nexuslink
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

#### 从源码编译

```bash
# 克隆仓库
git clone https://github.com/yellowpeachxgp/NexusLink.git
cd NexusLink

# 编译社区服务器（Release 模式）
cargo build --release -p nexuslink-community-server

# 编译目录服务器（Release 模式）
cargo build --release -p nexuslink-directory-server

# 运行社区服务器
./target/release/nexuslink-community-server --config /path/to/config.toml

# 运行目录服务器
./target/release/nexuslink-directory-server --config /path/to/config.toml
```

#### 配置说明

服务端通过 TOML 格式的配置文件进行参数设置。示例配置：

```toml
# nexuslink-server.toml

[server]
# 服务器角色：community 或 directory
role = "community"
# 监听地址
bind_address = "0.0.0.0"
# HTTPS 端口
port = 443
# TLS 证书路径
tls_cert = "/certs/fullchain.pem"
tls_key = "/certs/privkey.pem"

[database]
# PostgreSQL 连接字符串
url = "postgres://nexuslink:password@localhost:5432/nexuslink"
# 连接池大小
max_connections = 20
# 连接超时（秒）
connect_timeout = 10

[redis]
# Redis 连接地址（可选，用于会话缓存和速率限制）
url = "redis://localhost:6379"
# 是否启用 Redis（关闭则使用内存缓存）
enabled = false

[community]
# 社区名称
name = "My Community"
# 管理员 UUID
admin_uuid = "nxl:your_uuid_here"
# 最大成员数
max_members = 10000
# 文件存储上限（字节）
storage_limit = 53687091200  # 50 GB
# 消息队列 TTL（秒）
message_ttl = 604800  # 7 天

[directory]
# 目录服务器 URL（注册用，可选）
url = "https://directory.nexuslink.org"
# 注册令牌
token = ""

[push]
# FCM 服务器密钥
fcm_key = ""
# APNs 配置
apns_cert = ""
apns_key_id = ""
apns_team_id = ""

[rate_limit]
# 每 IP 每秒最大请求数
requests_per_second = 50
# WebSocket 每秒最大消息数
ws_messages_per_second = 10
# 文件上传每分钟最大次数
upload_per_minute = 20

[logging]
# 日志级别：trace / debug / info / warn / error
level = "info"
# 日志格式：json / pretty
format = "json"
# 日志输出文件（空则输出到 stdout）
file = ""
```

### 服务端技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| Web 框架 | Axum | 基于 tower 生态的高性能异步 Web 框架，原生支持 WebSocket |
| 异步运行时 | tokio | Rust 异步运行时，提供多线程任务调度和 I/O 驱动 |
| 数据库驱动 | SQLx | 编译期检查的异步 SQL 工具包，支持 PostgreSQL 和 SQLite |
| 缓存 | Redis（可选） | 用于会话状态缓存、速率限制计数器和分布式锁 |
| 中间件 | tower | 提供超时、速率限制、日志、压缩等可组合的中间件层 |
| WebSocket | tokio-tungstenite | 基于 tokio 的异步 WebSocket 实现 |
| 序列化 | serde + Protocol Buffers | JSON（API 响应）+ Protobuf（消息传输） |
| TLS | rustls | 纯 Rust 实现的 TLS 库，无 OpenSSL 依赖 |
| 数据库迁移 | SQLx migrate | 内置的数据库迁移工具，支持版本化迁移脚本 |
| 配置管理 | config-rs | 支持多格式（TOML/YAML/ENV）的配置合并 |
| 日志 | tracing + tracing-subscriber | 结构化日志和分布式追踪 |
| 指标 | metrics + metrics-exporter-prometheus | Prometheus 兼容的指标导出 |

**服务端 Cargo 依赖示意：**

```toml
[dependencies]
# Web 框架
axum = { version = "0.7", features = ["ws", "multipart"] }
tower = { version = "0.4", features = ["full"] }
tower-http = { version = "0.5", features = ["cors", "compression-gzip", "trace", "timeout"] }

# 异步运行时
tokio = { version = "1", features = ["full"] }

# 数据库
sqlx = { version = "0.7", features = ["runtime-tokio", "tls-rustls", "postgres", "sqlite", "uuid", "chrono"] }

# 缓存（可选）
redis = { version = "0.24", features = ["tokio-comp"], optional = true }

# WebSocket
tokio-tungstenite = "0.21"

# TLS
rustls = "0.22"
axum-server = { version = "0.6", features = ["tls-rustls"] }

# 序列化
serde = { version = "1", features = ["derive"] }
serde_json = "1"
prost = "0.12"

# 配置
config = "0.14"

# 日志与追踪
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json", "env-filter"] }

# 指标
metrics = "0.22"
metrics-exporter-prometheus = "0.13"

# 工具
uuid = { version = "1", features = ["v7"] }
chrono = { version = "0.4", features = ["serde"] }
```

### 服务端监控与运维

#### 健康检查接口

```
GET /health

响应示例：
{
    "status": "healthy",
    "version": "0.3.0",
    "uptime_seconds": 86400,
    "database": "connected",
    "redis": "connected",            // 如果启用
    "websocket_connections": 342,
    "pending_messages": 1205
}
```

#### Prometheus 指标

服务器在 `/metrics` 端点暴露 Prometheus 格式的指标数据，包含：

- `nexuslink_ws_connections_active` -- 当前活跃 WebSocket 连接数
- `nexuslink_ws_messages_total` -- WebSocket 消息总数（按方向和类型分类）
- `nexuslink_http_requests_total` -- HTTP 请求总数（按路径、方法、状态码分类）
- `nexuslink_http_request_duration_seconds` -- HTTP 请求延迟分布
- `nexuslink_message_queue_size` -- 消息队列当前大小
- `nexuslink_message_queue_ttl_expired_total` -- 因 TTL 过期而清理的消息数
- `nexuslink_prekeys_available` -- 各用户可用预密钥数量
- `nexuslink_media_storage_bytes` -- 已使用的媒体存储空间
- `nexuslink_members_total` -- 社区成员总数
- `nexuslink_db_pool_connections` -- 数据库连接池状态

可配合 Grafana 仪表盘进行可视化监控。

#### 日志

服务端使用 `tracing` 框架进行结构化日志记录，支持 JSON 和人类可读两种输出格式。日志级别通过配置文件或环境变量 `RUST_LOG` 控制。

关键日志事件：
- 客户端连接/断开
- 认证成功/失败
- 消息投递状态
- 数据库操作异常
- 速率限制触发
- 审核操作记录

#### 备份策略

推荐的备份方案：

| 组件 | 备份方式 | 建议频率 |
|------|---------|---------|
| PostgreSQL | `pg_dump` 逻辑备份或 WAL 归档持续备份 | 每日全量 + 持续增量 |
| 媒体文件 | 文件系统快照或增量同步（rsync） | 每日增量 |
| 配置文件 | 版本控制（Git） | 每次变更时 |
| TLS 证书 | 安全备份至独立位置 | 每次续期时 |

---

## 技术栈

| 层级 | 技术 | 用途 |
|------|-----|------|
| **界面** | Flutter (Dart) | 跨平台原生 UI |
| **核心** | Rust | 密码学、网络、协议逻辑 |
| **桥接** | flutter_rust_bridge / FFI | 连接 Flutter 与 Rust |
| **身份** | nmshd/rust-crypto | 安全硬件密钥管理 |
| **加密** | Double Ratchet + X3DH | 端到端消息加密 |
| **P2P** | libp2p + WebRTC | 点对点直连 |
| **服务端框架** | Axum | 高性能异步 Web 框架 |
| **异步运行时** | tokio | 多线程异步任务调度 |
| **数据库驱动** | SQLx | 编译期检查的异步 SQL 工具包 |
| **服务端数据库** | PostgreSQL | 服务端持久化存储（推荐） |
| **客户端数据库** | SQLCipher | 客户端本地加密存储 |
| **缓存** | Redis（可选） | 会话缓存、速率限制、分布式锁 |
| **中间件** | tower / tower-http | 超时、限流、日志、压缩等可组合中间件 |
| **TLS** | rustls | 纯 Rust 实现的 TLS，无 OpenSSL 依赖 |
| **序列化** | Protocol Buffers / FlatBuffers | 传输格式 |
| **日志与追踪** | tracing | 结构化日志和分布式追踪 |
| **指标监控** | metrics + Prometheus | 服务端运行指标采集与导出 |
| **CI/CD** | GitHub Actions | 自动化测试与发布 |

---

## 快速开始

> **提示：** NexusLink 处于早期开发阶段。以下内容将在第 1 阶段完成后可用。

### 前置条件

- Rust 1.75+（通过 `rustup` 安装）
- Flutter 3.19+（通过 `flutter.dev` 安装）
- PostgreSQL 15+（服务端开发需要）
- Protobuf 编译器（`protoc`）
- Docker 和 Docker Compose（服务端部署推荐）

### 客户端开发

```bash
# 克隆仓库
git clone https://github.com/yellowpeachxgp/NexusLink.git
cd NexusLink

# 编译 Rust 核心库
cargo build --release -p nexuslink-core

# 运行 Rust 测试
cargo test -p nexuslink-core

# 进入 Flutter 项目目录
cd client/flutter

# 安装依赖
flutter pub get

# 运行应用（选择目标平台）
flutter run                    # 默认平台
flutter run -d chrome          # Web
flutter run -d macos           # macOS
flutter run -d ios             # iOS（需 Xcode）
flutter run -d android         # Android（需 Android SDK）

# 运行 Flutter 测试
flutter test
```

### 服务端开发

```bash
# 克隆仓库（如尚未克隆）
git clone https://github.com/yellowpeachxgp/NexusLink.git
cd NexusLink

# 方式一：使用 Docker 快速启动开发环境
cd server
docker compose -f docker-compose.dev.yml up -d

# 方式二：从源码编译和运行

# 确保 PostgreSQL 已运行并创建数据库
createdb nexuslink

# 编译社区服务器
cargo build --release -p nexuslink-community-server

# 编译目录服务器
cargo build --release -p nexuslink-directory-server

# 运行数据库迁移
cargo run -p nexuslink-community-server -- migrate

# 启动社区服务器（开发模式）
cargo run -p nexuslink-community-server -- --config server/community/config.dev.toml

# 启动目录服务器（开发模式）
cargo run -p nexuslink-directory-server -- --config server/directory/config.dev.toml

# 运行服务端测试
cargo test -p nexuslink-community-server
cargo test -p nexuslink-directory-server
cargo test -p nexuslink-server-shared

# 使用管理 CLI 工具
cargo run -p nexuslink-cli -- --help
cargo run -p nexuslink-cli -- server status
cargo run -p nexuslink-cli -- members list
```

---

## 项目结构

```
NexusLink/
|
|-- client/                            # 客户端应用
|   |-- flutter/                       # Flutter 跨平台 UI 层
|   |   |-- lib/
|   |   |   |-- screens/              # 页面视图（聊天、社区、设置、KYC 等）
|   |   |   |-- widgets/              # 可复用 UI 组件（消息气泡、频道列表等）
|   |   |   |-- services/             # 业务逻辑服务（网络请求、推送管理等）
|   |   |   |-- models/               # 数据模型（消息、用户、频道等）
|   |   |   |-- providers/            # 状态管理（会话状态、连接状态等）
|   |   |   |-- utils/                # UI 工具函数（格式化、验证等）
|   |   |   +-- l10n/                 # 国际化资源文件（中文、英文等）
|   |   |-- assets/                   # 静态资源
|   |   |   |-- icons/                # 应用图标
|   |   |   |-- images/               # 图片资源
|   |   |   +-- fonts/                # 字体文件
|   |   +-- test/                     # Widget 测试和集成测试
|   |
|   +-- core/                         # Rust 核心库（跨平台共享逻辑）
|       |-- src/
|       |   |-- identity/             # 身份系统：密钥生成、交叉签名、助记词恢复
|       |   |-- crypto/               # 加密引擎：Double Ratchet、X3DH、发送者密钥
|       |   |-- network/              # 网络层：libp2p 集成、WebRTC、NAT 穿透
|       |   |-- protocol/             # 协议层：消息编解码、Protobuf 序列化
|       |   |-- storage/              # 本地存储：SQLCipher 加密数据库操作
|       |   +-- ffi/                  # FFI 绑定：flutter_rust_bridge 接口定义
|       |-- tests/                    # Rust 集成测试
|       +-- benches/                  # 性能基准测试（加密吞吐量等）
|
|-- server/                            # 服务端应用
|   |-- community/                     # 社区服务器（可自托管）
|   |   |-- src/
|   |   |   |-- api/                  # 客户端 API 端点（REST + WebSocket 升级）
|   |   |   |-- chat/                 # 消息路由引擎：WebSocket 连接管理、消息扇出、
|   |   |   |                         #   离线消息队列、频道消息广播
|   |   |   |-- db/                   # 数据库操作层：连接池管理、查询构建、
|   |   |   |                         #   事务处理、迁移执行
|   |   |   |-- models/               # 数据模型定义：成员、频道、消息、设备、
|   |   |   |                         #   预密钥、媒体文件、审核日志
|   |   |   |-- services/             # 业务逻辑服务：认证服务、预密钥管理、
|   |   |   |                         #   推送通知分发、文件存储、成员管理
|   |   |   +-- federation/           # 目录注册模块：与目录服务器的通讯、
|   |   |                             #   社区信息同步、心跳上报
|   |   +-- migrations/               # 数据库迁移脚本（版本化 SQL 文件）
|   |
|   |-- directory/                     # 目录服务器（官方运营）
|   |   |-- src/
|   |   |   |-- api/                  # 目录 API 端点：社区注册/搜索/注销、
|   |   |   |                         #   KYC 认证接口、滥用举报接口
|   |   |   |-- auth/                 # 认证模块：KYC 流程管理、UUID 身份验证、
|   |   |   |                         #   工作量证明验证
|   |   |   |-- db/                   # 数据库操作层：社区索引、KYC 记录、
|   |   |   |                         #   举报记录、信誉评分
|   |   |   |-- models/               # 数据模型：社区注册信息、KYC 记录、
|   |   |   |                         #   滥用举报、审核队列
|   |   |   +-- services/             # 业务逻辑：社区搜索引擎、KYC 审核流程、
|   |   |                             #   反垃圾策略、社区健康探测
|   |   +-- migrations/               # 数据库迁移脚本
|   |
|   +-- shared/                        # 服务端共享代码库
|       +-- src/
|           |-- config/               # 统一配置管理：TOML 解析、环境变量覆盖、
|           |                         #   配置校验、默认值管理
|           |-- error/                # 统一错误类型定义、错误码规范、
|           |                         #   错误响应格式化
|           |-- types/                # 公共类型定义：UUID 类型、时间戳、
|           |                         #   分页参数、API 响应包装
|           +-- utils/                # 工具函数：密码学辅助、速率限制器、
|                                     #   健康检查逻辑、Prometheus 指标注册
|
|-- protocol/                          # 协议规范
|   |-- specs/                        # 人类可读的协议文档（Markdown 格式）
|   +-- schemas/                      # Protobuf / FlatBuffers 结构定义文件
|
|-- docs/                              # 项目文档
|   |-- cn/                           # 中文文档
|   |-- en/                           # 英文文档
|   |-- diagrams/                     # 架构图和流程图
|   +-- api/                          # API 参考文档
|
|-- tools/                             # 开发者工具
|   |-- cli/                          # nexuslink-cli 管理命令行工具
|   |   +-- src/                      # CLI 源码：服务器管理、成员管理、
|   |                                 #   数据库操作、诊断工具
|   +-- scripts/                      # 构建、部署、测试脚本
|
|-- .github/                           # GitHub 配置
|   |-- workflows/                    # CI/CD 流水线（构建、测试、发布）
|   +-- ISSUE_TEMPLATE/               # Issue 模板
|
|-- README.md                          # 项目主文档（中文，本文件）
|-- README_EN.md                       # 英文版 README
|-- ARCHITECTURE.md                    # 架构深度解析
|-- ROADMAP.md                         # 开发路线图
|-- CONTRIBUTING.md                    # 贡献指南
+-- LICENSE                            # AGPLv3 许可证
```

---

## 与现有方案的对比

| 特性 | NexusLink | Matrix | Telegram | Signal | Session |
|------|-----------|--------|----------|--------|---------|
| 无需注册 | 是（硬件密钥） | 否 | 否（手机号） | 否（手机号） | 是（密钥） |
| 默认端到端加密 | 是 | 可选 | 否 | 是 | 是 |
| 自建社区服务器 | 是 | 是（联邦） | 否 | 否 | 否 |
| P2P 直连模式 | 是 | 计划中 | 否 | 否 | 是（洋葱路由） |
| 官方目录 | 是 | matrix.org | Telegram Inc. | 无 | 无 |
| 隐私隔离 | 是（架构级） | 否（元数据泄露） | 否 | 无 | 部分 |
| 实名认证（可选） | 是（仅官方） | 否 | 否 | 否 | 否 |
| 多设备同步 | 是 | 是 | 是 | 是 | 是 |
| 身份恢复 | 助记词 | 密码 | 手机/云 | 手机/PIN | 助记词 |

---

## 安全模型

### 威胁模型

| 威胁 | 缓解措施 |
|------|---------|
| 服务器被入侵 | 端到端加密 -- 服务器永远没有明文消息 |
| 身份盗用 | 私钥存储在安全硬件中 -- 无法导出 |
| 元数据监控（官方） | 目录服务器无权访问通讯数据 |
| 元数据监控（社区） | 社区服务器可见元数据但不可见内容；用户选择信任哪个服务器 |
| 设备丢失 | 助记词恢复短语可在新设备恢复身份 |
| 中间人攻击 | 通过二维码/安全号码验证公钥 |

### NexusLink 不能防护的场景
- 被入侵的社区服务器可以看到通讯元数据（谁和谁在什么时候通讯）-- 这是为易用性做出的有意权衡
- 如果用户设备被入侵（恶意软件、root 权限），攻击者可以访问解密后的消息
- P2P 模式需要 STUN 服务器进行 NAT 穿透；这些服务器可以看到 IP 地址（但看不到消息内容）

---

## 开发路线图

详见 [ROADMAP.md](./ROADMAP.md)。

| 阶段 | 重点 | 状态 |
|------|------|------|
| 第 0 阶段 | 项目搭建、CI/CD、协议规范 | 计划中 |
| 第 1 阶段 | 身份系统、端到端加密核心 | 计划中 |
| 第 2 阶段 | P2P 消息传递（模式 C） | 计划中 |
| 第 3 阶段 | 社区服务器（模式 B） | 计划中 |
| 第 4 阶段 | 目录服务器 + 实名认证（模式 A） | 计划中 |
| 第 5 阶段 | 全平台发布与打磨 | 计划中 |
| 第 6 阶段 | 高级功能（语音、文件、ZKP） | 计划中 |

---

## 参与贡献

详见 [CONTRIBUTING.md](./CONTRIBUTING.md)。

欢迎以下方面的贡献：
- 协议设计与安全审查
- Rust 核心库开发
- Flutter UI/UX 设计与实现
- 服务端开发
- 文档编写与翻译
- 安全审计

---

## 许可证

NexusLink 采用 [GNU Affero 通用公共许可证 v3.0](./LICENSE) 授权。

这意味着：
- 你可以自由使用、修改和分发 NexusLink
- 任何修改必须同样以 AGPLv3 开源
- 如果你将修改版本作为网络服务运行，必须向用户提供源代码
- 这确保 NexusLink 始终对所有人保持开放和免费

---

## 致谢

NexusLink 建立在众多开源项目和协议的研究成果之上：

- [Signal 协议](https://signal.org/docs/) -- Double Ratchet 和 X3DH 密钥协商
- [Matrix](https://matrix.org/) -- 联邦消息传递和交叉签名概念
- [libp2p](https://libp2p.io/) -- 点对点网络协议栈
- [nmshd/rust-crypto](https://github.com/nmshd/rust-crypto) -- 跨平台安全硬件抽象
- [Session](https://getsession.org/) -- 无需注册的去中心化身份
- [Axum](https://github.com/tokio-rs/axum) -- 高性能异步 Web 框架
- [tokio](https://tokio.rs/) -- Rust 异步运行时

---

<p align="center">
  <strong>你的消息。你的身份。你的选择。</strong>
</p>
