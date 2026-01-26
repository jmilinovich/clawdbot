---
module: Gateway
date: 2026-01-26
problem_type: developer_experience
component: tooling
symptoms:
  - "Permission denied when editing files created by container"
  - "Files in ~/.clawdbot owned by root"
  - "Files in ~/clawd/skills owned by root"
  - "Need sudo to edit workspace files"
root_cause: config_error
resolution_type: config_change
severity: medium
tags: [docker, permissions, user, root, ownership]
---

# Troubleshooting: Container Creates Files as Root

## Problem
The clawdbot Docker container runs as root by default, so any files it creates (configs, session files, skills) are owned by root on the host. This causes permission errors when trying to edit them.

## Environment
- Module: Gateway / Docker
- Clawdbot Version: 2026.1.24-2
- Affected Component: All mounted volumes
- Date: 2026-01-26

## Symptoms
- `Permission denied` when editing files in `~/.clawdbot/` or `~/clawd/`
- `ls -la` shows files owned by `root:root`
- Need to run `sudo chown` frequently
- Edit tool fails with `EACCES: permission denied`

## Solution

### 1. Add user directive to docker-compose.yml

```yaml
services:
  clawdbot-gateway:
    image: ${CLAWDBOT_IMAGE:-clawdbot:local}
    user: "1002:1002"  # Replace with your UID:GID
    # ... rest of config
```

Get your UID/GID with: `id $USER`

### 2. Fix existing file ownership

```bash
sudo chown -R $USER:$USER ~/.clawdbot ~/clawd
```

### 3. Restart the container

```bash
cd ~/clawdbot
docker compose up -d
```

## Why This Works

- The `user:` directive tells Docker to run the container process as that UID/GID
- Files created by the container will now be owned by your user
- No more permission conflicts between host and container
- The container still has full access to mounted volumes

## Verification

```bash
# Check container is running as your user
docker exec clawdbot-clawdbot-gateway-1 id
# Should show: uid=1002 gid=1002 (or your UID/GID)

# Create a test file and check ownership
docker exec clawdbot-clawdbot-gateway-1 touch /home/node/.clawdbot/test-file
ls -la ~/.clawdbot/test-file
# Should show your username, not root
```

## Prevention

Always include `user: "UID:GID"` in docker-compose.yml when mounting host directories that you'll need to edit from the host.

## Related Issues

- See also: [session-file-locked-on-restart](../runtime-errors/session-file-locked-on-restart-20260126.md)
- See also: [jq-not-found-in-container](../runtime-errors/jq-not-found-in-container-20260126.md)
