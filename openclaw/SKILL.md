# OpenClaw Setup Skill

## Overview

OpenClaw is a personal AI assistant you run on your own devices. It connects to messaging channels you already use (WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, Microsoft Teams, Matrix, and more) and can speak/listen on macOS/iOS/Android with a live Canvas workspace.

**Key concepts:**
- **Gateway**: The central control plane that manages sessions, channels, tools, and events
- **Channels**: Messaging platforms OpenClaw connects to (WhatsApp, Telegram, Discord, etc.)
- **Skills**: Instructions that teach the agent how to use tools
- **Workspace**: Directory for agent files (default: `~/.openclaw/workspace`)
- **Sessions**: Conversation contexts, isolated per chat/group

---

# PRE-INSTALL: Requirements & Preparation

## System Requirements

| Requirement | Details |
|-------------|---------|
| **Node.js** | Version 22+ (required) |
| **OS** | macOS, Linux, or Windows (WSL2 strongly recommended for Windows) |
| **Docker** | Only needed for containerized deployments |
| **pnpm** | Only needed for building from source |

## Pre-Install Checks

### 1. Verify Node.js Version
```bash
node --version  # Must show v22.x.x or higher
```

**If Node.js is missing or outdated:**
```bash
# macOS (Homebrew)
brew install node@22

# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Windows (via nvm-windows in WSL2)
nvm install 22
nvm use 22
```

### 2. Verify npm is Working
```bash
npm --version
npm prefix -g   # Shows global install path
```

### 3. Check PATH Configuration
```bash
echo $PATH
# Ensure $(npm prefix -g)/bin is in PATH
```

**Fix PATH issues (add to `~/.zshrc` or `~/.bashrc`):**
```bash
export PATH="$(npm prefix -g)/bin:$PATH"
```

### 4. Pre-install Troubleshooting

| Issue | Symptom | Solution |
|-------|---------|----------|
| Node too old | `node --version` < 22 | Upgrade Node.js |
| npm not found | `command not found: npm` | Install Node.js properly |
| Permission denied | EACCES errors during install | Fix npm permissions or use `--prefix` |
| WSL not configured | Windows native npm issues | Install and use WSL2 |

---

# INSTALLATION

## Prerequisites

- **Node.js 22+** (required)
- For Docker: Docker Desktop or Docker Engine + Docker Compose v2
- For iMessage: macOS with BlueBubbles server

## Installation Methods

### Method 1: Quick Install (Recommended)

```bash
# macOS/Linux
curl -fsSL https://openclaw.ai/install.sh | bash

# Windows (PowerShell)
iwr -useb https://openclaw.ai/install.ps1 | iex

# Then run onboarding
openclaw onboard --install-daemon
```

### Method 2: npm/pnpm Global Install

```bash
npm install -g openclaw@latest
# or
pnpm add -g openclaw@latest

openclaw onboard --install-daemon
```

### Method 3: From Source (Development)

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm ui:build
pnpm build
pnpm openclaw onboard --install-daemon
```

### Method 4: Docker

```bash
# Clone repo and run setup script
git clone https://github.com/openclaw/openclaw.git
cd openclaw
./docker-setup.sh

# Or manual Docker Compose
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm openclaw-cli onboard
docker compose up -d openclaw-gateway
```

## Installation Troubleshooting

### `openclaw` Command Not Found After Install

**Diagnosis:**
```bash
node -v
npm -v
npm prefix -g
echo "$PATH"
```

**Fix:** Add npm global bin to PATH:
```bash
# Add to ~/.zshrc or ~/.bashrc
export PATH="$(npm prefix -g)/bin:$PATH"
# Then reload shell
source ~/.zshrc  # or ~/.bashrc
```

### sharp Build Errors (npm install)
```bash
# Force prebuilt binaries
SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
```

### pnpm "Ignored Build Scripts" Warning
```bash
pnpm add -g openclaw@latest
pnpm approve-builds -g  # Select openclaw, node-llama-cpp, sharp
```

### Permission Errors (EACCES)
```bash
# Option 1: Fix npm directory ownership
sudo chown -R $(whoami) $(npm config get prefix)/{lib/node_modules,bin,share}

