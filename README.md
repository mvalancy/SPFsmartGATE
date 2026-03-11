<p align="center">
  <img src="docs/screenshots/SPFsmartGATE.png" alt="SPFsmartGATE Logo" width="400"/>
</p>

<h1 align="center">SPFsmartGATE</h1>

<p align="center">
  <strong>A compiled security gate for AI agents. Built in Rust. Can't be talked out of doing its job.</strong>
</p>

<p align="center">
  <a href="LICENSE.md"><img src="https://img.shields.io/badge/License-PolyForm%20NC%201.0.0-blue.svg" alt="License"/></a>
  <a href="CHANGELOG.md"><img src="https://img.shields.io/badge/version-2.0.0-green.svg" alt="Version"/></a>
  <a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/built%20with-Rust-orange.svg" alt="Rust"/></a>
</p>

---

## What is this?

A compiled Rust binary that sits between an AI agent and your system. Every tool call — file reads, writes, bash commands, web requests — passes through a 5-stage security pipeline before execution. The rules are in the compiled code, not in a prompt. No amount of clever text can override them.

```mermaid
graph LR
    A["🤖 AI Agent"] -->|Every tool call| B{"🛡️ SPFsmartGATE"}
    B -->|"Safe ✅"| C["💻 Your System"]
    B -->|"Dangerous 🚫"| D["❌ Blocked + reason"]

    style A fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style B fill:#F39C12,stroke:#E67E22,color:#fff
    style C fill:#27AE60,stroke:#219A52,color:#fff
    style D fill:#E74C3C,stroke:#C0392B,color:#fff
```

When something is blocked, the AI gets a specific error (`BLOCKED | tool | reason`) so it can change direction — not just fail silently.

---

## Do you actually need this?

> **If you have a desktop or laptop with 16GB+ RAM: you probably don't.** Use Docker, a VM, or container sandboxing — it's simpler, more robust, and works with any model. SPFsmartGATE exists for environments where those options aren't available.

### The simpler alternatives you should try first

If you have a real computer (desktop, laptop, workstation, dev server), you have better options:

| Approach | What it does | When to use it |
|---|---|---|
| **Docker sandbox** | Runs the AI in an isolated container. If it breaks everything inside, your host is untouched. | Any desktop/laptop. Official Claude Code support. The simplest answer. |
| **Virtual machine** | Full OS isolation. The AI gets its own filesystem, network, and kernel. Even more isolated than Docker. | When you want total isolation, or you're running untrusted models you really don't trust. Heavier but bulletproof. |
| **Firecracker / microVM** | Lightweight VMs with ~80ms cold start (E2B uses this). VM-level isolation, container-level speed. | Cloud/server deployments, multi-tenant agent hosting. |
| **Claude Code deny rules + hooks** | Built-in path blocking, tool approval prompts, deterministic shell script hooks. | Already using Claude Code with frontier models. ~60-70% of SPFsmartGATE's security coverage with zero extra software. |

**If you have 32-128GB RAM, a GPU, and Docker installed — just use Docker.** It's the right tool. SPFsmartGATE is solving a different problem.

### Where SPFsmartGATE actually fits

The niche is narrow but real: **Android phones and tablets running local models through MCP.**

```mermaid
graph TD
    Q{"Where are you<br/>running AI agents?"} -->|"Desktop / laptop / server<br/>16GB+ RAM"| DOCKER["Use Docker or a VM<br/>You don't need SPFsmartGATE"]
    Q -->|"Android phone / tablet<br/>via Termux"| SPF["SPFsmartGATE is<br/>one of your few options"]
    Q -->|"Very low-RAM SBC<br/>1-2GB Pi, no Docker"| MAYBE["SPFsmartGATE helps<br/>if Docker won't fit"]

    style Q fill:#F39C12,stroke:#E67E22,color:#fff
    style DOCKER fill:#3498DB,stroke:#2980B9,color:#fff
    style SPF fill:#27AE60,stroke:#219A52,color:#fff
    style MAYBE fill:#F39C12,stroke:#E67E22,color:#fff
```

- **Android/Termux** — Docker doesn't work reliably here. If you're running Ollama or llama.cpp on your phone with MCP tool access, a compiled security gate is one of the few ways to prevent a 3B model from `rm -rf`-ing your storage. This is the primary use case.
- **Very low-RAM SBCs** (1-2GB Raspberry Pi) — Docker might not fit alongside the model. A 5MB compiled binary is lighter than a container runtime.
- **Untrusted MCP agents you want to try** — random agent frameworks from GitHub. The gate lets you give them tool access with guardrails, even on a powerful machine, if you don't want to bother setting up a container for a quick test.

**It does NOT matter what model you're running if you can just put it in a container.** A Docker sandbox protects against a misbehaving GPT-4, Llama 70B, or abliterated 3B model equally — by isolating the entire environment. SPFsmartGATE's per-tool-call filtering is a weaker isolation model that only makes sense when containers aren't an option.

