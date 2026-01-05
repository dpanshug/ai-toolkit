# Cursor Commands

Personal Cursor AI commands for productivity.

## Installation

Symlink to your Cursor commands directory:

```bash
ln -sf ~/Documents/cursor-commands/*.md ~/.cursor/commands/
```

## Available Commands

### `/checkout-pr`
Checkout a GitHub PR branch locally.

**Usage:** `/checkout-pr` → Enter PR number

### `/oc-login`
Login to OpenShift cluster using credentials from `.env.local`.

**Usage:** `/oc-login` → Enter cluster name (e.g., `ROSA`)

Clusters are defined in `.env.local` with section headers like:
```
# ROSA cluster
OC_URL=https://...
OC_USER=...
OC_PASSWORD=...
```