# Option 2: Use a different npm prefix
npm config set prefix ~/.npm-global
export PATH=~/.npm-global/bin:$PATH
```

### Install Verification
```bash
openclaw --version        # Should show version
openclaw doctor           # Check for issues
openclaw gateway status   # Gateway status
```

---

# POST-INSTALL: Configuration

Config file location: `~/.openclaw/openclaw.json` (JSON5 format)

### Essential Configuration Structure

```json5
{
  // Gateway settings
  gateway: {
    mode: "local",
    port: 18789,
    bind: "loopback",  // loopback | lan | all
    auth: {
      mode: "token",
      token: "your-long-random-token"  // Generate: openssl rand -hex 32
    }
  },
  
  // Agent defaults
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      model: {
        primary: "anthropic/claude-sonnet-4-5",
        fallbacks: ["openai/gpt-5.2"]
      }
    }
  },
  
  // Channel configurations
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",  // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123"]
    },
    telegram: {
      enabled: true,
      botToken: "123456:ABCDEF...",
      dmPolicy: "pairing"
    },
    discord: {
      enabled: true,
      botToken: "your-discord-bot-token",
      dmPolicy: "pairing"
    },
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      dmPolicy: "pairing"
    }
  },
  
  // Session isolation
  session: {
    dmScope: "per-channel-peer"  // Recommended for multi-user inboxes
  },
  
  // Tool permissions
  tools: {
    profile: "messaging",  // messaging | minimal | full
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" }
  }
}
```

### Configuration Methods

```bash
# Interactive wizard
openclaw onboard       # Full setup
openclaw configure     # Config wizard

# CLI one-liners
openclaw config get agents.defaults.workspace
openclaw config set gateway.port 18789
openclaw config unset tools.web.search.apiKey

# Control UI
openclaw dashboard     # Opens http://127.0.0.1:18789/
```

## Channel Setup

### WhatsApp
- Uses Baileys library (WhatsApp Web protocol)
- Requires QR code pairing
- Config: `channels.whatsapp`
- Stores credentials at: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`

**Setup Steps:**
```bash
# 1. Configure access policy in openclaw.json
# 2. Link WhatsApp
openclaw channels login --channel whatsapp
# 3. Scan QR code with WhatsApp
# 4. Start gateway
openclaw gateway
# 5. Approve first pairing
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

**Config Example:**
```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"]
    }
  }
}
```

### Telegram
- Uses grammY bot framework
- Requires bot token from @BotFather
- Config: `channels.telegram`
- Supports groups and forums

**Setup Steps:**
1. Chat with @BotFather on Telegram
2. Run `/newbot`, follow prompts
3. Copy the bot token

**Config Example:**
```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123456:ABCDEF...",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } }
    }
  }
}
```

**Finding Your Telegram User ID:**
- DM your bot, run `openclaw logs --follow`, look for `from.id`
- Or: `curl "https://api.telegram.org/bot<TOKEN>/getUpdates"`

**Privacy Mode:** Bots don't see all group messages by default. Either:
- Disable privacy via `/setprivacy` in @BotFather
- Make bot a group admin

### Discord
- Uses Discord Bot API + Gateway
- Requires bot token from Discord Developer Portal
- Config: `channels.discord`
- Supports servers, channels, and DMs

**Setup Steps:**
1. Create app at [Discord Developer Portal](https://discord.com/developers/applications)
2. Go to Bot → Reset Token → Copy token
3. Enable Privileged Intents:
   - Message Content Intent (required)
   - Server Members Intent (recommended)
4. Generate invite URL (OAuth2):
   - Scopes: `bot`, `applications.commands`
   - Permissions: View Channels, Send Messages, Read Message History, Embed Links, Attach Files
5. Add bot to server via invite URL

**Config Example:**
```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
      dmPolicy: "pairing",
      guilds: {
        "SERVER_ID": {
          requireMention: true,
          users: ["YOUR_USER_ID"]
        }
      }
    }
  }
}
```

### Slack
- Uses Bolt SDK (Socket Mode by default)
- Requires bot token (`xoxb-...`) + app token (`xapp-...`)
- Config: `channels.slack`

**Setup Steps:**
1. Create Slack app
2. Enable Socket Mode
3. Create App Token with `connections:write`
4. Install app to workspace
5. Copy Bot Token (`xoxb-...`)
6. Subscribe to bot events: `app_mention`, `message.channels`, `message.groups`, `message.im`, `message.mpim`

**Config Example:**
```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-...",
      botToken: "xoxb-..."
    }
  }
}
```

### Signal
- Uses signal-cli (external dependency)
- Privacy-focused
- Config: `channels.signal`

**Prerequisites:**
- `signal-cli` installed
- Phone number for verification

**Setup Path A (QR Link - Recommended):**
```bash
signal-cli link -n "OpenClaw"
# Scan QR with Signal app
```

**Setup Path B (SMS Register):**
```bash
# Install signal-cli
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/

