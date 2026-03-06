---
name: openclaw-vps-setup
description: Install, configure, and troubleshoot OpenClaw Gateway on a headless Linux VPS (Ubuntu/Debian). Use when a user asks to set up OpenClaw on a VPS, configure gateway tools (exec, web_search, memory), fix "no exec/command runner" errors, set up SSH tunnels for dashboard access, manage systemd services, or troubleshoot gateway tool loading issues.
---

# OpenClaw VPS Setup & Configuration

Complete guide for deploying OpenClaw Gateway on a headless Linux VPS with full tool access, SSH tunnel dashboard, and systemd service management.

## Prerequisites

- Ubuntu/Debian VPS with SSH access
- Node.js 22+ installed
- npm global prefix configured (`~/.npm-global/`)
- OpenClaw installed: `npm install -g openclaw`

## Core Rules

- Require explicit approval before changing gateway auth tokens or security settings.
- Always back up config before changes: OpenClaw auto-creates `.bak` files.
- Never expose gateway port (18789) to public internet — use SSH tunnel.
- Prefer `openclaw config set/unset` over manual JSON editing.
- After config changes, restart gateway or rely on hot-reload (most settings hot-apply).
- Test tool availability after config changes by asking the agent to list its tools.

---

## Phase 1: Installation & PATH Setup

### Install OpenClaw

```bash
npm install -g openclaw
openclaw --version
```

### Fix PATH for Non-Interactive SSH

npm global binaries (`~/.npm-global/bin/`) are not in PATH for non-interactive SSH sessions. This causes `ssh <host> openclaw ...` to fail with "command not found".

**Solution — symlink to system PATH:**

```bash
sudo ln -sf /home/ubuntu/.npm-global/bin/openclaw /usr/local/bin/openclaw
```

**Why**: Non-interactive SSH only loads `/usr/local/bin:/usr/bin:/bin`. Login shell config (`.bashrc`, `.profile`) is not sourced. Symlink is the most reliable fix.

**Verify:**

```bash
ssh <host> "openclaw --version"
```

### Initial Setup

```bash
openclaw onboard        # Interactive wizard
# OR manual:
openclaw setup          # Initialize config and workspace
openclaw configure      # Configure credentials, channels, gateway
```

---

## Phase 2: Gateway as Systemd Service

### Install Service

```bash
openclaw gateway install --port 18789
```

If this fails (e.g., `systemctl is-enabled unavailable`), ensure linger is enabled:

```bash
sudo loginctl enable-linger $(whoami)
```

Then retry `openclaw gateway install --port 18789 --force`.

### Manual Service Creation (Fallback)

If `openclaw gateway install` fails, create the service manually:

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/openclaw-gateway.service << 'EOF'
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
ExecStart=/home/ubuntu/.npm-global/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
Environment=PATH=/home/ubuntu/.npm-global/bin:/usr/local/bin:/usr/bin:/bin
Environment=HOME=/home/ubuntu

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable openclaw-gateway
systemctl --user start openclaw-gateway
```

**Important**: If you used manual creation, later run `openclaw gateway install --force` to sync the token. Otherwise you'll get: "Config token differs from service token."

### Service Management

```bash
systemctl --user status openclaw-gateway    # Check status
systemctl --user restart openclaw-gateway   # Restart
systemctl --user stop openclaw-gateway      # Stop
journalctl --user -u openclaw-gateway -f    # Tail logs
openclaw logs                               # Tail gateway logs via RPC
openclaw health                             # Health check
```

### Verify Gateway Is Running

```bash
ss -tlnp | grep 18789
# Expected: LISTEN on 127.0.0.1:18789
```

---

## Phase 3: SSH Tunnel for Dashboard Access

The gateway binds to `127.0.0.1` (loopback only). Access the dashboard from your local machine via SSH port forwarding.

### Create Tunnel (run on LOCAL machine)

```bash
ssh -N -L 18789:127.0.0.1:18789 <host>
```

Then open: `http://localhost:18789/`

### Troubleshooting Tunnel Issues

**"Address already in use"** — kill stale local process:

