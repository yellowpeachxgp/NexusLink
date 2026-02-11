<p align="center">
  <h1 align="center">NexusLink</h1>
  <p align="center">
    <strong>Hybrid Messaging Platform — Where Centralized Meets Decentralized</strong>
  </p>
  <p align="center">
    <a href="./README_CN.md">中文文档</a> |
    <a href="./docs/en/architecture.md">Architecture</a> |
    <a href="./ROADMAP.md">Roadmap</a> |
    <a href="./CONTRIBUTING.md">Contributing</a>
  </p>
</p>

---

## What is NexusLink?

NexusLink is an open-source, cross-platform instant messaging application that uniquely **bridges centralized and decentralized communication** within a single app. Users choose how they connect — through an official directory server, a self-hosted community server, or direct peer-to-peer — without sacrificing usability or security.

**No phone number. No email. No registration.** Your identity is a cryptographic key pair generated on your device's secure hardware. You own your identity.

### The Problem

| Centralized Apps (WeChat, Telegram, Discord) | Decentralized Apps (Signal, Briar, Session) |
|---|---|
| Easy to use, feature-rich | Privacy-first, censorship-resistant |
| You surrender data sovereignty | Limited social/community features |
| Single point of failure & surveillance | Fragmented user experience |
| No choice in who hosts your data | Technical barrier for average users |

**NexusLink fills the gap** — offering the community features of centralized platforms with the privacy and sovereignty of decentralized ones, letting each user decide their own trust level.

---

## Key Features

### Identity Without Registration
- Cryptographic key pair generated in device secure hardware (TPM / Secure Enclave / StrongBox)
- Your public key **is** your UUID — no server involved in identity creation
- Multi-device sync via cross-signing protocol
- Identity migration via mnemonic recovery phrase (12-24 words)

### Three Connection Modes

```
┌─────────────────────────────────────────────────────────────┐
│                    NexusLink Client                         │
│                                                             │
│  [A] Official Server     [B] Community Server    [C] P2P   │
│  - Real-name verified    - Self-hosted            - Direct  │
│  - Browse communities    - Data sovereignty       - No server│
│  - Regulated channels    - Custom rules           - E2E only │
└─────────────────────────────────────────────────────────────┘
```

**Mode A: Official Directory Server**
- Real-name (KYC) verified UUID required
- Browse and join registered communities
- Access official public channels
- Anti-abuse protections

**Mode B: Community Self-Hosted Server**
- Anyone can deploy with the same server codebase
- Optionally register in the official directory for discoverability
- Full data sovereignty — the official server never sees your messages, members, or metadata
- Community-defined moderation rules

**Mode C: Peer-to-Peer**
- Direct device-to-device communication via libp2p + WebRTC
- Exchange public keys via QR code or out-of-band
- Zero server dependency (STUN for NAT traversal only)
- Maximum privacy for sensitive conversations

### Privacy by Architecture
- End-to-end encryption (Double Ratchet protocol) for all messages
- Official directory server is a **phonebook only** — stores community listings, not messages or membership
- Community servers are **privacy shields** — metadata never flows to the official server
- Zero-knowledge proof ready for future KYC (prove identity without revealing details)

### Cross-Platform
- iOS, Android, Windows, macOS, Linux
- Flutter UI + Rust core via FFI
- Consistent experience across all platforms

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                         Client                               │
│  ┌────────────────────┐  ┌─────────────────────────────────┐ │
│  │   Flutter UI Layer │  │        Rust Core Library        │ │
│  │                    │◄─┤  - Identity (Secure HW)         │ │
│  │  - Chat screens    │  │  - E2E Encryption               │ │
│  │  - Community view  │  │  - P2P Network (libp2p)         │ │
│  │  - Settings        │  │  - Protocol codec               │ │
│  │  - Multi-language  │  │  - Local encrypted storage      │ │
│  └────────────────────┘  └─────────────────────────────────┘ │
└──────────────────┬───────────────────┬───────────────────────┘
                   │                   │
         ┌─────────┘                   └──────────┐
         ▼                                        ▼