# Register
signal-cli -a +<PHONE> register
# If captcha required, get token from signalcaptchas.org/registration/generate.html
signal-cli -a +<PHONE> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<PHONE> verify <CODE>
```

**Config Example:**
```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"]
    }
  }
}
```

### iMessage (BlueBubbles)
- **Recommended**: Use BlueBubbles extension
- Requires macOS with BlueBubbles server
- Config: `extensions/bluebubbles`
- Full feature support (edit, unsend, effects, reactions)

**Setup:**
1. Install BlueBubbles on macOS
2. Configure webhook URL
3. Enable in OpenClaw config

### Microsoft Teams
- Uses Bot Framework
- Plugin: `extensions/msteams`
- Enterprise support

## Model Provider Configuration

### API Keys via Environment

```bash
# In ~/.openclaw/.env or environment
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
OPENROUTER_API_KEY=sk-or-...
```

### Supported Providers
- **Anthropic** (Claude) - Recommended: Opus 4.6 for best results
- **OpenAI** (GPT)
- **OpenRouter** (aggregator)
- **Google** (Gemini)
- **Venice AI** - Privacy-first option
- **Amazon Bedrock**
- **Custom OpenAI-compatible endpoints**

### Model Selection

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["anthropic/claude-sonnet-4-5", "openai/gpt-5.2"]
      },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "anthropic/claude-sonnet-4-5": { alias: "Sonnet" }
      }
    }
  }
}
```

## Security Configuration

### DM Policies
- `pairing` (default): Unknown senders get a one-time pairing code
- `allowlist`: Only senders in `allowFrom` list
- `open`: Allow all DMs (requires `allowFrom: ["*"]`)
- `disabled`: Ignore all DMs

### Hardened Baseline Config

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    auth: { mode: "token", token: "replace-with-long-random-token" }
  },
  session: {
    dmScope: "per-channel-peer"  // Isolate DMs per user
  },
  tools: {
    profile: "messaging",
    deny: ["group:automation", "group:runtime", "group:fs"],
    fs: { workspaceOnly: true },
    exec: { security: "deny", ask: "always" },
    elevated: { enabled: false }
  },
  channels: {
    whatsapp: { dmPolicy: "pairing" }
  }
}
```

### Security Audit

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --fix
```

## Essential Commands

### Gateway Management

```bash
# Start gateway
openclaw gateway --port 18789
openclaw gateway --port 18789 --verbose  # Debug mode
openclaw gateway --force                  # Force-kill existing and start

# Check status
openclaw gateway status
openclaw gateway status --deep

# Service control
openclaw gateway install    # Install as system service
openclaw gateway restart
openclaw gateway stop
```

### Diagnostics

```bash
openclaw doctor             # Health check + migrations
openclaw doctor --yes       # Auto-accept repairs
openclaw doctor --deep      # Scan for extra gateway installs
openclaw doctor --fix       # Apply recommended fixes

openclaw logs --follow      # Follow gateway logs
openclaw channels status --probe  # Check channel health
```

### Messaging

```bash
# Send a message
openclaw message send --to +15555550123 --message "Hello"

# Agent interaction
openclaw agent --message "What's the weather?" --thinking high

# Control UI
openclaw dashboard
```

### Pairing (DM Approvals)

```bash
openclaw pairing approve <channel> <code>
openclaw devices list
openclaw devices approve <requestId>
```

## Docker Environment Variables

For Docker deployment, set these in your environment or `.env`:

```bash
OPENCLAW_GATEWAY_TOKEN=your-token
OPENCLAW_CONFIG_DIR=~/.openclaw
OPENCLAW_WORKSPACE_DIR=~/.openclaw/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_GATEWAY_BIND=lan

# Model provider keys
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...

# Channel tokens
TELEGRAM_BOT_TOKEN=123456:ABCDEF...
DISCORD_BOT_TOKEN=...
SLACK_BOT_TOKEN=xoxb-...
```

## Troubleshooting

### Common Issues

1. **Gateway won't start**
   ```bash
   openclaw doctor              # Check for config issues
   openclaw gateway status      # Check if already running
   ss -ltnp | grep 18789        # Check port usage (Linux)
   lsof -i :18789               # Check port usage (macOS)
   ```

