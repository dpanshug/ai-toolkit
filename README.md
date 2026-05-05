# AI Toolkit

A personal collection of AI agent configurations, prompts, and automation tools for everyday productivity.

## 📁 Structure

```
ai-toolkit/
├── cursor-commands/     # Cursor AI slash commands
├── openclaw/            # OpenClaw Docker setup docs
└── ...                  # More to come
```

---

## 🎯 Cursor Commands

Custom slash commands for [Cursor](https://cursor.sh) AI IDE.

### Installation

Symlink all commands to your Cursor commands directory:

```bash
ln -sf ~/Documents/projects/ai-toolkit/cursor-commands/*.md ~/.cursor/commands/
```

### Available Commands

| Command | Description |
|---------|-------------|
| `/checkout-pr` | Checkout a GitHub PR branch locally |
| `/oc-login` | Login to OpenShift cluster from `.env.local` credentials |
| `/openclaw-doctor` | Check OpenClaw Docker health and auto-fix common issues |

#### `/checkout-pr`

Quickly checkout any GitHub PR to your local machine.

**Usage:** `/checkout-pr` → Enter PR number

#### `/oc-login`

Login to OpenShift clusters using credentials stored in `.env.local`.

**Usage:** `/oc-login` → Enter cluster name (e.g., `ROSA`)

Clusters are defined in your `.env.local` with section headers:
```ini
# ROSA cluster
OC_URL=https://...
OC_USER=...
OC_PASSWORD=...
```
#### `/openclaw-doctor`

Full health check of the local OpenClaw Docker setup: containers, gateway, device pairing, Telegram, Claude proxy, and Google OAuth tokens. Auto-fixes common issues.

**Usage:** `/openclaw-doctor`

---

## 🦞 OpenClaw Setup

Local OpenClaw setup running via Docker with Telegram bot, Claude proxy, and Google Workspace (Gmail + Calendar) integration.

### Prerequisites

- Docker Desktop (set to launch at login)
- OpenClaw repo cloned at `~/openclaw`
- `gog` CLI ([steipete/gogcli](https://github.com/steipete/gogcli)) installed on host via Homebrew

### Architecture

```
┌─ Docker ───────────────────────────────────────────┐
│  openclaw-gateway (port 18789)                     │
│    ├── Agent runtime (Claude Sonnet 4)             │
│    ├── Telegram bot (@dip_openclaw_bot)            │
│    ├── gog binary (Linux arm64) → Gmail/Calendar   │
│    └── connects to claude-proxy:18800              │
│                                                    │
│  claude-proxy (port 18800)                         │
│    └── Anthropic API proxy via Red Hat GCP         │
└────────────────────────────────────────────────────┘
     │
     ├── ~/.openclaw/           (bind mount → /home/node/.openclaw)
     ├── ~/.openclaw/workspace/ (bind mount → workspace)
     └── ~/.openclaw/gogcli/    (bind mount → /home/node/.config/gogcli)
```

### Key files

| File | Purpose |
|------|---------|
| `~/openclaw/docker-compose.yml` | Main compose (from repo, don't edit) |
| `~/openclaw/docker-compose.override.yml` | Local overrides (gitignored) |
| `~/.openclaw/openclaw.json` | Main config (models, channels, gateway) |
| `~/.openclaw/workspace/USER.md` | Agent's knowledge about you |
| `~/.openclaw/workspace/TOOLS.md` | Available tools (gog commands) |
| `~/.openclaw/workspace/AGENTS.md` | Session startup behavior |
| `~/.openclaw/bin/gog` | Linux arm64 gog binary for container |
| `~/.openclaw/gogcli/` | Google OAuth credentials + tokens |
| `~/.openclaw/devices/` | Device pairing state |

### Fresh machine setup

```bash
# 1. Clone OpenClaw
git clone https://github.com/openclaw/openclaw ~/openclaw
cd ~/openclaw

# 2. Create docker-compose.override.yml
cat > docker-compose.override.yml << 'EOF'
services:
  openclaw-gateway:
    environment:
      GOG_KEYRING_PASSWORD: ${GOG_KEYRING_PASSWORD:-openclaw-local}
    volumes:
      - ${HOME:-/tmp}/.openclaw/gogcli:/home/node/.config/gogcli
EOF

# 3. Start containers
docker compose up -d

# 4. Install gog (host) and download Linux binary for container
brew install steipete/tap/gog
mkdir -p ~/.openclaw/bin
gh release download --repo openclaw/gogcli --pattern "gogcli_*_linux_arm64.tar.gz" --dir /tmp/ --clobber
tar -xzf /tmp/gogcli_*_linux_arm64.tar.gz -C ~/.openclaw/bin/
chmod +x ~/.openclaw/bin/gog

# 5. Authorize Google (opens browser)
gog auth add your-email@example.com --services gmail,calendar --readonly

# 6. Set up file-based keyring for container
mkdir -p ~/.openclaw/gogcli
cp ~/Library/Application\ Support/gogcli/credentials.json ~/.openclaw/gogcli/
gog auth tokens export your-email@example.com --out ~/.openclaw/gogcli/token.json --overwrite

# 7. Configure container's gog keyring
docker exec -e XDG_CONFIG_HOME=/home/node/.openclaw -e GOG_KEYRING_PASSWORD=openclaw-local \
  openclaw-openclaw-gateway-1 /home/node/.openclaw/bin/gog auth keyring file
docker exec -e XDG_CONFIG_HOME=/home/node/.openclaw -e GOG_KEYRING_PASSWORD=openclaw-local \
  openclaw-openclaw-gateway-1 /home/node/.openclaw/bin/gog auth tokens import /home/node/.openclaw/gogcli/token.json

# 8. Verify
docker exec openclaw-openclaw-gateway-1 /bin/sh -c \
  'unset XDG_CONFIG_HOME; /home/node/.openclaw/bin/gog gmail messages list "in:inbox" --account your-email@example.com --limit 1'
```

### Troubleshooting

**Device pairing deadlock** (gateway unreachable, "device metadata change pending approval"):
```bash
python3 -c "
import json, os
for path in ['pending.json', 'paired.json']:
    fp = os.path.expanduser(f'~/.openclaw/devices/{path}')
    with open(fp) as f: globals()[path.split('.')[0]] = json.load(f)
for did in paired:
    if 'platform' in paired[did]: del paired[did]['platform']
with open(os.path.expanduser('~/.openclaw/devices/pending.json'), 'w') as f: json.dump({}, f)
with open(os.path.expanduser('~/.openclaw/devices/paired.json'), 'w') as f: json.dump(paired, f, indent=2)
print('Fixed.')
"
```

**Google OAuth token expired** (gog returns 401):
```bash
gog auth add dipgupta@redhat.com --services gmail,calendar --readonly --force-consent
gog auth tokens export dipgupta@redhat.com --out ~/.openclaw/gogcli/token.json --overwrite
docker exec -e XDG_CONFIG_HOME=/home/node/.openclaw -e GOG_KEYRING_PASSWORD=openclaw-local \
  openclaw-openclaw-gateway-1 /home/node/.openclaw/bin/gog auth tokens import /home/node/.openclaw/gogcli/token.json
```

**Important note:** OpenClaw's security policy strips `XDG_CONFIG_HOME` from the agent's shell environment. That's why gog config must live at the default path (`~/.config/gogcli/` inside the container), mounted via docker-compose override.

---

## 🚀 Future Additions

- **MCP Servers** — Model Context Protocol server configurations
- **System Prompts** — Reusable prompts for various AI assistants
- **Workflows** — Multi-step automation recipes

---

## License

MIT
