---
module: Gateway
date: 2026-01-26
problem_type: security_issue
component: authentication
symptoms:
  - "Gateway still requires token auth after removing gateway.auth from clawdbot.json"
  - "disconnected (1008): unauthorized: gateway token missing"
  - "disconnected (1008): pairing required after providing token"
root_cause: config_error
resolution_type: config_change
severity: high
tags: [gateway, authentication, cloudflare-tunnel, env-vars, control-ui]
---

# Troubleshooting: Gateway Token Active Despite Config Removal + Control UI Pairing Error

## Problem
After removing `gateway.auth` from clawdbot.json, the gateway still required token authentication because `CLAWDBOT_GATEWAY_TOKEN` was set in `.env`. After fixing the token issue, the Control UI showed "pairing required" when accessed via Cloudflare Tunnel.

## Environment
- Module: Gateway
- Clawdbot Version: 2026.1.24-2
- Affected Component: Gateway authentication, Control UI
- Date: 2026-01-26

## Symptoms
- Gateway requires token auth even after removing `gateway.auth` from config
- Error: `disconnected (1008): unauthorized: gateway token missing`
- After providing token via URL parameter: `disconnected (1008): pairing required`
- Control UI inaccessible via Cloudflare Tunnel (https://echo.mili.dev)

## What Didn't Work

**Attempted Solution 1:** Removing `gateway.auth` section from clawdbot.json
- **Why it failed:** The gateway auth resolution reads from multiple sources with fallback:
  ```typescript
  const token = authConfig.token ?? env.CLAWDBOT_GATEWAY_TOKEN ?? undefined;
  const mode = authConfig.mode ?? (password ? "password" : token ? "token" : "none");
  ```
  The env var `CLAWDBOT_GATEWAY_TOKEN` in `.env` was still being read as fallback.

**Attempted Solution 2:** Adding `?token=<token>` to the URL
- **Why it failed:** Token auth worked, but then hit "pairing required" error. The Control UI requires device identity verification by default when accessed via HTTPS.

## Solution

**Three configuration changes required:**

### 1. Remove env var from `.env`

```bash
# Before (.env):
CLAWDBOT_GATEWAY_TOKEN=ec1ac66cf55205f5fb6035307e5a5e42e7b82048388649cb18e94c09d9fe7cfe

# After (.env):
# Line removed entirely
```

### 2. Add gateway.auth to clawdbot.json (with new token)

```json
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "<new-generated-token>"
    }
  }
}
```

Generate new token: `openssl rand -hex 32`

### 3. Add controlUi.allowInsecureAuth for tunnel access

```json
{
  "gateway": {
    "controlUi": {
      "allowInsecureAuth": true
    },
    "auth": {
      "mode": "token",
      "token": "<token>"
    }
  }
}
```

This disables device identity + pairing for the Control UI (safe when behind Cloudflare Access).

### 4. Restart gateway

```bash
docker compose restart clawdbot-gateway
```

## Why This Works

1. **Config layering**: Clawdbot reads config from multiple sources (json, env vars) with fallbacks. The env var takes precedence over missing json config. Removing the env var ensures only json config is used.

2. **Control UI pairing**: By default, Clawdbot requires device identity verification for the Control UI to prevent unauthorized access. When accessing via Cloudflare Tunnel:
   - The connection appears as HTTP to the gateway (tunnel terminates HTTPS)
   - Device identity APIs may not work through the tunnel
   - `allowInsecureAuth: true` bypasses this since Cloudflare Access provides authentication

3. **Security layers**: With this setup, you have:
   - Ports bound to 127.0.0.1 only (no direct internet access)
   - Cloudflare Tunnel (outbound-only, no exposed ports)
   - Cloudflare Access (sign-in required)
   - Gateway token authentication

## Prevention

- When troubleshooting auth issues, check both `.env` AND `clawdbot.json` for config values
- Understand the config resolution order: env vars can override json config
- For tunnel/proxy setups, check if `controlUi.allowInsecureAuth` is needed
- Document: https://docs.clawd.bot/web/control-ui explains pairing requirements
- Use `clawdbot config get gateway` to see effective config values

## Related Issues

No related issues documented yet.