┌─────────────────────┐               ┌───────────────────────┐
│  Directory Server   │               │  Community Server     │
│  (Official)         │               │  (Self-hosted)        │
│                     │               │                       │
│  - Community index  │   register    │  - Chat hosting       │
│  - KYC service      │◄─────────────┤  - Member management  │
│  - Abuse reporting  │   (metadata   │  - File storage       │
│                     │    only)      │  - Custom moderation  │
└─────────────────────┘               └───────────────────────┘
                                               │
                                      P2P direct connection
                                               │
                                      ┌────────▼──────────┐
                                      │   Another Client   │
                                      └───────────────────┘
```

For detailed architecture documentation, see [ARCHITECTURE.md](./docs/en/architecture.md).

---

## Project Structure

```
NexusLink/
├── client/                     # Client applications
│   ├── flutter/                # Flutter UI (cross-platform)
│   │   ├── lib/
│   │   │   ├── screens/        # Page screens
│   │   │   ├── widgets/        # Reusable UI components
│   │   │   ├── services/       # Business logic services
│   │   │   ├── models/         # Data models
│   │   │   ├── providers/      # State management
│   │   │   ├── utils/          # UI utilities
│   │   │   └── l10n/           # Internationalization (i18n)
│   │   ├── assets/             # Icons, images, fonts
│   │   └── test/               # Widget & integration tests
│   │
│   └── core/                   # Rust core library (shared logic)
│       ├── src/
│       │   ├── identity/       # Key generation, cross-signing, recovery
│       │   ├── crypto/         # E2E encryption, Double Ratchet, key exchange
│       │   ├── network/        # libp2p, WebRTC, NAT traversal
│       │   ├── protocol/       # Message encoding/decoding, wire format
│       │   ├── storage/        # Local encrypted database (SQLCipher)
│       │   └── ffi/            # FFI bindings for Flutter
│       ├── tests/              # Integration tests
│       └── benches/            # Performance benchmarks
│
├── server/                     # Server applications
│   ├── directory/              # Official directory server
│   │   ├── src/
│   │   │   ├── api/            # REST/gRPC endpoints
│   │   │   ├── auth/           # KYC verification, UUID auth
│   │   │   ├── db/             # Database operations
│   │   │   ├── models/         # Data models
│   │   │   └── services/       # Business logic
│   │   └── migrations/         # Database migrations
│   │
│   ├── community/              # Community server (self-hostable)
│   │   ├── src/
│   │   │   ├── api/            # Client-facing API
│   │   │   ├── chat/           # Message routing, rooms, channels
│   │   │   ├── db/             # Database operations
│   │   │   ├── models/         # Data models
│   │   │   ├── services/       # Business logic
│   │   │   └── federation/     # Directory server registration
│   │   └── migrations/         # Database migrations
│   │
│   └── shared/                 # Shared server code
│       └── src/
│           ├── config/         # Configuration management
│           ├── error/          # Error types
│           ├── types/          # Shared type definitions
│           └── utils/          # Common utilities
│
├── protocol/                   # Protocol specifications
│   ├── specs/                  # Human-readable protocol docs
│   └── schemas/                # Protobuf / FlatBuffers schemas
│
├── docs/                       # Documentation
│   ├── en/                     # English docs
│   ├── cn/                     # Chinese docs
│   ├── diagrams/               # Architecture diagrams
│   └── api/                    # API reference
│
├── tools/                      # Developer tools
│   ├── cli/                    # CLI management tool
│   │   └── src/
│   └── scripts/                # Build, deploy, test scripts
│
├── .github/                    # GitHub configuration
│   ├── workflows/              # CI/CD pipelines
│   └── ISSUE_TEMPLATE/         # Issue templates
│
├── README.md                   # This file (English)
├── README_CN.md                # Chinese README
├── ROADMAP.md                  # Development roadmap
├── ARCHITECTURE.md             # Architecture deep-dive
├── CONTRIBUTING.md             # Contribution guide
├── LICENSE                     # AGPLv3
└── Cargo.toml                  # Rust workspace root
```

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
| **Server** | Rust (Axum / Actix-web) | HTTP/gRPC server framework |
| **Database** | PostgreSQL (server), SQLCipher (client) | Persistent storage |
| **Serialization** | Protocol Buffers / FlatBuffers | Wire format |
| **CI/CD** | GitHub Actions | Automated testing & releases |

---

## Quick Start

> **Note:** NexusLink is in early development. The following will be available once Phase 1 is complete.

### Prerequisites
- Rust 1.75+ (with `cargo`)
- Flutter 3.19+ (with `dart`)
- PostgreSQL 15+ (for server)
- Protobuf compiler (`protoc`)

### Build from Source

```bash
# Clone the repository
git clone https://github.com/user/nexuslink.git
cd nexuslink