2. **Channel not connecting**
   ```bash
   openclaw channels status --probe
   openclaw logs --follow
   ```

3. **Authentication issues**
   ```bash
   openclaw doctor --repair
   openclaw login               # Re-authenticate with model provider
   ```

4. **Config validation errors**
   ```bash
   openclaw doctor              # Shows exact issues
   openclaw doctor --fix        # Apply repairs
   ```

5. **Missing dependencies**
   ```bash
   node --version               # Must be 22+
   openclaw doctor --deep       # Full system check
   ```

---

# RUNTIME TROUBLESHOOTING

## First Response Ladder (Run These First)

When something is broken, run these commands in order:

```bash
openclaw status              # Quick status overview
openclaw status --all        # Full report
openclaw gateway probe       # Test gateway reachability
openclaw gateway status      # Gateway runtime state
openclaw doctor              # Health check + migrations
openclaw channels status --probe  # Channel connectivity
openclaw logs --follow       # Live log stream
```

**Healthy Output Checklist:**
- `Runtime: running`
- `RPC probe: ok`
- Channels show `connected` or `ready`
- No repeating fatal errors in logs

## Gateway Problems

### Gateway Won't Start

**Error Signatures & Solutions:**

| Log Signature | Cause | Fix |
|--------------|-------|-----|
| `Gateway start blocked: set gateway.mode=local` | Gateway mode not set | Set `gateway.mode="local"` in config |
| `refusing to bind gateway ... without auth` | Non-loopback bind without token | Set `gateway.auth.token` or use `--bind loopback` |
| `another gateway instance is already listening` | Port conflict | `openclaw gateway --force` or kill existing process |
| `EADDRINUSE` | Port 18789 in use | Check `ss -ltnp \| grep 18789` / `lsof -i :18789` |

**Commands:**
```bash
openclaw gateway status
openclaw gateway status --deep
openclaw doctor
openclaw gateway --force      # Force-kill and restart
```

### Gateway Service Not Running

```bash
# Check service status
openclaw gateway status

# Reinstall service
openclaw gateway install

# Manual restart
openclaw gateway restart

# Check for config issues
openclaw doctor --deep
```

### Port Conflicts

```bash
# Linux
ss -ltnp | grep 18789    # Find what's using port

# macOS
lsof -i :18789

# Force kill existing gateway and restart
openclaw gateway --force
```

## No Replies Problem

**Diagnosis Steps:**
```bash
openclaw status
openclaw channels status --probe
openclaw pairing list <channel>
openclaw config get channels
openclaw logs --follow
```

**Common Causes:**

| Log Signature | Meaning | Fix |
|--------------|---------|-----|
| `drop guild message (mention required` | Group mention gating | Mention the bot or set `requireMention: false` |
| `pairing request` | Sender not approved | `openclaw pairing approve <channel> <code>` |
| `blocked` / `allowlist` | Sender/channel filtered | Add to `allowFrom` list |

## Dashboard/Control UI Issues

**Symptoms:**
- "unauthorized" or "disconnected (1008)"
- Cannot connect to Control UI
- Reconnect loop

**Commands:**
```bash
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

**Common Causes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `device identity required` | HTTP/non-secure context | Use HTTPS or localhost |
| `unauthorized` / reconnect loop | Token mismatch | Verify `gateway.auth.token` matches |
| `gateway connect failed:` | Wrong URL/port | Check dashboard URL in `openclaw gateway status` |

**Fix Device Pairing:**
```bash
openclaw dashboard --no-open   # Get fresh URL
openclaw devices list
openclaw devices approve <requestId>
```

## Channel Troubleshooting

### WhatsApp Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| Connected but no DM replies | `openclaw pairing list whatsapp` | Approve sender or update allowlist |
| Group messages ignored | Check `requireMention` config | Mention bot or relax mention policy |
| Random disconnect/relogin loops | `openclaw channels status --probe` | Re-login, check credentials directory |

**Re-link WhatsApp:**
```bash
openclaw channels login --channel whatsapp
```

### Telegram Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| `/start` but no replies | `openclaw pairing list telegram` | Approve pairing |
| Bot online but group silent | Privacy mode + mention config | Disable privacy mode via `/setprivacy` in BotFather |
| Send failures | Check logs for API errors | Fix DNS/network to `api.telegram.org` |
| Allowlist blocks you after upgrade | `openclaw security audit` | Run `openclaw doctor --fix` to resolve @usernames to IDs |

**Finding Your Telegram User ID:**
```bash
# Method 1: DM your bot and check logs
openclaw logs --follow
# Look for "from.id" in the log output

