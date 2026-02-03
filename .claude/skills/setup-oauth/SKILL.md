---
name: setup-oauth
description: Configure OAuth authentication for NanoClaw using Claude subscription (Pro/Teams/Max). Use when setting up authentication with CLAUDE_CODE_OAUTH_TOKEN. Triggers on "setup oauth", "oauth token", "setup token", or authentication issues on macOS Tahoe.
---

# OAuth Authentication Setup

This skill configures NanoClaw to use your Claude subscription (Pro/Teams/Max) instead of an Anthropic API key.

## When to Use OAuth vs API Key

| Method | When to Use |
|--------|-------------|
| **OAuth Token** | Claude Pro/Teams/Max subscriber, no API billing needed |
| **API Key** | Pay-per-use, organizational accounts, no Claude subscription |

OAuth tokens are valid for 1 year and use your existing Claude subscription quota.

## Prerequisites

- Active Claude Pro, Teams, or Max subscription
- Claude Code CLI installed (`claude --version`)
- Logged into Claude Code at least once

## Setup Methods

### Method 1: Extract from Credentials File (Standard)

Works when `~/.claude/.credentials.json` exists:

```bash
TOKEN=$(cat ~/.claude/.credentials.json 2>/dev/null | jq -r '.claudeAiOauth.accessToken // empty')
if [ -n "$TOKEN" ]; then
  grep -q "^CLAUDE_CODE_OAUTH_TOKEN=" .env 2>/dev/null && \
    sed -i '' "s|^CLAUDE_CODE_OAUTH_TOKEN=.*|CLAUDE_CODE_OAUTH_TOKEN=$TOKEN|" .env || \
    echo "CLAUDE_CODE_OAUTH_TOKEN=$TOKEN" >> .env
  echo "OAuth token configured: ${TOKEN:0:20}...${TOKEN: -4}"
else
  echo "No token found - try Method 2"
fi
```

### Method 2: Generate Setup Token (macOS Tahoe+ / Missing Credentials)

On macOS Tahoe and newer, the credentials file may not exist. Use `claude --setup-token` instead:

**Step 1:** Tell the user to run in a separate terminal:
```bash
claude --setup-token
```

This outputs a long-lived token starting with `sk-ant-oat01-`.

**Step 2:** Once they provide the token, configure it:
```bash
TOKEN="sk-ant-oat01-..."  # User-provided token
grep -q "^CLAUDE_CODE_OAUTH_TOKEN=" .env 2>/dev/null && \
  sed -i '' "s|^CLAUDE_CODE_OAUTH_TOKEN=.*|CLAUDE_CODE_OAUTH_TOKEN=$TOKEN|" .env || \
  echo "CLAUDE_CODE_OAUTH_TOKEN=$TOKEN" >> .env
echo "OAuth token configured"
```

**Note:** This is the recommended approach per [upstream PR #50](https://github.com/gavrielc/nanoclaw/pull/50) which addresses [issue #41](https://github.com/gavrielc/nanoclaw/issues/41).

### Method 3: Manual Configuration

If neither method works, the user can manually add to `.env`:

```bash
# .env
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-...
```

## Verification

Verify the token is configured correctly:

```bash
# Check .env has the token
grep "^CLAUDE_CODE_OAUTH_TOKEN=" .env && echo "Token present in .env" || echo "Token MISSING"

# Check token format
TOKEN=$(grep "^CLAUDE_CODE_OAUTH_TOKEN=" .env | cut -d= -f2)
if [[ "$TOKEN" == sk-ant-oat01-* ]]; then
  echo "Token format: Valid (sk-ant-oat01-*)"
else
  echo "Token format: Unexpected (should start with sk-ant-oat01-)"
fi
```

Test that the container can access the token:

```bash
# For Apple Container
container run -i --entrypoint /bin/sh nanoclaw-agent:latest \
  -c 'source /workspace/env-dir/env 2>/dev/null; echo "Token length: ${#CLAUDE_CODE_OAUTH_TOKEN}"'

# For Docker
docker run -i --env-file .env --entrypoint /bin/sh nanoclaw-agent:latest \
  -c 'echo "Token length: ${#CLAUDE_CODE_OAUTH_TOKEN}"'
```

## Troubleshooting

**"No token found" with Method 1:**
- Ensure you're logged into Claude Code: run `claude` and complete login
- On macOS Tahoe+, use Method 2 instead

**"claude --setup-token" not recognized:**
- Update Claude Code: `claude update`
- Requires Claude Code CLI version with setup-token support

**Token not working in container:**
- Only `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY` are passed to containers (security measure)
- Check token is in `.env` at the project root
- Rebuild container if env mounting logic changed: `./container/build.sh`

**"Authentication failed" in agent:**
- Token may be expired (1 year validity)
- Regenerate with `claude --setup-token`

## Security Notes

- OAuth tokens are extracted from `.env` and mounted into containers
- Tokens are never logged or exposed to agents beyond the env variable
- Only authentication variables are passed to containers (see `src/container-runner.ts`)

## Related

- `/setup` - Full NanoClaw setup including auth
- `/debug` - Troubleshoot container and auth issues
- [Upstream PR #50](https://github.com/gavrielc/nanoclaw/pull/50) - Adds setup-token flow to main setup skill
- [Upstream Issue #41](https://github.com/gavrielc/nanoclaw/issues/41) - Original report about missing credentials on Tahoe
