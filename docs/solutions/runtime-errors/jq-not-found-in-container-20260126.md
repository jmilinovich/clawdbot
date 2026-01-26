---
module: Gateway
date: 2026-01-26
problem_type: runtime_error
component: tooling
symptoms:
  - "exec failed: ./call.sh: line 33: jq: command not found"
  - "Command exited with code 127"
  - "Skills using shell scripts fail when they need jq"
root_cause: missing_tooling
resolution_type: code_fix
severity: medium
tags: [jq, shell-script, container, skills, json-parsing]
---

# Troubleshooting: jq Not Found in Container

## Problem
Skills that use shell scripts with `jq` for JSON parsing fail because `jq` is not installed in the clawdbot Docker container.

## Environment
- Module: Gateway / Skills
- Clawdbot Version: 2026.1.24-2
- Affected Component: Shell-based skills (e.g., phone-call)
- Date: 2026-01-26

## Symptoms
- Error: `./call.sh: line 33: jq: command not found`
- Exit code 127 (command not found)
- Skills that call external APIs and parse JSON responses fail

## Solution

Replace `jq` with `node` for JSON parsing. Node.js is always available in the container.

**Before (broken):**
```bash
echo "$response" | jq -r '.status'
```

**After (fixed):**
```bash
echo "$response" | node -e "
const data = JSON.parse(require('fs').readFileSync(0, 'utf8'));
if (data.status === 'success' || data.call_id) {
  console.log('✅ Call initiated!');
  console.log('Call ID:', data.call_id);
} else {
  console.log('❌ Error:', data.message || data.error || JSON.stringify(data));
}
"
```

## Why This Works

- The clawdbot container is Node.js-based, so `node` is always available
- Node can read stdin via `require('fs').readFileSync(0, 'utf8')`
- `JSON.parse()` handles the same parsing that `jq` would do
- No need to modify the Docker image or install additional packages

## Prevention

When writing shell scripts for clawdbot skills:
- Don't assume `jq`, `python`, or other tools are available
- Use `node` for JSON parsing - it's guaranteed to be present
- Or write the skill in JavaScript/TypeScript instead of shell

## Related Issues

- See also: [container-runs-as-root-permission-issues](../developer-experience/container-runs-as-root-20260126.md)
