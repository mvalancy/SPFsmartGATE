# scripts/ — Setup and Boot Scripts

2 support scripts for LMDB5 virtual filesystem management.

| Script | Size | What it does |
|--------|------|-------------|
| **boot-lmdb5.sh** | 1.1 KB | Called by session-start.sh on every session boot. Loads the LMDB5 manifest from `storage/lmdb5_manifest.json`, locates the SPF binary blob, and exports virtual path environment variables (`SPF_AGENT_HOME`, `SPF_ACTIVE`, etc.). |
| **install-lmdb5.sh** | 15 KB | One-time installation script. Copies Claude CLI, configs, and data into LMDB5 virtual filesystem containment. Runs 11 phases: preflight checks, backup, binary hashing, config staging, manifest creation, boot injection, symlink setup, and verification. |

## When are these used?

- **boot-lmdb5.sh** runs automatically every time a Claude Code session starts (sourced by `hooks/session-start.sh`)
- **install-lmdb5.sh** runs once during initial setup (`bash scripts/install-lmdb5.sh`) to set up the LMDB5 agent home directory

## Deep dive

See [Deployment Guide](../docs/deployment.md) for the full build and deploy pipeline.

---

## License

**Free for personal use.** Commercial use requires a paid license.

Licensed under the [PolyForm Noncommercial License 1.0.0](../LICENSE.md).
See [COMMERCIAL_LICENSE.md](../COMMERCIAL_LICENSE.md) for business use, or email **joepcstone@gmail.com**.

---

<p align="center">
  Copyright 2026 Joseph Stone. All Rights Reserved.<br/>
  <em>SPFsmartGATE and the StoneCell Processing Formula (SPF) are proprietary intellectual property.</em>
</p>