**Scope limitation:** SPFsmartGATE only gates **MCP tool calls** (JSON-RPC over stdio). If your architecture has the model generate code (Lua, Python) that a separate runtime executes, the gate never sees those actions — you need to sandbox the runtime instead. See [threat model](docs/threat-model.md) for details.

Read more: **[Who needs this?](docs/threat-model.md)** — detailed use cases, platform-by-platform guide

---

## What can I actually use this with?

SPFsmartGATE has two layers, and they work with different things:

```mermaid
graph TD
    RUST["🛡️ Rust Binary<br/>Standard MCP server<br/>Works with any MCP client"] --- HOOKS["📜 Hook Scripts<br/>31 shell scripts<br/>Claude Code only"]

    RUST -->|"Full support"| CC["Claude Code"]
    RUST -->|"Binary works,<br/>no hooks"| CURSOR["Cursor, Windsurf,<br/>Zed, other MCP clients"]
    RUST -->|"Binary works,<br/>no hooks"| CUSTOM["Custom agents<br/>Python, Node, etc."]
    HOOKS -->|"Blocks native tools,<br/>forces MCP route"| CC

    RUST -.->|"Doesn't apply"| NONMCP["Non-MCP agents<br/>Lua scripting, REST APIs"]

    style RUST fill:#F39C12,stroke:#E67E22,color:#fff
    style HOOKS fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style CC fill:#27AE60,stroke:#219A52,color:#fff
    style CURSOR fill:#3498DB,stroke:#2980B9,color:#fff
    style CUSTOM fill:#3498DB,stroke:#2980B9,color:#fff
    style NONMCP fill:#95A5A6,stroke:#7F8C8D,color:#fff
```

| Setup | What works | What doesn't |
|---|---|---|
| **Claude Code** (full setup) | Everything. Hooks block native tools (Read, Write, Bash, etc.) and force them through the Rust gate. The AI can't bypass security by using native tools instead of `spf_*` tools. | — |
| **Other MCP clients** (Cursor, Windsurf, custom agents) | The Rust binary works as a standard MCP server — any client that spawns a subprocess and talks JSON-RPC over stdio can connect. All 55 gated tools, all security checks, all LMDB tracking. | The hook scripts don't work. These clients have their own native tools, and there's nothing forcing them to use `spf_read` instead of their own `Read`. You'd need to configure the client to only use SPF tools. |
| **Non-MCP agents** (Lua scripting, REST-based, code generation) | Nothing. If the agent doesn't talk MCP, the gate never sees the actions. | Everything. Security needs to be in the executor layer instead. |

**Bottom line:** Full plug-and-play with Claude Code. Works as an MCP server with other clients but needs manual configuration. Irrelevant if MCP isn't in the picture.

Read more: **[Who needs this?](docs/threat-model.md)** — includes code-generation vs. MCP architecture comparison

---

## Quick start

```bash
git clone https://github.com/STONE-CELL-SPF-JOSEPH-STONE/SPFsmartGATE.git
cd SPFsmartGATE
bash setup.sh
./target/release/spf-smart-gate serve
```

> **Need Rust?** Install from [rustup.rs](https://rustup.rs/) first.

---

## Project structure

```
SPFsmartGATE/
├── src/                 # Rust source (15 modules, ~7,800 lines)
├── hooks/               # 31 shell scripts for monitoring
├── scripts/             # Setup and boot scripts
├── LIVE/                # Runtime data (databases, binaries)
├── docs/                # Technical documentation
├── config.json          # Runtime configuration
├── setup.sh             # One-command installer
└── build.sh             # Cross-platform build script
```

---

## Documentation

Start here and go deeper:

| Document | What you'll learn |
|----------|-------------------|
| **[Who needs this?](docs/threat-model.md)** | Risk by model type, real use cases, device-by-device platform guide |
| **[How it works](docs/how-it-works.md)** | 5-stage pipeline, architecture, blocked examples, the BLOCKED feedback loop |
| **[How it compares](docs/comparison.md)** | Claude Code overlap, MCP gateways, sandboxing, memory frameworks, honest gaps |
| [Why SPFsmartGATE?](docs/why-spf.md) | Feature highlights and design philosophy |
| [Developer Bible](docs/developer-bible.md) | Complete technical reference — every module, every feature |
| [MCP Tools Reference](docs/mcp-tools.md) | All 55 tools with parameters and examples |
| [Hook System](docs/hooks.md) | 31 monitoring scripts explained |
| [Deployment Guide](docs/deployment.md) | Building, installing, configuring |
| [Security Policy](SECURITY.md) | Vulnerability reporting |
| [Changelog](CHANGELOG.md) | Version history |

---

## License

**Free for personal use.** Commercial use requires a paid license.

Licensed under the [PolyForm Noncommercial License 1.0.0](LICENSE.md).
See [COMMERCIAL_LICENSE.md](COMMERCIAL_LICENSE.md) for business use, or email **joepcstone@gmail.com**.

---

<p align="center">
  Copyright 2026 Joseph Stone. All Rights Reserved.<br/>
  <em>SPFsmartGATE and the StoneCell Processing Formula (SPF) are proprietary intellectual property.</em>
</p>
