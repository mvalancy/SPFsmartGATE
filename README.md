<p align="center">
  <img src="docs/screenshots/SPFsmartGATE.png" alt="SPFsmartGATE Logo" width="400"/>
</p>

<h1 align="center">SPFsmartGATE</h1>

<p align="center">
  <strong>A security guard for your AI coding assistant. Built in Rust. Can't be talked out of doing its job.</strong>
</p>

<p align="center">
  <a href="LICENSE.md"><img src="https://img.shields.io/badge/License-PolyForm%20NC%201.0.0-blue.svg" alt="License"/></a>
  <a href="CHANGELOG.md"><img src="https://img.shields.io/badge/version-2.0.0-green.svg" alt="Version"/></a>
  <a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/built%20with-Rust-orange.svg" alt="Rust"/></a>
</p>

---

## The problem

You're using an AI assistant like Claude Code to help you write code. It can read files, write code, run terminal commands, and search the web. Most of the time it works great. But sometimes:

- It edits a file it never actually read and **introduces bugs into working code**
- It accidentally puts a secret or API key into content that shouldn't have it
- It writes to a path outside your project without you noticing
- It makes a long chain of rapid changes that are hard to review or undo

Tools like Claude Code already have approval prompts, which help. But those are still **you** being the security guard — reading every action, deciding yes or no, hoping you catch the risky ones. SPFsmartGATE automates that layer so the obviously-dangerous stuff never even reaches you.

```mermaid
graph LR
    A["🤖 AI Agent"] -->|Every action| B{"🛡️ SPFsmartGATE"}
    B -->|"Safe ✅"| C["💻 Your Computer"]
    B -->|"Dangerous 🚫"| D["❌ Blocked"]

    style A fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style B fill:#F39C12,stroke:#E67E22,color:#fff
    style C fill:#27AE60,stroke:#219A52,color:#fff
    style D fill:#E74C3C,stroke:#C0392B,color:#fff
```

## The solution

**SPFsmartGATE is a compiled Rust program that sits between the AI and your computer.** Every tool call — every file read, every write, every bash command, every web request — has to pass through the gate first. If it's dangerous, the gate blocks it. Period.

The key difference from prompt-based safety: **the AI can't override the binary.** System prompts are text — in theory, a prompt injection could trick the AI into ignoring them. SPFsmartGATE's Rust binary enforces rules at the code level. The AI calls tools, the binary checks them, end of story.

> **Important caveat:** The full security model also depends on Claude Code hook scripts being configured correctly (the setup script handles this). The hooks redirect native tools through the gate. If hooks are misconfigured, native tools could bypass the gate. The Rust binary itself is solid — but it's one layer in a defense-in-depth system, not a magic bullet.

```mermaid
graph TB
    subgraph "❌ Text-based safety"
        P["📝 System prompt:<br/>'Don't delete files'"] --> AI1["🤖 AI reads the rules"]
        AI1 --> BYPASS["⚠️ Prompt injection<br/>can override this"]
    end

    subgraph "✅ SPFsmartGATE"
        GATE["🔒 Compiled Rust binary<br/>Rules are in the code"] --> AI2["🤖 AI has no choice"]
        AI2 --> SAFE["✅ Dangerous actions<br/>are physically blocked"]
    end

    style P fill:#E74C3C,stroke:#C0392B,color:#fff
    style AI1 fill:#E74C3C,stroke:#C0392B,color:#fff
    style BYPASS fill:#E74C3C,stroke:#C0392B,color:#fff
    style GATE fill:#27AE60,stroke:#219A52,color:#fff
    style AI2 fill:#27AE60,stroke:#219A52,color:#fff
    style SAFE fill:#27AE60,stroke:#219A52,color:#fff
```

---

## Who is this for?

- **Solo developers using AI coding assistants** (primarily Claude Code) who want an extra safety net beyond the built-in approval prompts
- **Anyone running AI tools on their personal machine** who wants defense-in-depth — dangerous patterns blocked automatically, everything logged

> **Current scope:** This is a single-user tool, primarily developed and tested on Android/Termux. It works on Linux/macOS/Windows (CI builds for all), but Android is where it's been battle-tested. Multi-agent team setups are a design concept described in the docs, not a shipped feature yet.

---

## What does it actually do?

Every AI action passes through **5 security checks** before it touches your system:

