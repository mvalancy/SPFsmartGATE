# Who needs SPFsmartGATE?

```mermaid
graph TD
    Q1{"What model are<br/>you running?"} -->|"Frontier<br/>Claude, GPT-4o, Gemini"| LOW["Low risk"]
    Q1 -->|"Mid-size open source<br/>Llama 70B, Mistral"| MED["Medium risk"]
    Q1 -->|"Small local or uncensored<br/>0.5B-7B, dolphin, abliterated"| HIGH["High risk"]

    LOW --> Q2{"Can you use<br/>Docker?"}
    MED --> Q3{"Can you use<br/>Docker?"}
    HIGH --> Q4{"Can you use<br/>Docker?"}

    Q2 -->|Yes| NO1["You don't need SPFsmartGATE"]
    Q2 -->|No| MAYBE["Consider it"]
    Q3 -->|Yes| NO2["Docker is simpler"]
    Q3 -->|No| YES1["SPFsmartGATE helps"]
    Q4 -->|Yes| BOTH["Docker + gate = best"]
    Q4 -->|No| YES2["You need SPFsmartGATE"]

    style Q1 fill:#F39C12,stroke:#E67E22,color:#fff
    style Q2 fill:#3498DB,stroke:#2980B9,color:#fff
    style Q3 fill:#3498DB,stroke:#2980B9,color:#fff
    style Q4 fill:#3498DB,stroke:#2980B9,color:#fff
    style LOW fill:#27AE60,stroke:#219A52,color:#fff
    style MED fill:#F39C12,stroke:#E67E22,color:#fff
    style HIGH fill:#E74C3C,stroke:#C0392B,color:#fff
    style NO1 fill:#95A5A6,stroke:#7F8C8D,color:#fff
    style NO2 fill:#95A5A6,stroke:#7F8C8D,color:#fff
    style MAYBE fill:#F39C12,stroke:#E67E22,color:#fff
    style YES1 fill:#27AE60,stroke:#219A52,color:#fff
    style YES2 fill:#27AE60,stroke:#219A52,color:#fff
    style BOTH fill:#27AE60,stroke:#219A52,color:#fff
```

**Honest answer: it depends on what models you're running and where.**

If you're using Claude Opus/Sonnet through Anthropic's API, you probably don't. These models have extensive safety training, and in practice they don't go rogue. People run `claude --dangerously-skip-permissions` for months of continuous autonomous use without incidents. The model itself is well-behaved enough that a compiled security gate is overkill for most workflows.

**The real risk isn't frontier models. It's everything else.**

The MCP protocol is model-agnostic — any AI agent that speaks MCP can call tools on your system. As local and open-source models become more capable, more people are plugging them into the same tool-calling infrastructure. These models don't have the same safety training:

| Risk level | Models | Do you need a gate? |
|---|---|---|
| **Low** | Claude Opus/Sonnet, GPT-4o, Gemini Pro | **Probably not.** Strong safety training. Well-behaved in practice. |
| **Medium** | Llama 3 70B, Mistral Large, Command R+ | **It helps.** Decent but less tested safety. More likely to make mistakes under complex prompts. |
| **High** | Small local models (0.5B-7B), uncensored fine-tunes (dolphin, abliterated), unknown third-party MCP agents | **Yes.** Weak or no safety training. Chaotic tool calls. This is what SPFsmartGATE is built for. |

---

## Where SPFsmartGATE earns its keep

- **Small local models on device** (0.5B - 7B running on phones, Raspberry Pi, edge hardware via Ollama/llama.cpp) — these models are chaotic. They hallucinate tool calls, invent file paths, and can attempt destructive commands because they lack the safety training of larger models. If you're running a 3B model on your phone through MCP, a compiled gate is the difference between "experimental" and "dangerous."

- **Uncensored/abliterated fine-tunes** — models like dolphin-mixtral or abliterated llama variants are built specifically to remove safety guardrails. People run them for good reasons (research, creative work, avoiding over-refusal), but giving them unrestricted tool access on your actual filesystem is genuinely risky.

- **Multi-agent chains** — when one model orchestrates others, the outer model might be safe but inner models might not be. A gate on the tool layer catches problems regardless of which model in the chain made the call.

- **Android/Termux development** — Docker sandboxing doesn't work reliably on mobile. If you're running local models on an Android device with MCP tool access, SPFsmartGATE is one of the few options for security gating in that environment.

- **Unknown third-party MCP agents** — random agent frameworks from GitHub that you want to try but don't fully trust. The gate lets you give them tool access with guardrails.

## Where you don't need it

- **Claude Code on Opus/Sonnet** — the model is well-behaved, Claude Code has built-in deny rules and approval prompts, and Docker sandboxing is available on desktop. Six months of `--dangerously-skip-permissions` with zero incidents is a real data point.

- **Any workflow where Docker sandboxing works** — container isolation is simpler and more robust than filtering individual tool calls. If Docker is an option, use Docker.

- **Standard desktop development with frontier models** — if you're using GPT-4, Claude, or Gemini Pro through their official APIs with normal safety settings, the models themselves are the guardrail.

- **Code-generation architectures (Lua, Python scripting, etc.)** — if the model generates code that a separate runtime executes, SPFsmartGATE has zero visibility into those actions. See below.

## When SPFsmartGATE doesn't apply at all

SPFsmartGATE gates **MCP tool calls** — the model says `tools/call spf_write` and the gate intercepts it. That's one specific architecture.

