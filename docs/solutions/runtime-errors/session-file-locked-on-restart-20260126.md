---
module: Gateway
date: 2026-01-26
problem_type: runtime_error
component: background_job
symptoms:
  - "session file locked (timeout 10000ms)"
  - "Agent failed before reply"
  - "Error occurs after gateway restart"
root_cause: async_timing
resolution_type: workflow_improvement
severity: medium
tags: [session-lock, restart, stale-lock, gateway]
---

# Troubleshooting: Session File Locked After Gateway Restart

## Problem
After restarting the gateway (via `docker compose restart` or config reload), Telegram messages fail with "session file locked" errors. The lock file persists from the previous process and isn't cleaned up.

## Environment
- Module: Gateway
- Clawdbot Version: 2026.1.24-2
- Affected Component: Session management
- Date: 2026-01-26

## Symptoms
- Error: `session file locked (timeout 10000ms): pid=7 /home/node/.clawdbot/agents/main/sessions/<session-id>.jsonl.lock`
- Telegram bot shows: `⚠️ Agent failed before reply: session file locked`
- Occurs immediately after gateway restart or config reload
- Multiple messages all fail with same error

## What Didn't Work

**Waiting for timeout**: The lock has a 10-second timeout, but the lock file itself persists indefinitely if not cleaned up.

## Solution

Clear stale lock files before or after restart:

```bash
docker exec clawdbot-clawdbot-gateway-1 find /home/node/.clawdbot/agents -name "*.lock" -delete 2>/dev/null || true
```

This has been integrated into:
- `/clawdbot-commit` skill (Step 7c)
- `/clawdbot-update` skill (Step 7c)

## Why This Works

1. Lock files are created when a session is being written to
2. During restart, the process is killed before it can release the lock
3. The new process sees the stale lock file and waits for timeout
4. Deleting the lock file allows the new process to acquire fresh locks

The lock files are safe to delete because:
- The old process (that created them) is dead
- The new process will create fresh locks as needed
- Session data is in the `.jsonl` file, not the `.lock` file

## Prevention

The fix is now integrated into restart workflows:
- `/clawdbot-commit` clears locks before restarting
- `/clawdbot-update` clears locks before restarting

For manual restarts, run the cleanup command before `docker compose up -d`.

## Related Issues

No related issues documented yet.
