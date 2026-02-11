# Contributing to NexusLink / 参与贡献

Thank you for your interest in contributing to NexusLink! This document provides guidelines for contributing.

感谢你对 NexusLink 贡献的兴趣！本文档提供贡献指南。

---

## Code of Conduct / 行为准则

- Be respectful and constructive / 尊重他人，建设性交流
- Focus on the technical merits / 关注技术价值
- No harassment, discrimination, or personal attacks / 禁止骚扰、歧视或人身攻击

## How to Contribute / 如何贡献

### Report Bugs / 报告 Bug

1. Search existing issues to avoid duplicates / 搜索已有 issue 避免重复
2. Open a new issue with the `bug` label / 使用 `bug` 标签创建新 issue
3. Include: platform, version, steps to reproduce, expected vs actual behavior / 包含：平台、版本、复现步骤、预期与实际行为

### Suggest Features / 建议功能

1. Open an issue with the `feature-request` label / 使用 `feature-request` 标签创建 issue
2. Describe the use case, not just the solution / 描述使用场景，而不仅是解决方案

### Submit Code / 提交代码

1. Fork the repository / Fork 仓库
2. Create a feature branch: `git checkout -b feature/my-feature` / 创建功能分支
3. Write code with tests / 编写代码和测试
4. Ensure CI passes: `cargo test && cd client/flutter && flutter test` / 确保 CI 通过
5. Submit a Pull Request with a clear description / 提交 PR 并附清晰描述

### Code Style / 代码风格

**Rust:**
- Follow `rustfmt` defaults / 遵循 rustfmt 默认配置
- Run `cargo clippy` with no warnings / 运行 clippy 无警告
- Write doc comments for public APIs / 为公开 API 编写文档注释

**Dart (Flutter):**
- Follow `dart format` defaults / 遵循 dart format 默认配置
- Run `dart analyze` with no issues / 运行 analyze 无问题

**Commits:**
- Use conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`
- Keep commits atomic and focused / 保持提交原子化和聚焦

### Security Vulnerabilities / 安全漏洞

**Do NOT report security vulnerabilities via public GitHub issues.**

**不要通过公开的 GitHub issue 报告安全漏洞。**

Instead, please email: security@nexuslink.org (to be set up)

请发邮件至：security@nexuslink.org（待设立）

## Development Setup / 开发环境搭建

```bash
# Prerequisites / 前置条件
# - Rust 1.75+ (rustup.rs)
# - Flutter 3.19+ (flutter.dev)
# - PostgreSQL 15+ (for server development)
# - Protobuf compiler (protoc)

# Clone / 克隆
git clone https://github.com/user/nexuslink.git
cd nexuslink

# Build Rust core / 构建 Rust 核心
cargo build

# Run Rust tests / 运行 Rust 测试
cargo test

# Build Flutter app / 构建 Flutter 应用
cd client/flutter
flutter pub get
flutter run

# Run Flutter tests / 运行 Flutter 测试
flutter test
```

## License / 许可证

By contributing, you agree that your contributions will be licensed under AGPLv3.

参与贡献即表示你同意你的贡献将在 AGPLv3 下授权。
