# SPFsmartGATE Documentation

Start broad, go deep. Each layer adds detail.

```mermaid
graph TD
    README["📖 README.md<br/>What is this? Do I need it?"] --> L1
    L1["Layer 1<br/>Who, how, compared to what"] --> L2
    L2["Layer 2<br/>Design philosophy"] --> L3
    L3["Layer 3<br/>Implementation details"]

    README -.-> TM["threat-model.md"]
    README -.-> HW["how-it-works.md"]
    README -.-> CMP["comparison.md"]
    L1 -.-> WHY["why-spf.md"]
    L1 -.-> BIBLE["developer-bible.md"]
    L2 -.-> TOOLS["mcp-tools.md"]
    L2 -.-> HOOKS["hooks.md"]
    L2 -.-> DEPLOY["deployment.md"]

    style README fill:#E74C3C,stroke:#C0392B,color:#fff
    style L1 fill:#F39C12,stroke:#E67E22,color:#fff
    style L2 fill:#3498DB,stroke:#2980B9,color:#fff
    style L3 fill:#27AE60,stroke:#219A52,color:#fff
    style TM fill:#F39C12,stroke:#E67E22,color:#fff
    style HW fill:#F39C12,stroke:#E67E22,color:#fff
    style CMP fill:#F39C12,stroke:#E67E22,color:#fff
    style WHY fill:#3498DB,stroke:#2980B9,color:#fff
    style BIBLE fill:#3498DB,stroke:#2980B9,color:#fff
    style TOOLS fill:#27AE60,stroke:#219A52,color:#fff
    style HOOKS fill:#27AE60,stroke:#219A52,color:#fff
    style DEPLOY fill:#27AE60,stroke:#219A52,color:#fff
```

## Layer 1 — Why and who

| Document | Description |
|----------|-------------|
| [Who needs this?](threat-model.md) | Risk by model type, real use cases, device-by-device platform guide |
| [How it works](how-it-works.md) | 5-stage pipeline, architecture, blocked examples, BLOCKED feedback loop |
| [How it compares](comparison.md) | Claude Code overlap, MCP gateways, sandboxing, memory frameworks, honest gaps |

## Layer 2 — Design and philosophy

| Document | Description |
|----------|-------------|
| [Why SPFsmartGATE?](why-spf.md) | Feature highlights, value proposition, architectural advantages |
| [Developer Bible](developer-bible.md) | Complete technical reference covering all 13 system blocks |

## Layer 3 — Implementation details

| Document | Description |
|----------|-------------|
| [MCP Tools Reference](mcp-tools.md) | All 55 exposed tools with parameters, LMDB routing, and handler details |
| [Hook System](hooks.md) | Dual-layer hook architecture — 31 scripts for monitoring and enforcement |
| [Deployment Guide](deployment.md) | Build system, deployment scripts, config.json structure, LIVE directory layout |

## Screenshots

Screenshots from the Android/Termux deployment are in [screenshots/](screenshots/).

## Quick links

- [Main README](../README.md) — Project overview and quick start
- [Security Policy](../SECURITY.md) — Vulnerability reporting
- [Changelog](../CHANGELOG.md) — Version history
- [License](../LICENSE.md) — PolyForm Noncommercial 1.0.0
- [Commercial License](../COMMERCIAL_LICENSE.md) — Business use terms