# Method 2: Use Bot API
curl "https://api.telegram.org/bot<BOT_TOKEN>/getUpdates"
```

### Discord Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| Bot online but no guild replies | `openclaw channels status --probe` | Allow guild/channel, check message content intent |
| Group messages ignored | Check logs for mention gating | Mention bot or set `requireMention: false` |
| DM replies missing | `openclaw pairing list discord` | Approve DM pairing |

**Required Discord Bot Intents:**
- Message Content Intent (required)
- Server Members Intent (recommended)
- Presence Intent (optional)

### Slack Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| Socket connected but no responses | `openclaw channels status --probe` | Verify app token + bot token + scopes |
| DMs blocked | `openclaw pairing list slack` | Approve pairing or adjust DM policy |
| Channel message ignored | Check `groupPolicy` | Allow channel or set policy to `open` |

**Required Slack Bot Events:**
- `app_mention`
- `message.channels`, `message.groups`, `message.im`, `message.mpim`
- `reaction_added`, `reaction_removed`

### Signal Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| Daemon reachable but bot silent | `openclaw channels status --probe` | Verify `signal-cli` daemon URL/account |
| DM blocked | `openclaw pairing list signal` | Approve sender |
| Group replies don't trigger | Check group allowlist | Add sender/group to allowlist |

### iMessage/BlueBubbles Issues

| Symptom | Check | Fix |
|---------|-------|-----|
| No inbound events | Webhook/server reachability | Fix webhook URL or BlueBubbles server |
| Can send but no receive (macOS) | macOS privacy permissions | Re-grant TCC permissions, restart channel |
| DM sender blocked | `openclaw pairing list bluebubbles` | Approve pairing |

## Model/Authentication Issues

### "No credentials found"

```bash
# Check auth status
openclaw models status

# For API key auth (recommended)
export ANTHROPIC_API_KEY="sk-ant-..."  # or add to ~/.openclaw/.env
openclaw models status

# For setup-token (subscription)
claude setup-token
openclaw models auth setup-token --provider anthropic
```

### OAuth Token Expired

```bash
openclaw doctor       # Will detect and offer to refresh
openclaw models status --check  # Exit 1=expired, 2=expiring
```

### API Key Not Being Read

Check precedence (highest → lowest):
1. Process environment
2. `./.env` (current directory)
3. `~/.openclaw/.env`
4. Config `env` block in `openclaw.json`

```bash
# Verify key is set
openclaw config get env
cat ~/.openclaw/.env
```

## Common Error Messages & Solutions

| Error Message | Cause | Solution |
|--------------|-------|----------|
| `Gateway start blocked: set gateway.mode=local` | Mode not configured | `openclaw config set gateway.mode local` |
| `refusing to bind gateway ... without auth` | Security requirement | Set `gateway.auth.token` or bind to loopback only |
| `EADDRINUSE` | Port conflict | `openclaw gateway --force` |
| `device identity required` | Non-HTTPS access | Use localhost or HTTPS |
| `unauthorized` | Token mismatch | Check `gateway.auth.token` value |
| `pairing request` | Unknown sender | `openclaw pairing approve <channel> <code>` |
| `mention required` | Group mention gating | Mention bot or disable `requireMention` |
| `missing_scope` / `Forbidden` / `401/403` | Channel permission issue | Reconfigure channel tokens/scopes |
| `No credentials found` | Missing API key/token | Add API key to env or run `openclaw onboard` |

---

# UPDATING & MAINTENANCE

## Update Methods

### Recommended: Re-run Installer
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
# Add --no-onboard to skip wizard
```

### npm/pnpm Update
```bash
npm i -g openclaw@latest
# or
pnpm add -g openclaw@latest

# After update
openclaw doctor
openclaw gateway restart
```

### Source Install Update
```bash
openclaw update
# or manual:
git pull
pnpm install
pnpm build
pnpm ui:build
openclaw doctor
```

### Switch Update Channels
```bash
openclaw update --channel beta
openclaw update --channel dev
openclaw update --channel stable
```

## After Every Update

```bash
openclaw doctor            # Always run after updating
openclaw gateway restart   # Restart gateway
openclaw health            # Verify health
```

## Rollback to Previous Version