```bash
lsof -ti :18789 | xargs kill
```

**"Connection refused"** — gateway not running on remote:

```bash
ssh <host> "ss -tlnp | grep 18789"
# If empty: systemctl --user restart openclaw-gateway
```

**"channel open failed: connect refused"** — tunnel works but remote port not listening. Check service status.

### Security Model

- **Network**: Gateway binds `127.0.0.1` only — not reachable from public internet
- **Transport**: SSH tunnel provides encryption
- **Application**: `#token=...` in URL fragment for WebSocket/API auth (client-side only, never sent to server)

**Never open port 18789 in VPS security group.** SSH tunnel is the correct access method.

---

## Phase 4: Tool Configuration (CRITICAL)

### Tool Profiles

OpenClaw has 4 tool profiles that control which tools the agent can use:

| Profile | Tools Included |
|---|---|
| `minimal` | `session_status` only |
| `messaging` | `message` + session tools only |
| `coding` | `group:fs` + `group:runtime` + `group:sessions` + `group:memory` + `image` |
| `full` | All tools, no restrictions |

**Default for new installs is `messaging`** — this is why exec/process/read/write are missing.

### Set Full Tool Access

```bash
openclaw config set tools.profile full
```

### CRITICAL: Do NOT Set tools.allow with group:openclaw

**This is a common pitfall.** Setting:

```json
{ "tools": { "allow": ["group:openclaw"] } }
```

...creates a whitelist that **excludes** `exec`, `process`, `read`, `write`, `edit`. These core tools belong to `group:runtime` and `group:fs`, which are separate from `group:openclaw`.

**Fix — remove the allow list entirely:**

```bash
openclaw config unset tools.allow
```

With `profile: full` and no `allow` list, all tools are available.

### Tool Groups Reference

| Group | Tools |
|---|---|
| `group:runtime` | `exec`, `bash`, `process` |
| `group:fs` | `read`, `write`, `edit`, `apply_patch` |
| `group:sessions` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status` |
| `group:memory` | `memory_search`, `memory_get` |
| `group:web` | `web_search`, `web_fetch` |
| `group:ui` | `browser`, `canvas` |
| `group:automation` | `cron`, `gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:openclaw` | All built-in OpenClaw tools (**but NOT runtime/fs**) |

### Configure Exec Tool

For the agent to run shell commands on the VPS host:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.security full
openclaw config set tools.exec.ask off
```

| Setting | Value | Meaning |
|---|---|---|
| `host` | `gateway` | Execute on VPS directly (not in Docker sandbox) |
| `security` | `full` | No command filtering |
| `ask` | `off` | No approval prompts |

**Security note**: `security: full` + `ask: off` = agent can run ANY command. Only appropriate for personal single-user bots. For shared bots, use `security: allowlist`.

### Disable Sandbox (if not using Docker)

```bash
openclaw config set agents.defaults.sandbox.mode off
```

If sandbox is not explicitly off and no Docker is configured, exec may fail silently.

### Enable Web Tools

```bash
openclaw config set tools.web.search.enabled true
openclaw config set tools.web.fetch.enabled true
openclaw config set tools.web.search.apiKey '<BRAVE_API_KEY>'
```

Get a free Brave Search API key (2000 queries/month): https://brave.com/search/api/

### Verify Tool Loading

After configuration, **always verify** the agent actually has the tools:

```bash
openclaw agent --agent main --message 'List all your available tool names, one per line.'
```

**Expected output must include**: `exec`, `process`, `read`, `write`, `edit`

If `exec` is missing, check:
1. `tools.profile` is `full`
2. `tools.allow` is NOT set (or includes `group:runtime` + `group:fs`)
3. Gateway was restarted after config changes
4. Telegram session was reset with `/reset`

### Diagnostic Commands

```bash
openclaw sandbox explain                    # Show effective tool policy
openclaw sandbox explain --session <key>    # Per-session tool policy
openclaw config get tools                   # Current tools config
openclaw approvals get                      # Exec approval status
```

---

## Phase 5: Channel Configuration

### Telegram Setup

