# How SPFsmartGATE works

SPFsmartGATE is a compiled Rust binary that sits between any MCP-speaking AI agent and your system. Every tool call passes through a 5-stage security pipeline before execution. The AI can't override it — the rules are in the compiled code, not in a prompt.

> **Important caveat:** The full security model also depends on Claude Code hook scripts being configured correctly (the setup script handles this). The hooks redirect native tools through the gate. If hooks are misconfigured, native tools could bypass the gate. The Rust binary itself is solid — but it's one layer in a defense-in-depth system, not a magic bullet.

---

## The 5-stage pipeline

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

### What each stage does

| Stage | Module | What it checks |
|---|---|---|
| **Rate Limit** | `rate.rs` | Max 60 operations/minute. Prevents runaway loops. |
| **Complexity Score** | `calculate.rs` | Assigns a risk tier (SIMPLE → CRITICAL) based on what the tool does. Higher-tier actions get more scrutiny. |
| **Validation** | `validate.rs` | Is the path allowed? Is the command dangerous (`rm -rf`, `curl \| sh`)? Has the file been read before editing (Build Anchor)? |
| **Content Scan** | `inspect.rs` | Scans content for credential patterns (AWS keys, tokens), path traversal (`../../../etc/passwd`), shell injection. |
| **Final Decision** | `gate.rs` | Aggregates all checks. Approve, block, or flag for user review. |

---

## Architecture

```mermaid
graph TD
    CLI["🖥️ Your AI Assistant"] -->|"Every tool call"| MCP["📡 MCP Server"]
    MCP --> GATE["🛡️ Security Pipeline"]
    GATE -->|"Approved"| FS["📁 Your Files"]
    GATE -->|"Approved"| WEB["🌐 Internet"]
    GATE <-->|"Log and track"| DB["💾 6 Databases"]

    style CLI fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style MCP fill:#3498DB,stroke:#2980B9,color:#fff
    style GATE fill:#F39C12,stroke:#E67E22,color:#fff
    style DB fill:#1ABC9C,stroke:#16A085,color:#fff
    style FS fill:#27AE60,stroke:#219A52,color:#fff
    style WEB fill:#E67E22,stroke:#D35400,color:#fff
```

**Plain English version:**
1. You use an AI assistant in your terminal, like normal
2. The AI tries to read files, write code, run commands, or hit the web
3. SPFsmartGATE intercepts every action *before* it happens
4. The gate checks safety, logs what happened, and blocks anything dangerous
5. Safe actions go through normally

---

## Real examples of things it stops

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

## What happens when something is blocked?

The AI **does** find out. When the gate blocks an action, it sends back a specific error message explaining what was blocked and why:

- `BLOCKED | spf_write | path /etc/hosts is blocked` — the AI knows the path isn't allowed and can try a different location
- `BLOCKED | spf_edit | BUILD ANCHOR — must read file before editing` — the AI knows it needs to read the file first, then retry
- `BLOCKED | spf_bash | dangerous command pattern detected` — the AI knows the command was rejected and can try a safer approach
- `BLOCKED | unknown tool 'evil_tool' — not in gate allowlist` — the AI knows that tool doesn't exist in the system

This is important. A silent block would be useless — the AI would just keep retrying the same thing. Instead, the error message gives the AI enough information to **change direction**: use a different path, read the file first, rephrase the command, or ask the user for guidance. The gate blocks the dangerous action *and* teaches the AI why, so it can recover.

---

## Key features

| Feature | What it means |
|---|---|
| **55 gated tools** | File ops, search, web, memory — all passing through security checks |
| **10 hard-blocked tools** | Filesystem tools that are too dangerous are permanently disabled |
| **Build Anchor Protocol** | Formalizes read-before-write into the gate pipeline (Claude Code's Edit tool also requires this natively) |
| **Persistent memory** | 6 LMDB databases for session/config/project state (note: Claude Code and other IDEs now have built-in memory too) |
| **Works 100% offline** | Everything except web tools runs locally — no cloud, no data leaving your device |
| **Cross-platform** | Battle-tested on Android/Termux; CI builds for Linux, macOS, Windows |
| **Single compiled binary** | Core gate is one Rust binary — but hook scripts need Python 3 for complexity calculation and state tracking |
| **43 security boundary tests** | Run `cargo test` to prove every protection works |

> **Note:** 9 Brain tools (vector search) and 16 RAG tools (document collection) require separate external binaries not included in this repo. The core gate, all file/web/bash tools, and all security features work standalone.

---

Copyright 2026 Joseph Stone. All Rights Reserved.