```bash
# Check current published version
npm view openclaw version

# Install specific version
npm i -g openclaw@<version>
# or
pnpm add -g openclaw@<version>

# Then restart
openclaw doctor
openclaw gateway restart
```

## Uninstall

### With CLI Still Installed
```bash
openclaw uninstall --all --yes
```

### Manual Uninstall
```bash
# Stop and remove service
openclaw gateway stop
openclaw gateway uninstall

# Remove state and config
rm -rf ~/.openclaw

# Remove CLI
npm rm -g openclaw
# or: pnpm remove -g openclaw

# macOS app
rm -rf /Applications/OpenClaw.app
```

### Remove Orphaned Services

**macOS (launchd):**
```bash
launchctl bootout gui/$UID/bot.molt.gateway
rm -f ~/Library/LaunchAgents/bot.molt.gateway.plist
```

**Linux (systemd):**
```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

---

# FAQ & COMMON QUESTIONS

## General Questions

**Q: What ports does OpenClaw use?**
A: Default gateway port is 18789. Can be changed via `gateway.port` config.

**Q: Where is my data stored?**
A: All data is in `~/.openclaw/` including config, credentials, sessions, and workspace.

**Q: Can I run multiple agents?**
A: Yes, use `openclaw agents add <name>` to create additional agents with separate workspaces.

**Q: Which model provider is recommended?**
A: Anthropic Claude Opus 4.6 is recommended for best results. Use API key auth.

**Q: Can I use my own API keys?**
A: Yes, set them in `~/.openclaw/.env` or via environment variables.

## Channel Questions

**Q: Which channel is easiest to set up?**
A: Telegram - just need a bot token from @BotFather.

**Q: WhatsApp keeps disconnecting?**
A: Re-link with `openclaw channels login --channel whatsapp`. Check credentials directory health.

**Q: Discord bot doesn't respond in servers?**
A: Enable Message Content Intent in Discord Developer Portal and verify guild allowlist.

**Q: How do I approve someone to message my bot?**
A: Use pairing mode (default). When they message, run `openclaw pairing approve <channel> <code>`.

## Security Questions

**Q: Is my data secure?**
A: All data stays local. Set `gateway.auth.token` for network access. Use pairing mode for DMs.

**Q: How do I restrict who can message the bot?**
A: Use `dmPolicy: "pairing"` (default) or `dmPolicy: "allowlist"` with specific `allowFrom` entries.

**Q: Should I expose the gateway to the internet?**
A: Use `bind: loopback` for local-only. For remote access, always set a strong `gateway.auth.token`.

## Troubleshooting Questions

**Q: How do I see what's happening?**
A: `openclaw logs --follow` for live logs, `openclaw status --all` for full status.

**Q: How do I reset everything?**
A: `openclaw doctor --reset` or delete `~/.openclaw/` and run `openclaw onboard` again.

**Q: Config won't validate?**
A: Run `openclaw doctor --fix` to auto-repair. Check for deprecated/legacy keys.

**Q: Gateway starts but immediately stops?**
A: Check `openclaw logs --follow` for errors. Usually config validation or port conflict.

### Path References

| Path | Purpose |
|------|---------|
| `~/.openclaw/` | State directory |
| `~/.openclaw/openclaw.json` | Main config file |
| `~/.openclaw/.env` | Environment variables |
| `~/.openclaw/workspace/` | Default workspace |
| `~/.openclaw/credentials/` | Channel credentials |
| `~/.openclaw/sessions/` | Session transcripts |
| `~/.openclaw/skills/` | Managed/local skills |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `OPENCLAW_HOME` | Override home directory for all path resolution |
| `OPENCLAW_STATE_DIR` | Override state directory (default: `~/.openclaw`) |
| `OPENCLAW_CONFIG_PATH` | Override config file path |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token |
| `OPENCLAW_GATEWAY_PORT` | Gateway port (default: 18789) |

## Skills System

Skills are loaded from:
1. **Bundled skills**: Shipped with OpenClaw
2. **Managed skills**: `~/.openclaw/skills`
3. **Workspace skills**: `<workspace>/skills`

Precedence: workspace > managed > bundled

### ClawHub (Skill Registry)

```bash
# Install a skill
clawhub install <skill-slug>

# Update all skills
clawhub update --all