# Build the Rust core library
cargo build --release -p nexuslink-core

# Build the community server
cargo build --release -p nexuslink-community-server

# Build the Flutter client
cd client/flutter
flutter pub get
flutter run
```

### Deploy a Community Server

```bash
# Using Docker (recommended)
docker pull nexuslink/community-server:latest
docker run -d \
  -p 443:443 \
  -v /path/to/data:/data \
  -e NEXUSLINK_DOMAIN=your.domain.com \
  nexuslink/community-server:latest

# Or build from source
cargo run --release -p nexuslink-community-server -- \
  --config /path/to/config.toml
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
| Phase 5 | Cross-platform release & polish | Planned |
| Phase 6 | Advanced features (voice, file, ZKP) | Planned |

---

## Comparison with Existing Solutions

| Feature | NexusLink | Matrix | Telegram | Signal | Session |
|---------|-----------|--------|----------|--------|---------|
| No registration required | Yes (HW key) | No | No (phone) | No (phone) | Yes (key) |
| E2E encryption default | Yes | Optional | No | Yes | Yes |
| Self-hosted community | Yes | Yes (federation) | No | No | No |
| P2P direct mode | Yes | Planned | No | No | Yes (onion) |
| Official directory | Yes | matrix.org | Telegram Inc. | N/A | N/A |
| Privacy isolation | Yes (by design) | No (metadata leaks) | No | N/A | Partial |
| Real-name gate (optional) | Yes (for official) | No | No | No | No |
| Multi-device sync | Yes | Yes | Yes | Yes | Yes |
| Identity recovery | Mnemonic phrase | Password | Phone/cloud | Phone/PIN | Mnemonic |

---

## Security Model

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| Server compromise | E2E encryption — server never has plaintext messages |
| Identity theft | Private key in secure hardware — cannot be exported |
| Metadata surveillance (official) | Directory server has no access to communication data |
| Metadata surveillance (community) | Community server sees metadata but not content; users choose which server to trust |
| Device loss | Mnemonic recovery phrase restores identity on new device |
| Man-in-the-middle | Public key verification via QR code / safety numbers |

### What NexusLink Does NOT Protect Against
- A compromised community server can see communication metadata (who talks to whom, when) — this is a conscious trade-off for usability
- If a user's device is compromised (malware, root access), the attacker has access to decrypted messages
- P2P mode requires STUN servers for NAT traversal; these servers can see IP addresses (but not message content)

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for detailed guidelines.

We welcome contributions in all areas:
- Protocol design and security review
- Rust core library development
- Flutter UI/UX design and implementation
- Server development
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

NexusLink builds upon the research and work of many open-source projects and protocols:

- [Signal Protocol](https://signal.org/docs/) — Double Ratchet and X3DH key agreement
- [Matrix](https://matrix.org/) — Federated messaging and cross-signing concepts
- [libp2p](https://libp2p.io/) — Peer-to-peer networking stack
- [nmshd/rust-crypto](https://github.com/nmshd/rust-crypto) — Cross-platform secure hardware abstraction
- [Session](https://getsession.org/) — Decentralized identity without registration

---

<p align="center">
  <strong>Your messages. Your identity. Your choice.</strong>
</p>