A different, equally common architecture: the model **generates code** (Lua, Python, JavaScript, etc.) and a separate executor runs it. Game engines, scripting sandboxes, and many agent frameworks work this way. In that pattern:

```mermaid
graph LR
    MODEL["🤖 Model"] -->|"Generates code"| LUA["📜 Lua / Python<br/>script"]
    LUA -->|"Executed by"| RUNTIME["⚙️ Separate Runtime<br/>Game engine, sandbox, etc."]
    RUNTIME -->|"Touches"| SYSTEM["💻 Filesystem,<br/>Network, etc."]

    SPF["🛡️ SPFsmartGATE"] -.->|"Never sees<br/>these actions"| RUNTIME

    style MODEL fill:#6C5CE7,stroke:#5A4BD1,color:#fff
    style LUA fill:#F39C12,stroke:#E67E22,color:#fff
    style RUNTIME fill:#E74C3C,stroke:#C0392B,color:#fff
    style SYSTEM fill:#3498DB,stroke:#2980B9,color:#fff
    style SPF fill:#95A5A6,stroke:#7F8C8D,color:#fff
```

The gate sits on the MCP layer. If the dangerous action never goes through MCP, the gate is irrelevant. Security in code-generation architectures belongs in the **executor** — not the protocol layer:

- **Game engines** (Unity, Godot) sandbox Lua/GDScript with whitelisted API surfaces
- **Jupyter/IPython** can restrict kernel capabilities
- **Docker/Firecracker** isolate the entire execution environment
- **seccomp/AppArmor** restrict system calls at the OS level
- **WASM sandboxes** (Wasmtime, Wasmer) give memory-safe execution with no host access by default

This isn't a flaw in SPFsmartGATE — it's a scope boundary. MCP tool gating and code execution sandboxing solve different problems at different layers. If your AI agents talk MCP, SPFsmartGATE gates them. If they generate code for a runtime to execute, you need to sandbox the runtime.

---

## Where does it actually run?

SPFsmartGATE compiles to a single native binary. It runs on anything with a Linux terminal — but whether you *need* it depends on the device.

```mermaid
graph LR
    SPF["🛡️ SPFsmartGATE"]

    SPF --> PHONE["📱 Android/Termux<br/>Primary platform"]
    SPF --> SBC["🔧 Raspberry Pi<br/>Jetson, SBCs"]
    SPF --> LINUX["🐧 Linux x86_64"]
    SPF --> MAC["🍎 macOS<br/>ARM + Intel"]
    SPF --> WIN["🪟 Windows<br/>Least tested"]
    SPF -.->|"Not supported"| IOS["📵 iOS"]

    style SPF fill:#F39C12,stroke:#E67E22,color:#fff
    style PHONE fill:#27AE60,stroke:#219A52,color:#fff
    style SBC fill:#27AE60,stroke:#219A52,color:#fff
    style LINUX fill:#3498DB,stroke:#2980B9,color:#fff
    style MAC fill:#3498DB,stroke:#2980B9,color:#fff
    style WIN fill:#95A5A6,stroke:#7F8C8D,color:#fff
    style IOS fill:#E74C3C,stroke:#C0392B,color:#fff
```

### Phones and tablets

| Device | Can it run? | Do you need it? |
|---|---|---|
| **Android phone/tablet (Termux)** | Yes — primary platform, battle-tested | **Yes.** Docker doesn't work here. If you're running local models (Ollama, llama.cpp) with MCP tool access on your phone, this is one of the few ways to gate them. |
| **iPhone / iPad** | **No.** iOS doesn't allow terminal apps or background binaries. | N/A — iOS sandboxes every app. There's no shared filesystem for an AI agent to access. |

### Single-board computers and edge devices

| Device | Can it run? | Do you need it? |
|---|---|---|
| **Raspberry Pi** | Yes — compiles to aarch64-linux | **Yes.** Pi users run small local models for home automation, coding, IoT. Docker is available but heavy on limited RAM. A lightweight compiled gate makes more sense here. |
| **NVIDIA Jetson** | Yes — aarch64-linux, same as Pi | **Yes.** Jetson runs local models with GPU acceleration. Same situation — lightweight gate beats a Docker container on constrained hardware. |
| **Other Linux SBCs** (Orange Pi, Rock, etc.) | Yes — if it runs Linux and has a Rust toolchain | Same as Pi/Jetson. Anywhere you're running local models on limited hardware. |

### Desktops and laptops

| Device | Can it run? | Do you need it? |
|---|---|---|
| **Linux desktop/laptop** | Yes — CI builds for x86_64 | **Probably not.** Docker works fine here. If you're using Claude/GPT through their APIs, the models are well-behaved and Docker sandboxing is the simpler answer. *Could* be useful if you're running uncensored local models without containers. |
| **macOS** | Yes — CI builds for Apple Silicon and Intel | **Probably not.** Same as Linux — Docker and Apple Container sandboxing work well. Built-in Claude Code security is usually enough. |
| **Windows** | Compiles via CI, least tested | **Probably not.** WSL2 + Docker is the standard approach. SPFsmartGATE hasn't been stress-tested here. |

### The pattern

**SPFsmartGATE is most valuable where two things are true at the same time:**
1. You're running models that lack strong safety training (small local models, uncensored fine-tunes)
2. Docker/container sandboxing isn't available or practical (phones, low-RAM SBCs, edge devices)

On a desktop with Docker and a frontier model, you don't need it.

---

Copyright 2026 Joseph Stone. All Rights Reserved.