# Browse skills
# https://clawhub.com
```

## Multi-Agent Setup

```bash
openclaw agents add <name>           # Create new agent
openclaw agents list                 # List agents
openclaw configure --section agents  # Configure agents
```

Each agent can have:
- Separate workspace
- Own sessions
- Custom model/auth profiles
- Dedicated channel bindings

## VPS/Remote Deployment

For running on a VPS:

```bash
# Set local mode
openclaw config set gateway.mode local

# Use lan binding for remote access
openclaw config set gateway.bind lan

# Generate strong token
export TOKEN=$(openssl rand -hex 32)
openclaw config set gateway.auth.token "$TOKEN"

# Start with nohup
nohup openclaw gateway run --bind lan --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &

# Verify
ss -ltnp | grep 18789
tail -n 50 /tmp/openclaw-gateway.log
```

## Quick Client Setup Checklist

- [ ] Node.js 22+ installed
- [ ] OpenClaw installed (`openclaw --version`)
- [ ] Onboarding completed (`openclaw onboard --install-daemon`)
- [ ] Gateway running (`openclaw gateway status`)
- [ ] Model provider authenticated (API key or OAuth)
- [ ] At least one channel configured
- [ ] DM policy set appropriately (pairing recommended)
- [ ] Security audit passed (`openclaw security audit`)
- [ ] Control UI accessible (`openclaw dashboard`)

---

# COMPLETE COMMAND REFERENCE

## Gateway Commands
```bash
openclaw gateway                  # Start gateway (foreground)
openclaw gateway --port 18789     # Specify port
openclaw gateway --verbose        # Debug mode
openclaw gateway --force          # Kill existing and start
openclaw gateway status           # Check status
openclaw gateway status --deep    # Detailed status
openclaw gateway status --json    # JSON output
openclaw gateway install          # Install as system service
openclaw gateway uninstall        # Remove system service
openclaw gateway restart          # Restart gateway service
openclaw gateway stop             # Stop gateway
openclaw gateway probe            # Test gateway reachability
```

## Diagnostics Commands
```bash
openclaw doctor                   # Health check + migrations
openclaw doctor --yes             # Auto-accept repairs
openclaw doctor --fix             # Apply recommended fixes
openclaw doctor --repair          # Apply repairs without prompts
openclaw doctor --repair --force  # Force aggressive repairs
openclaw doctor --deep            # Scan for extra installs
openclaw doctor --non-interactive # Safe migrations only

openclaw status                   # Quick status
openclaw status --all             # Full status report
openclaw health                   # Health check
openclaw logs --follow            # Stream logs
```

## Channel Commands
```bash
openclaw channels status          # Channel status
openclaw channels status --probe  # Probe connectivity
openclaw channels login --channel <name>  # Login to channel
openclaw channels login --channel whatsapp --account work  # Multi-account

