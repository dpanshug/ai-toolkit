---
description: Check OpenClaw Docker setup health and auto-fix common issues (gateway, pairing, Telegram, Claude proxy)
---

You are diagnosing and fixing the local OpenClaw Docker setup. Run all checks, report findings, and auto-fix what you can. Be concise in reporting.

## Step 1: Docker containers

Check that required containers are running:

```bash
docker ps --format '{{.Names}} {{.Status}}' | grep -E "openclaw|claude-proxy"
```

If `openclaw-openclaw-gateway-1` or `openclaw-claude-proxy-1` are missing or exited, fix by running:

```bash
cd ~/openclaw && docker compose up -d
```

Wait 10 seconds after starting, then re-check.

## Step 2: Gateway health endpoint

```bash
curl -s --max-time 5 http://127.0.0.1:18789/healthz
```

Expected: `{"ok":true,"status":"live"}`. If unreachable, the container may still be starting — wait and retry once. If still failing, check container logs:

```bash
docker logs openclaw-openclaw-gateway-1 --tail 30
```

## Step 3: Device pairing

Read the pending pairing state:

```bash
cat ~/.openclaw/devices/pending.json
```

If the file contains a pending request (not empty `{}`), check:
- Does the pending `deviceId` match an entry in `~/.openclaw/devices/paired.json`?
- Does the pending `publicKey` match the paired entry's `publicKey`?
- Is `isRepair` true?

If ALL three are true, this is a safe re-approval (same device, key unchanged, just metadata drift like platform). Fix it:

```python
python3 -c "
import json, os

pending_path = os.path.expanduser('~/.openclaw/devices/pending.json')
paired_path = os.path.expanduser('~/.openclaw/devices/paired.json')

with open(pending_path) as f:
    pending = json.load(f)
with open(paired_path) as f:
    paired = json.load(f)

fixed = False
for req_id, req in list(pending.items()):
    device_id = req.get('deviceId', '')
    if device_id in paired and paired[device_id].get('publicKey') == req.get('publicKey') and req.get('isRepair'):
        # Remove platform field entirely to prevent flip-flop between host (darwin) and container (linux)
        if 'platform' in paired[device_id]:
            del paired[device_id]['platform']
        fixed = True

if fixed:
    with open(pending_path, 'w') as f:
        json.dump({}, f)
    with open(paired_path, 'w') as f:
        json.dump(paired, f, indent=2)
    print('FIXED: Cleared stale pending request and removed platform field to prevent host/container flip-flop.')
else:
    if pending:
        print('WARNING: Pending pairing request exists but does not look auto-fixable. Manual approval needed.')
        print('Run: pnpm openclaw devices list')
    else:
        print('OK: No pending pairing requests.')
"
```

## Step 4: Gateway connectivity via CLI

```bash
cd ~/openclaw && pnpm openclaw status 2>&1 | grep -E "Gateway|reachable|unreachable"
```

Verify the output shows "reachable". If it says "unreachable" after Step 3 fix, restart the gateway:

```bash
cd ~/openclaw && docker compose restart openclaw-gateway
```

Wait 15 seconds, then re-check status.

## Step 5: Telegram bot

Check config:

```bash
python3 -c "
import json, os
with open(os.path.expanduser('~/.openclaw/openclaw.json')) as f:
    cfg = json.load(f)
tg = cfg.get('channels', {}).get('telegram', {})
if not tg.get('enabled'):
    print('ISSUE: Telegram channel is disabled in config.')
elif not tg.get('botToken'):
    print('ISSUE: Telegram botToken is missing from config.')
else:
    print('OK: Telegram configured, token present.')
"
```

Check gateway logs for Telegram connection status:

```bash
docker logs openclaw-openclaw-gateway-1 --tail 100 2>&1 | grep -i "telegram" | tail -5
```

If logs show connection errors or "polling stopped", restart the gateway:

```bash
cd ~/openclaw && docker compose restart openclaw-gateway && sleep 10 && docker logs openclaw-openclaw-gateway-1 --tail 20 2>&1 | grep -i telegram
```

## Step 6: Claude proxy

Verify the proxy container is running and the gateway can reach it:

```bash
docker exec openclaw-openclaw-gateway-1 node -e "fetch('http://claude-proxy:18800/').then(r=>{console.log('Claude proxy reachable, status:',r.status)}).catch(e=>console.log('UNREACHABLE:',e.message))"
```

If unreachable, check if the proxy container is on the same Docker network:

```bash
docker network inspect openclaw_default --format '{{range .Containers}}{{.Name}} {{end}}' 2>/dev/null
```

If the proxy is missing from the network, restart it:

```bash
cd ~/openclaw && docker compose up -d openclaw-claude-proxy
```

## Step 7: Google OAuth (gog) token health

Verify gog can authenticate inside the container:

```bash
docker exec openclaw-openclaw-gateway-1 /bin/sh -c 'unset XDG_CONFIG_HOME; /home/node/.openclaw/bin/gog auth list'
```

Expected: shows `dipgupta@redhat.com` with `calendar,gmail` scopes. If it shows "No tokens stored" or errors:

1. Check the volume mount exists: `docker inspect openclaw-openclaw-gateway-1 --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' | grep gogcli`
2. If mount is missing, ensure `docker-compose.override.yml` has the gogcli volume line and run `docker compose up -d`
3. If mount exists but token expired, re-auth from host:

```bash
gog auth add dipgupta@redhat.com --services gmail,calendar --readonly --force-consent
gog auth tokens export dipgupta@redhat.com --out ~/.openclaw/gogcli/token.json --overwrite
docker exec -e XDG_CONFIG_HOME=/home/node/.openclaw -e GOG_KEYRING_PASSWORD=openclaw-local openclaw-openclaw-gateway-1 /home/node/.openclaw/bin/gog auth tokens import /home/node/.openclaw/gogcli/token.json
```

Then verify with a live call:

```bash
docker exec openclaw-openclaw-gateway-1 /bin/sh -c 'unset XDG_CONFIG_HOME; /home/node/.openclaw/bin/gog gmail messages list "in:inbox" --account dipgupta@redhat.com --limit 1'
```

If this returns mail, report OK. If it errors with 401/403, the token is expired and needs re-auth (instructions above).

## Step 8: Summary

Report a concise summary table:

| Check | Status |
|-------|--------|
| Docker containers | ... |
| Gateway health | ... |
| Device pairing | ... |
| Gateway connectivity | ... |
| Telegram bot | ... |
| Claude proxy | ... |

Use OK/FIXED/WARNING/FAILED for each. If everything is OK or FIXED, confirm OpenClaw is ready to use.
