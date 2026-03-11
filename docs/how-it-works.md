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
| **Works 100% offline** | Everything except web tools runs locally — no cloud, no data leaving your device |
| **Cross-platform** | Battle-tested on Android/Termux; CI builds for Linux, macOS, Windows |
| **Single compiled binary** | Core gate is one Rust binary — but hook scripts need Python 3 for complexity calculation and state tracking |
| **43 security boundary tests** | Run `cargo test` to prove every protection works |

---

## The 6 databases — what they actually do

SPFsmartGATE has 6 LMDB databases. The docs list "persistent memory" as a feature, but here's what each one actually does and whether it matters to you:

```mermaid
graph TD
    DB["💾 6 LMDB Databases"] --> GATE_DB["Security plumbing<br/>Powers the gate itself"]
    DB --> AGENT_DB["Agent memory<br/>Human-write-only"]
    DB --> EXT_DB["Brain + RAG<br/>Needs external binaries"]

    GATE_DB --> SESSION["SESSION.DB<br/>Files read/written, rate limits,<br/>complexity history, action log"]
    GATE_DB --> CONFIG["CONFIG.DB<br/>Blocked paths, allowed paths,<br/>dangerous command patterns"]

    AGENT_DB --> LMDB5["LMDB5.DB<br/>Memories, preferences,<br/>cross-session context"]

    EXT_DB --> BRAIN["Brain tools<br/>Needs stoneshell-brain binary<br/>Not included in repo"]
    EXT_DB --> RAG["RAG tools<br/>Needs Python server.py<br/>Not included in repo"]

    GATE_DB --- FS["SPF_FS.DB — Virtual filesystem"]
    GATE_DB --- PROJ["PROJECTS.DB — Project key-value store"]
    GATE_DB --- TMP["TMP.DB — Trust levels, access logs"]

    style DB fill:#F39C12,stroke:#E67E22,color:#fff
    style GATE_DB fill:#3498DB,stroke:#2980B9,color:#fff
    style AGENT_DB fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style EXT_DB fill:#95A5A6,stroke:#7F8C8D,color:#fff
    style SESSION fill:#3498DB,stroke:#2980B9,color:#fff
    style CONFIG fill:#3498DB,stroke:#2980B9,color:#fff
    style LMDB5 fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style BRAIN fill:#95A5A6,stroke:#7F8C8D,color:#fff
    style RAG fill:#95A5A6,stroke:#7F8C8D,color:#fff
    style FS fill:#3498DB,stroke:#2980B9,color:#fff
    style PROJ fill:#3498DB,stroke:#2980B9,color:#fff
    style TMP fill:#3498DB,stroke:#2980B9,color:#fff
```

### Security plumbing (ships with the binary, works out of the box)

| Database | What it does | Who uses it |
|---|---|---|
| **SESSION.DB** | Tracks files read/written this session, rate limit timestamps, complexity scores, action manifests. This is what powers Build Anchor (read-before-write) and rate limiting. | The gate itself — you don't interact with it directly. |
| **CONFIG.DB** | Stores blocked paths, allowed paths, dangerous command patterns, tier thresholds. Loaded at startup, queried on every tool call. | The gate reads it. You configure it via `config.json` or CLI. |
| **SPF_FS.DB** | A 4GB virtual filesystem in LMDB. Stores file content and metadata for the `spf_fs_*` tools (which are currently hard-blocked anyway). | Mostly unused in practice — the blocked FS tools route here. |
| **PROJECTS.DB** | Simple key-value store for project metadata. | Available via `spf_projects_*` tools — basic persistent storage. |
| **TMP.DB** | Trust levels, access logs, and resource tracking for the `/tmp/` and `/projects/` device mounts. | Internal bookkeeping for file operations on device-backed directories. |

**Bottom line:** SESSION and CONFIG are essential security plumbing. The other three are internal infrastructure that most users won't interact with directly.

### Agent memory (human-write-only)

| Database | What it does | The catch |
|---|---|---|
| **LMDB5.DB** | Stores memories (preferences, facts, instructions), cross-session context, and working state. The AI can search and read these memories. | The AI **cannot write** to it. `agent_remember`, `agent_forget`, and `agent_set_state` are all HARD BLOCKED from the AI — only the human can add memories via CLI. This is a deliberate security choice (prevents the AI from poisoning its own memory), but it means this isn't an automatic learning system. |

### Brain + RAG (requires external software not in this repo)

25 of the 55 tools (9 Brain + 16 RAG) call out to **external binaries that are not included**:
- **Brain tools** need `stoneshell-brain` — a separate Rust project that provides vector search via sentence transformers
- **RAG tools** need a Python `server.py` — a document collection and indexing server

Without these binaries installed separately, those 25 tools return errors. **The core security gate (30 tools) works fully standalone** — you only need the external binaries if you want vector search and document collection.

> **In context:** Claude Code, Copilot, Windsurf, and others all have built-in memory systems now. SPFsmartGATE's persistence layer is not a differentiator in 2026 — it's internal plumbing that supports the security gate. The gate itself is the value proposition.

---

Copyright 2026 Joseph Stone. All Rights Reserved.
