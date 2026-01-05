# AI Toolkit

A personal collection of AI agent configurations, prompts, and automation tools for everyday productivity.

## 📁 Structure

```
ai-toolkit/
├── cursor-commands/     # Cursor AI slash commands
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
---

## License

MIT