```mermaid
graph TD
    REQ["🤖 AI wants to do something"] --> S1
    S1["1️⃣ Rate Limit<br/>Is it going too fast?"] --> S2
    S2["2️⃣ Complexity Score<br/>How risky is this action?"] --> S3
    S3["3️⃣ Validation<br/>Allowed path? Read the file first?"] --> S4
    S4["4️⃣ Content Scan<br/>Passwords? Shell injection?"] --> S5
    S5["5️⃣ Final Decision<br/>Approve or escalate"] --> OUT

    OUT{"Result"}
    OUT -->|"✅ Safe"| EXEC["Action executes"]
    OUT -->|"🚫 Dangerous"| BLOCK["Blocked — AI gets told why"]

    style REQ fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style S1 fill:#3498DB,stroke:#2980B9,color:#fff
    style S2 fill:#3498DB,stroke:#2980B9,color:#fff
    style S3 fill:#3498DB,stroke:#2980B9,color:#fff
    style S4 fill:#3498DB,stroke:#2980B9,color:#fff
    style S5 fill:#3498DB,stroke:#2980B9,color:#fff
    style OUT fill:#F39C12,stroke:#E67E22,color:#fff
    style EXEC fill:#27AE60,stroke:#219A52,color:#fff
    style BLOCK fill:#E74C3C,stroke:#C0392B,color:#fff
```

### Real examples of things it stops:

| What the AI tries to do | What happens | Why |
|---|---|---|
| `rm -rf /home` | **Blocked** | Dangerous command pattern detected |
| Write a file containing `AKIA...` (AWS key) | **Blocked** | Credential pattern found in content |
| Edit `/etc/hosts` | **Blocked** | Path is outside the allowed write list |
| Edit `app.py` without reading it first | **Blocked** | Read-before-write enforced at the gate level (also enforced natively by Claude Code's Edit tool) |
| 100 file writes in 30 seconds | **Blocked** | Rate limit: max 60 writes/minute |
| Fetch `http://169.254.169.254` (cloud metadata) | **Blocked** | SSRF protection — private IPs blocked |
| Read your source code | **Allowed** | Reads are safe and tracked for auditing |
| Write to your project folder | **Allowed** | Path is on the allowed list, file was read first |

---

## How it fits together

```mermaid
graph TB
    subgraph "Your Computer"
        subgraph "SPFsmartGATE (the gate)"
            MCP["📡 MCP Server<br/>Speaks the standard<br/>AI tool protocol"]
            GATE["🛡️ Security Pipeline<br/>5-stage check on<br/>every action"]
            DB["💾 6 Databases<br/>Session logs, config,<br/>agent memory"]
            HOOKS["🪝 31 Hook Scripts<br/>Monitor and log<br/>the full lifecycle"]
        end
        CLI["🖥️ Your AI Assistant<br/>(Claude Code, etc.)"]
        FS["📁 Your Files"]
        WEB["🌐 Internet"]
    end

    CLI -->|"Every tool call"| MCP
    MCP --> GATE
    GATE -->|"Approved"| FS
    GATE -->|"Approved"| WEB
    GATE <--> DB
    HOOKS -.->|"Observe & log"| GATE

    style CLI fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style MCP fill:#3498DB,stroke:#2980B9,color:#fff
    style GATE fill:#F39C12,stroke:#E67E22,color:#fff
    style DB fill:#1ABC9C,stroke:#16A085,color:#fff
    style HOOKS fill:#9B59B6,stroke:#8E44AD,color:#fff
    style FS fill:#27AE60,stroke:#219A52,color:#fff
    style WEB fill:#E67E22,stroke:#D35400,color:#fff
```

**Plain English version:**
1. You use an AI assistant in your terminal, like normal
2. The AI tries to read files, write code, run commands, or hit the web
3. SPFsmartGATE intercepts every action *before* it happens
4. The gate checks safety, logs what happened, and blocks anything dangerous
5. Safe actions go through — you don't even notice the gate is there

The AI doesn't know it's being gated. It just calls tools like `spf_read` and `spf_write` instead of the native ones. From its perspective, everything works normally. From your perspective, the dangerous stuff never happens.

---

## Key features

| Feature | What it means for you |
|---|---|
| **55 gated tools** | File ops, search, web, memory — all passing through security checks |
| **10 hard-blocked tools** | Filesystem tools that are too dangerous are permanently disabled |
| **Build Anchor Protocol** | Formalizes read-before-write into the gate pipeline (Claude Code's Edit tool also requires this natively) |
| **Persistent memory** | 6 LMDB databases for session/config/project state (note: Claude Code and other IDEs now have built-in memory too) |
| **Works 100% offline** | Everything except web tools runs locally — no cloud, no data leaving your device |
| **Cross-platform** | Battle-tested on Android/Termux; CI builds for Linux, macOS, Windows |
| **Single compiled binary** | Core gate is one Rust binary — but hook scripts need Python 3 for complexity calculation and state tracking |
| **43 security boundary tests** | Run `cargo test` to prove every protection works |

> **Note:** 9 Brain tools (vector search) and 16 RAG tools (document collection) are available but require separate external binaries not included in this repo. The core gate, all file/web/bash tools, and all security features work standalone.

---

## Quick start

```bash
# 1. Clone the repo
git clone https://github.com/STONE-CELL-SPF-JOSEPH-STONE/SPFsmartGATE.git
cd SPFsmartGATE

# 2. Run the setup script (handles everything)
bash setup.sh

# 3. Start the MCP server
./target/release/spf-smart-gate serve

# 4. In a new terminal, start Claude through the gate
HOME=~/SPFsmartGATE/LIVE/LMDB5 claude
```

> **Need Rust?** Install it from [rustup.rs](https://rustup.rs/) first.

### Manual build (if you prefer)

```bash
cargo build --release                     # Build the binary
./target/release/spf-smart-gate init-config  # Initialize databases
./target/release/spf-smart-gate serve        # Start the server
```

---

## CLI commands

| Command | What it does |
|---------|-------------|
| `serve` | Start the MCP security gateway |
| `status` | Show current gate status |
| `session` | Show session details |
| `calculate` | Check complexity score for a tool |
| `gate` | Process a single tool call |
| `reset` | Reset session state |
| `init-config` | Set up configuration database |
| `refresh-paths` | Reload blocked/allowed paths |

---

## Project structure

```
SPFsmartGATE/
├── src/                 # Rust source code (15 modules, ~7,800 lines)
├── hooks/               # 31 shell scripts for monitoring
├── scripts/             # Setup and boot scripts
├── LIVE/                # Runtime data (databases, binaries)
├── docs/                # Detailed technical documentation
│   ├── developer-bible.md   # Complete technical deep-dive
│   ├── mcp-tools.md         # All 55 tools explained
│   ├── hooks.md             # Hook system architecture
│   ├── deployment.md        # Build and deploy guide
│   ├── why-spf.md           # Feature highlights & philosophy
│   └── screenshots/         # Images and screenshots
├── config.json          # Runtime configuration
├── build.sh             # Cross-platform build script
├── setup.sh             # One-command installer
├── CHANGELOG.md         # Version history
├── SECURITY.md          # Vulnerability reporting
├── LICENSE.md           # PolyForm Noncommercial 1.0.0
├── COMMERCIAL_LICENSE.md # Commercial use terms
└── NOTICE.md            # Copyright and attribution
```

---

## Documentation

| Document | What's in it |
|----------|-------------|
| **[Why SPFsmartGATE?](docs/why-spf.md)** | Why this exists, what makes it different |
| **[Developer Bible](docs/developer-bible.md)** | Complete technical reference — every module, every feature |
| **[MCP Tools Reference](docs/mcp-tools.md)** | All 55 tools with parameters and examples |
| **[Hook System](docs/hooks.md)** | 31 monitoring scripts explained |
| **[Deployment Guide](docs/deployment.md)** | Building, installing, and configuring |
| **[Security Policy](SECURITY.md)** | How to report vulnerabilities |
| **[Changelog](CHANGELOG.md)** | What changed in each version |

---

## Platform support

| Platform | Status |
|----------|--------|
| Android (Termux) | Primary development and testing platform |
| Linux x86_64 | CI builds, should work |
| macOS (Apple Silicon) | CI builds, should work |
| macOS (Intel) | CI builds (cross-compile), less tested |
| Windows x86_64 | CI builds, platform code exists but least tested |

---

## How this compares to other tools

There are other ways to solve the "keep the AI from breaking my system" problem. Here's an honest look at where SPFsmartGATE sits relative to them.

### Claude Code already does a lot of this

Claude Code's built-in security has gotten strong:
- **Deny rules** block specific tools/paths and are enforced even in bypass mode
- **PreToolUse hooks** can block any tool call via shell scripts — this is deterministic, not prompt-based
- **The Edit tool already requires a read first** — Claude Code natively refuses to edit a file you haven't read in the conversation
- **Docker sandboxing** is officially supported — runs the AI in an isolated container where it can't damage the host at all
- **[claude-code-permissions-hook](https://github.com/kornysietsma/claude-code-permissions-hook)** is an open-source TOML-configured hook that does regex path blocking and shell injection detection — similar scope to SPFsmartGATE's path/content scanning in a much smaller package

**Honest overlap:** ~60-70% of SPFsmartGATE's security features can be achieved with Claude Code's native tools. If you're on desktop and can use Docker, containerized sandboxing is arguably a simpler and more robust answer.

### MCP gateways

| Tool | What it does | How it compares |
|---|---|---|
| **[Lasso MCP Gateway](https://github.com/lasso-security/mcp-gateway)** (~288 stars) | MCP proxy with PII scanning, prompt injection detection, per-server risk scores | More mature plugin architecture; better PII detection |
| **IBM ContextForge** | MCP federation with RBAC, rate limiting, observability | Enterprise-focused, multi-tenant — different target audience |
| **Prompt Security MCP Gateway** | Dynamic risk scores across 13K+ MCP servers | Commercial; more comprehensive risk scoring |
| **Docker MCP Gateway** | Container isolation per MCP server | Stronger isolation model |

### Sandboxing (the "just put it in a box" approach)

| Tool | Stars | Approach |
|---|---|---|
| **[E2B](https://e2b.dev)** | ~11.1K | Firecracker microVMs, 80ms cold start. If the sandbox breaks, the host is untouched. |
| **Docker Sandboxes** | — | Official Claude Code support. Full filesystem/network isolation. |

Container sandboxing is a fundamentally different approach — instead of filtering individual actions, it isolates the entire environment. It's simpler and arguably more robust. **The main case where it doesn't work: Android/Termux**, where Docker is unreliable or unavailable.

### AI memory frameworks

Persistent AI memory is now a competitive space, not a gap:
- **Mem0** — dedicated memory layer with semantic search, 26% accuracy boost in benchmarks
- **Zep** — temporal knowledge graphs, enterprise-focused
- **Copilot Memory** — built into GitHub Copilot as of March 2026
- **Windsurf Memories** — built into Windsurf IDE
- **Claude Code auto-memory** — built into Claude Code
- **Dozens of MCP memory servers** on GitHub (mcp-memory-service, mcp-knowledge-graph, Hindsight, etc.)

SPFsmartGATE's 6 LMDB databases are a specific architectural approach, but the general concept of persistent AI memory is mainstream in 2026.

### What SPFsmartGATE actually adds

After all the overlap, here's what remains:

- **All-in-one integration** — path blocking, content scanning, rate limiting, session tracking, and persistent memory in a single compiled binary, configured once. Stitching together Claude Code deny rules + a permissions hook + a memory MCP server + Docker gets you similar coverage but requires assembling multiple tools.
- **Works on Android/Termux where Docker can't** — Docker is unreliable on Termux. If you develop on mobile, SPFsmartGATE gives you security gating that container approaches can't provide there. (Other AI tools do target Termux — OpenClaw, DroidClaw, agent-loop — but none focus on security gating specifically.)
- **Formalized workflow discipline** — the complexity scoring, tier system, and resource allocation formula are an opinionated framework for how an AI agent should approach tasks. Whether the specific formula is well-calibrated is debatable (the exponents are arbitrary and produce extreme values with small inputs), but the concept of "think more before acting on complex tasks" is sound.
- **Compiled Rust binary** — harder to tamper with at runtime compared to Node.js/Python gateways. Deterministic performance with no GC pauses during security checks.

### What it doesn't do that competitors do

- **No prompt injection detection** — Lasso and Rebuff AI detect prompt injection attacks; SPFsmartGATE doesn't
- **No PII scanning** — Lasso integrates Presidio for PII detection; SPFsmartGATE scans for credential patterns but not general PII
- **No container isolation** — if a tool call passes the gate, it executes with full access to your system within the allowed paths
- **No multi-tenant/team support** — IBM ContextForge and MintMCP handle this; SPFsmartGATE is single-user
- **Noncommercial license** — every major competitor uses Apache 2.0 or similar permissive licenses, which makes them easier to adopt

---

## Current limitations

- **Early project** — v2.0.0 is the first public release. The security features are real and tested, but it hasn't had wide community adoption yet.
- **Android-first** — primarily built and tested on Termux/Android. Other platforms compile via CI but haven't been stress-tested.
- **Hook dependency** — the full security model relies on Claude Code hooks being correctly configured. If hooks break, native tools can bypass the gate.
- **Python 3 required** — the Rust binary is standalone, but the hook system needs Python 3 for complexity calculation and state management.
- **Brain + RAG tools need external binaries** — 25 of the 55 tools require separate programs not included in this repo. The core 30 tools work standalone.
- **Single-user, Claude Code specific** — the hook layer is built for Claude Code's API. The Rust binary speaks standard MCP and works with any client, but you'd need to adapt the hook layer.
- **Complexity formula needs calibration** — the SPF formula exponents (1, 7, 10) are hand-tuned, not empirically derived. They produce extreme values with small inputs (3 dependencies = score of 2,187). The concept is sound but the specific numbers need real-world calibration.

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