openclaw pairing list <channel>   # List pending pairings
openclaw pairing approve <channel> <code>  # Approve pairing
```

## Device/UI Commands
```bash
openclaw dashboard                # Open Control UI
openclaw dashboard --no-open      # Just print URL
openclaw devices list             # List paired devices
openclaw devices approve <id>     # Approve device
```

## Config Commands
```bash
openclaw config get <path>        # Get config value
openclaw config set <path> <value>  # Set config value
openclaw config set <path> <value> --json  # Set JSON value
openclaw config unset <path>      # Remove config value
openclaw configure                # Config wizard
openclaw onboard                  # Full setup wizard
```

## Model Commands
```bash
openclaw models status            # Auth status
openclaw models status --check    # Exit code auth check
openclaw models auth setup-token --provider <name>  # Setup token
openclaw models auth paste-token --provider <name>  # Paste token
```

## Agent Commands
```bash
openclaw agent --message "Hello"  # Send to agent
openclaw agent --message "Hello" --thinking high  # With thinking
openclaw agents add <name>        # Create new agent
openclaw agents list              # List agents
```

## Message Commands
```bash
openclaw message send --to <target> --message "Hello"
```

## Update/Install Commands
```bash
openclaw update                   # Update (source install)
openclaw update --channel <stable|beta|dev>  # Switch channel
openclaw uninstall                # Uninstall
openclaw uninstall --all --yes    # Full uninstall
```

## Security Commands
```bash
openclaw security audit           # Security audit
openclaw security audit --deep    # Deep audit
openclaw security audit --fix     # Fix issues
openclaw security audit --json    # JSON output
```

## Cron Commands
```bash
openclaw cron status              # Cron status
openclaw cron list                # List jobs
openclaw cron runs --id <id> --limit 20  # Job history
```

---

# DOCTOR COMMAND DETAILS

`openclaw doctor` is the essential repair/migration tool. It:

1. **Config Normalization**: Migrates deprecated config keys
2. **Legacy State Migration**: Moves old sessions/credentials to new paths
3. **State Integrity**: Checks permissions, missing files
4. **Model Auth Health**: Checks OAuth expiry, can refresh tokens
5. **Sandbox Image Repair**: Rebuilds Docker images if needed
6. **Service Migration**: Updates launchd/systemd configs
7. **Security Warnings**: Flags risky DM policies
8. **Channel Status**: Probes channel connectivity

**Current Migrations Handled:**
- `routing.allowFrom` → `channels.whatsapp.allowFrom`
- `routing.groupChat.*` → `channels.<provider>.groups.*`
- `routing.bindings` → top-level `bindings`
- `agent.*` → `agents.defaults` + `tools.*`
- `identity` → `agents.list[].identity`
- Legacy WhatsApp credentials to new path structure

**When to Run:**
- After every update
- When config validation fails
- When channels stop working
- When gateway won't start
- Periodically for health checks

---

# PATH & ENVIRONMENT REFERENCE

## Path References

| Path | Purpose |
|------|---------|
| `~/.openclaw/` | State directory |
| `~/.openclaw/openclaw.json` | Main config file (JSON5) |
| `~/.openclaw/.env` | Environment variables |
| `~/.openclaw/workspace/` | Default workspace |
| `~/.openclaw/credentials/` | Channel credentials |
| `~/.openclaw/credentials/whatsapp/<id>/` | WhatsApp auth state |
| `~/.openclaw/sessions/` | Session transcripts (legacy) |
| `~/.openclaw/agents/<id>/sessions/` | Per-agent sessions |
| `~/.openclaw/skills/` | Managed/local skills |

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `OPENCLAW_HOME` | Override home directory for all path resolution |
| `OPENCLAW_STATE_DIR` | Override state directory (default: `~/.openclaw`) |
| `OPENCLAW_CONFIG_PATH` | Override config file path |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token |
| `OPENCLAW_GATEWAY_PORT` | Gateway port (default: 18789) |
| `OPENCLAW_GATEWAY_PASSWORD` | Alternative to token auth |
| `OPENCLAW_LOAD_SHELL_ENV` | Import keys from login shell |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `OPENAI_API_KEY` | OpenAI API key |
| `GEMINI_API_KEY` | Google Gemini API key |
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `SLACK_BOT_TOKEN` | Slack bot token |
| `SLACK_APP_TOKEN` | Slack app token |

## Environment Precedence (highest → lowest)
1. Process environment
2. `./.env` (current directory)
3. `~/.openclaw/.env`
4. Config `env` block in `openclaw.json`
5. Shell env import (if enabled)

## Documentation Links

- **Main Docs**: https://docs.openclaw.ai
- **Getting Started**: https://docs.openclaw.ai/start/getting-started
- **Configuration**: https://docs.openclaw.ai/gateway/configuration
- **Configuration Reference**: https://docs.openclaw.ai/gateway/configuration-reference
- **Channels**: https://docs.openclaw.ai/channels
- **WhatsApp**: https://docs.openclaw.ai/channels/whatsapp
- **Telegram**: https://docs.openclaw.ai/channels/telegram
- **Discord**: https://docs.openclaw.ai/channels/discord
- **Slack**: https://docs.openclaw.ai/channels/slack
- **Signal**: https://docs.openclaw.ai/channels/signal
- **Security**: https://docs.openclaw.ai/gateway/security
- **Authentication**: https://docs.openclaw.ai/gateway/authentication
- **Docker**: https://docs.openclaw.ai/install/docker
- **Wizard**: https://docs.openclaw.ai/start/wizard
- **Troubleshooting Hub**: https://docs.openclaw.ai/help/troubleshooting
- **Gateway Troubleshooting**: https://docs.openclaw.ai/gateway/troubleshooting
- **Channel Troubleshooting**: https://docs.openclaw.ai/channels/troubleshooting
- **Doctor**: https://docs.openclaw.ai/gateway/doctor
- **Updating**: https://docs.openclaw.ai/install/updating
- **Uninstall**: https://docs.openclaw.ai/install/uninstall
- **Environment Variables**: https://docs.openclaw.ai/help/environment
- **Discord Community**: https://discord.gg/clawd
