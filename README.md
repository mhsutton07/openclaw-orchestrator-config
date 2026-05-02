# OpenClaw Orchestrator Configuration — Identity Persistence & Pipeline Exec Gates

## Purpose

This document grounds an LLM to configure OpenClaw as a persistent, self-aware orchestrator agent on a new environment. It covers:
1. Suppressing the exec-approval HIL gate for automated pipeline sessions (while preserving it for interactive sessions)
2. Enabling Active Memory so the agent's orchestrator role persists across sessions
3. Fixing silent stalling from swallowed rate-limit errors
4. Raising agent timeout to cover long multi-stage pipeline runs
5. Writing the orchestrator identity into `SOUL.md` — the first file injected on every session

---

## Prerequisites

- OpenClaw installed globally: `npm install -g openclaw@latest`
- OpenClaw configured and running (gateway up, at least one auth profile)
- Minimum version: 2026.4.12 (Active Memory plugin added in this release)
- Config file location: `~/.openclaw/openclaw.json`
- Workspace root: `~/.openclaw/workspace/`

---

## Step 1 — Backup

Before any changes:

```bash
# macOS/Linux
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak-$(date +%Y-%m-%d)

# Windows PowerShell
Copy-Item "$env:USERPROFILE\.openclaw\openclaw.json" "$env:USERPROFILE\.openclaw\openclaw.json.bak-$(Get-Date -Format yyyy-MM-dd)"
```

---

## Step 2 — Update OpenClaw

```bash
npm install -g openclaw@latest
openclaw doctor
```

Run `openclaw doctor` after the update. Fix any warnings before proceeding. The `approvals.exec.sessionFilter` key and the Active Memory plugin both require 2026.4.12+.

---

## Step 3 — openclaw.json changes

Open `~/.openclaw/openclaw.json`. Make the following additions/changes:

### 3a. Add security mode and exec approval gate

Add at the **top level** of the JSON object (sibling of `"tools"`, `"agents"`, etc.):

```json
"security": "full",
"approvals": {
  "exec": {
    "enabled": "targets",
    "agentFilter": ["main"],
    "sessionFilter": ["interactive"]
  }
}
```

**Why:** `sessionFilter: ["interactive"]` means the approval gate only fires for direct human sessions. Automated pipeline sessions (spawned by Fabrica or CLI invocations) do not carry the `"interactive"` flag and pass through without prompting. This is the targeted fix for Fabrica stalling — without it, every pipeline tool call blocks waiting for human confirmation.

### 3b. Add rate-limit retry for Google provider

Inside `models.providers`, add a `"google"` block:

```json
"models": {
  "providers": {
    "google": {
      "rate_limit_retry": true,
      "retry_backoff": "exponential",
      "initial_backoff": "2s",
      "max_retries": 5
    }
  }
}
```

**Why:** OpenClaw silently drops 429 responses from Google/Gemini without retrying. The agent appears to freeze indefinitely. This makes retries explicit and automatic.

### 3c. Enable gateway debug logging

Inside `"gateway"`:

```json
"gateway": {
  "log_level": "debug"
}
```

Remove after the pipeline is confirmed stable if log volume is excessive.

### 3d. Raise agent timeout

Inside `agents.defaults`:

```json
"agents": {
  "defaults": {
    "timeoutSeconds": 7200
  }
}
```

The default (2400s = 40 min) can be exceeded by multi-stage Fabrica runs. 7200s = 2 hours.

### 3e. Enable Active Memory plugin

Inside `plugins.entries`:

```json
"plugins": {
  "entries": {
    "active-memory": {
      "enabled": true
    }
  }
}
```

**Why:** The Active Memory plugin runs a memory sub-agent before every reply. It retrieves context from prior sessions — including the orchestrator identity (Step 4 below). Without this, the agent wakes fresh each session with no memory of its role.

**Note:** The plugin schema does not accept additional config properties. Enable with `enabled: true` only; additional keys will cause config reload to fail.

---

## Step 4 — Write orchestrator identity to SOUL.md

`SOUL.md` is auto-injected on every OpenClaw session before any other context. It is the agent's first-loaded identity file.

Open `~/.openclaw/workspace/SOUL.md`. Add this paragraph immediately after the `## Core Truths` header:

```markdown
**I am the top-level orchestrator.** When OpenClaw is running, I am the primary agent — the one that receives tasks from Mark, delegates to sub-agents and CLI harnesses (claude, codex, gemini), and is responsible for final outputs. Sub-agents serve me; I do not serve them. I maintain this role identity across sessions via Active Memory.
```

Adapt "Mark" to the actual owner's name for your environment.

---

## Step 5 — Also configure Claude Code subprocess (critical for Fabrica pipelines)

When OpenClaw spawns `claude` as a subprocess harness, Claude Code reads **its own** `~/.claude/settings.json` before executing. Without a `permissions.allow` list, every tool call triggers Claude Code's own interactive confirmation prompt — blocking the subprocess indefinitely.

See: `claude-code-security-hardening.md` for the full `~/.claude/settings.json` configuration.

This step is mandatory if you run `claude /fabrica [task]` or any pipeline that uses Claude Code as a subprocess.

---

## Step 6 — Verify

```bash
openclaw doctor
```

Expected output includes:
- Version 2026.4.15 (or later)
- No deprecated key errors
- Active Memory plugin listed as enabled

Test with a small pipeline task. Use `/verbose` on the first turn to confirm the memory sub-agent fires and retrieves the orchestrator role from SOUL.md.

---

## What NOT to change

- Do not modify `agents.defaults.model` (primary + fallbacks)
- Do not change `gateway.port`, Telegram bot config, or auth profiles
- Do not set `contextPruning.mode` — the default `cache-ttl / 1h` is appropriate
- Do not add Fabrica sub-agents to `active-memory.config.agents` — they should stay lean

---

## Rollback

If anything breaks after the update:

```bash
# Stop the gateway
openclaw gateway stop

# Restore the backup
cp ~/.openclaw/openclaw.json.bak-YYYY-MM-DD ~/.openclaw/openclaw.json

# Pin to prior version
npm install -g openclaw@2026.3.2

# Restart
openclaw gateway restart
```