```bash
openclaw config set channels.telegram.enabled true
openclaw config set channels.telegram.botToken '<BOT_TOKEN>'
openclaw config set channels.telegram.dmPolicy pairing
```

**DM Policies**: `pairing` (approve new users) | `allowlist` (explicit list) | `open` (anyone) | `disabled`

### Disable Group Messages (if DM-only)

```bash
openclaw config set channels.telegram.groupPolicy disabled
```

### Remove a Channel

```bash
openclaw config unset channels.whatsapp
openclaw config unset plugins.entries.whatsapp
systemctl --user restart openclaw-gateway
```

Always check both `channels.<provider>` AND `plugins.entries.<provider>`.

---

## Phase 6: Memory Configuration

Memory search requires an embedding provider API key:

```bash
# Option 1: Use OpenAI embeddings
export OPENAI_API_KEY='sk-...'

# Option 2: Disable memory search if not needed
openclaw config set agents.defaults.memorySearch.enabled false
```

Run `openclaw doctor` to see memory status and provider requirements.

---

## Complete Recommended Config

Minimal production config for a personal VPS bot with full capabilities:

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.3-codex" },
      workspace: "~/.openclaw/workspace",
      sandbox: { mode: "off" },
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8 },
    },
  },
  tools: {
    profile: "full",
    // NO tools.allow — let profile: full handle everything
    web: {
      search: { enabled: true, apiKey: "<BRAVE_API_KEY>" },
      fetch: { enabled: true },
    },
    exec: {
      host: "gateway",
      security: "full",
      ask: "off",
    },
  },
  channels: {
    telegram: {
      enabled: true,
      botToken: "<BOT_TOKEN>",
      dmPolicy: "pairing",
      groupPolicy: "disabled",
    },
  },
  gateway: {
    port: 18789,
    mode: "local",
    bind: "loopback",
    auth: { mode: "token" },
  },
}
```

---

## Troubleshooting

### "no ACP runtime/command runner available"

Agent cannot execute commands. Checklist:

1. `openclaw config get tools.profile` must be `full` (not `messaging`)
2. `openclaw config get tools.allow` should be unset or include `group:runtime`
3. `openclaw config get tools.exec.host` must be `gateway`
4. `openclaw config get agents.defaults.sandbox.mode` must be `off` (if no Docker)
5. Restart gateway: `systemctl --user restart openclaw-gateway`
6. Reset Telegram session: send `/reset` in chat
7. Verify: `openclaw agent --agent main --message 'List your tool names'` must include `exec`

### "Port already in use" on gateway restart

```bash
sudo apt-get install -y lsof
lsof -ti :18789 | xargs kill -9
sleep 1
systemctl --user restart openclaw-gateway
```

### "Config token differs from service token"

```bash
openclaw gateway install --force --port 18789
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

### Gateway starts but tools not loading

Config changes may not apply to existing sessions. After any tool config change:

1. Restart gateway: `systemctl --user restart openclaw-gateway`
2. Reset chat session: `/reset` in Telegram
3. Verify tools: `openclaw agent --agent main --message 'List your tool names'`

### SSH tunnel "connection refused" errors

Three checks in order:

1. **Local port free?** `lsof -ti :18789 | xargs kill` (on local machine)
2. **Gateway running?** `ssh <host> "ss -tlnp | grep 18789"` (must show LISTEN)
3. **Service healthy?** `ssh <host> "systemctl --user status openclaw-gateway"`

---

## Useful Commands Cheat Sheet

```bash
# Config
openclaw config get tools               # View tools config
openclaw config set <path> <value>       # Set config value
openclaw config unset <path>             # Remove config key

# Diagnostics
openclaw doctor                          # Full health check
openclaw health                          # Gateway health
openclaw sandbox explain                 # Tool policy debug
openclaw security audit --deep           # Security audit
openclaw channels list                   # Channel status
openclaw sessions                        # Active sessions
openclaw skills list                     # Available skills

# Service
systemctl --user status openclaw-gateway
systemctl --user restart openclaw-gateway
openclaw logs                            # Gateway logs

# Tools test
openclaw agent --agent main --message '<prompt>'
```
