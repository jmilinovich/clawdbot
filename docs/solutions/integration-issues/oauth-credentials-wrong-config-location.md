---
title: "OAuth Token Stored in Wrong Location Bypasses Auth System"
slug: oauth-credentials-wrong-config-location
date: 2026-01-26
category: integration-issues
tags:
  - oauth
  - api-key
  - auth-profiles
  - configuration
  - environment-variables
  - credentials
module: auth-profiles
symptoms:
  - Bot stops working after disabling API key
  - OAuth flow completed but agent still uses old credentials
  - Agent ignores OAuth token despite successful authentication
severity: high
---

# OAuth Token Stored in Wrong Location Bypasses Auth System

## Problem

After completing an OAuth authentication flow, the bot continued using an old API key instead of the new OAuth credentials. Disabling the API key caused the bot to stop working entirely, even though OAuth authentication had been completed successfully.

## Root Cause

The OAuth access token (`sk-ant-oat01-...`) was stored in the wrong location:

- **Wrong:** `clawdbot.json` under `env.ANTHROPIC_API_KEY`
- **Correct:** `~/.clawdbot/auth-profiles.json` as an OAuth/token credential

When stored in the `env` section of `clawdbot.json`, the token gets loaded into `process.env` and appears to the system as a regular API key—not as an OAuth credential. This causes problems because:

1. The token isn't recognized as OAuth and may not be refreshed properly
2. If a different (disabled) API key exists in `auth-profiles.json`, confusion arises about which credential to use
3. The OAuth token's special handling is bypassed entirely

## Solution

### Step 1: Remove OAuth token from clawdbot.json

Edit `~/.clawdbot/clawdbot.json` and remove `ANTHROPIC_API_KEY` from the `env` section:

**Before:**
```json
{
  "env": {
    "ANTHROPIC_API_KEY": "sk-ant-oat01-xxxxxxxxxxxx",
    "OTHER_VAR": "some-value"
  }
}
```

**After:**
```json
{
  "env": {
    "OTHER_VAR": "some-value"
  }
}
```

### Step 2: Create proper auth-profiles.json

Create or update `~/.clawdbot/auth-profiles.json` with the OAuth token stored correctly:

```json
{
  "version": 1,
  "profiles": {
    "anthropic:oauth": {
      "type": "token",
      "provider": "anthropic",
      "token": "sk-ant-oat01-xxxxxxxxxxxx"
    }
  },
  "order": {
    "anthropic": ["anthropic:oauth"]
  }
}
```

For full OAuth with refresh capability (if you have a refresh token):

```json
{
  "version": 1,
  "profiles": {
    "anthropic:oauth": {
      "type": "oauth",
      "provider": "anthropic",
      "access": "sk-ant-oat01-xxxxxxxxxxxx",
      "refresh": "sk-ant-ort01-xxxxxxxxxxxx",
      "expires": 1735689600000
    }
  },
  "order": {
    "anthropic": ["anthropic:oauth"]
  }
}
```

### Step 3: Update agent-specific auth profiles

If your agent has its own auth profile at `~/.clawdbot/agents/<agent-id>/agent/auth-profiles.json`, update it similarly:

```json
{
  "version": 1,
  "profiles": {
    "anthropic:oauth": {
      "type": "token",
      "provider": "anthropic",
      "token": "sk-ant-oat01-xxxxxxxxxxxx"
    }
  },
  "order": {
    "anthropic": ["anthropic:oauth"]
  },
  "lastGood": {
    "anthropic": "anthropic:oauth"
  }
}
```

### Why This Works

1. **Correct credential type recognition**: The auth system checks `auth-profiles.json` before environment variables
2. **Proper precedence**: Auth profiles take priority, so the OAuth token will be used instead of any stale API keys
3. **Token prefix validation**: The `sk-ant-oat01-` prefix indicates an OAuth Access Token; when stored properly, the system can handle it correctly

## Prevention

### Best Practices

1. **Use dedicated credential stores, not environment variables**
   - OAuth tokens belong in `auth-profiles.json`, not `env.ANTHROPIC_API_KEY`
   - Environment variables are for static API keys, not OAuth tokens that expire and refresh

2. **Validate storage location after OAuth flow**
   - After OAuth callback, verify the token was written to `auth-profiles.json`
   - Check that no duplicate exists in `clawdbot.json` env section

3. **Use the proper auth commands**
   - Run `clawdbot models auth login --provider anthropic` for OAuth setup
   - Use `clawdbot models auth setup-token` to sync with Claude Code CLI

### Warning Signs

| Symptom | Likely Cause |
|---------|--------------|
| Auth works initially, then fails after token refresh | Token stored in location that doesn't support refresh |
| "Invalid API key" errors with valid OAuth session | Reading from `env.ANTHROPIC_API_KEY` instead of auth-profiles |
| Different behavior between sessions | Token duplicated across multiple storage locations |

### Diagnostic Commands

```bash
# Check auth-profiles.json exists and has correct structure
cat ~/.clawdbot/auth-profiles.json | jq '.profiles | keys'

# Verify no OAuth tokens in env config
cat ~/.clawdbot/clawdbot.json | jq '.env | has("ANTHROPIC_API_KEY")'
# Should return false for OAuth-only setups

# Check agent-specific auth profiles
cat ~/.clawdbot/agents/main/agent/auth-profiles.json | jq '.profiles | keys'

# Run doctor to check auth status
clawdbot doctor
```

## Related Documentation

- [OAuth Architecture](/concepts/oauth) - Token storage and refresh flow
- [Gateway Authentication](/gateway/authentication) - Auth setup guide
- [Configuration Guide](/gateway/configuration) - Auth-profiles configuration
- [Gateway Token Env Override Issue](/solutions/security-issues/gateway-token-env-override-20260126) - Similar config precedence issue

## Key Insight

The authentication system has a clear precedence order:

1. Explicit profile ID (if specified)
2. Auth profiles (`auth-profiles.json`) - checked in configured order
3. Environment variables (including those from `clawdbot.json` env section)
4. Custom provider config

OAuth tokens should always be stored in auth-profiles (layer 2), not in environment variables (layer 3). Storing them in the wrong layer causes the system to treat them as static API keys, losing OAuth-specific functionality like token refresh and proper error handling.
