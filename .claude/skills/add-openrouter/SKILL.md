---
name: add-openrouter
description: Add OpenRouter support to use alternative LLM providers via ANTHROPIC_BASE_URL
---

# Add OpenRouter Support

This skill configures NanoClaw to use OpenRouter as an LLM provider, enabling access to Claude and other models through OpenRouter's unified API. Addresses upstream issue #38.

## How It Works

The Claude Agent SDK (used in `container/agent-runner/src/index.ts`) respects the `ANTHROPIC_BASE_URL` environment variable. By pointing it at OpenRouter's Anthropic-compatible endpoint and swapping the API key, the agent uses OpenRouter without any code changes.

The key insight: **no source code modifications are needed**. This is purely an environment variable configuration change, plus updating the container runner's env allowlist to pass `ANTHROPIC_BASE_URL` into containers.

---

## Prerequisites

Ask the user:

> Do you already have an OpenRouter account and API key?

If not, guide them:

> 1. Go to [https://openrouter.ai](https://openrouter.ai) and create an account
> 2. Navigate to [https://openrouter.ai/keys](https://openrouter.ai/keys)
> 3. Click **"Create Key"**
> 4. Give it a name (e.g., "nanoclaw")
> 5. Copy the key - it starts with `sk-or-v1-...`

Wait for the user to provide their key.

---

## Step 1: Configure Environment Variables

Add two variables to `.env`:

```bash
# OpenRouter configuration
ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
ANTHROPIC_API_KEY=sk-or-v1-your-openrouter-key-here
```

If the user already has an `ANTHROPIC_API_KEY` for direct Anthropic access, warn them:

> **Note:** Setting `ANTHROPIC_API_KEY` to your OpenRouter key will replace your direct Anthropic authentication. If you want to switch back later, save your original key somewhere safe first.

To configure:

```bash
# Back up existing key if present
grep "^ANTHROPIC_API_KEY=" .env && echo "Backing up existing key..." && cp .env .env.backup

# Set OpenRouter base URL
grep -q "^ANTHROPIC_BASE_URL=" .env && \
  sed -i '' 's|^ANTHROPIC_BASE_URL=.*|ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1|' .env || \
  echo 'ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1' >> .env

# Set OpenRouter API key (replace with actual key from user)
grep -q "^ANTHROPIC_API_KEY=" .env && \
  sed -i '' "s|^ANTHROPIC_API_KEY=.*|ANTHROPIC_API_KEY=${OPENROUTER_KEY}|" .env || \
  echo "ANTHROPIC_API_KEY=${OPENROUTER_KEY}" >> .env
```

Verify:

```bash
grep -E "^(ANTHROPIC_BASE_URL|ANTHROPIC_API_KEY)" .env
```

Expected output:
```
ANTHROPIC_BASE_URL=https://openrouter.ai/api/v1
ANTHROPIC_API_KEY=sk-or-v1-...
```

---

## Step 2: Update Container Runner Env Allowlist

The container runner in `src/container-runner.ts` filters which environment variables are passed into agent containers. `ANTHROPIC_BASE_URL` must be added to the allowlist so the agent inside the container can reach OpenRouter.

Find this line in `src/container-runner.ts`:

```typescript
const allowedVars = ['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY'];
```

Replace with:

```typescript
const allowedVars = ['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY', 'ANTHROPIC_BASE_URL'];
```

Then rebuild:

```bash
npm run build
```

---

## Step 3: Test the Connection

### Quick validation (outside container)

```bash
curl -s https://openrouter.ai/api/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $(grep '^ANTHROPIC_API_KEY=' .env | cut -d= -f2)" \
  -d '{
    "model": "anthropic/claude-sonnet-4",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Say hello in one word."}]
  }' | jq '.content[0].text'
```

If this returns a response, OpenRouter is working.

### Full integration test

Start NanoClaw and send a test message through your configured channel (WhatsApp, Telegram, etc.):

```bash
npm run dev
```

Send a simple message to the agent and verify it responds.

---

## Compatible Models

OpenRouter provides access to many models through its unified API. When using with NanoClaw (which uses the Claude Agent SDK), the SDK sends requests in Anthropic's message format. OpenRouter translates these to the target model.

### Anthropic Models (Native Compatibility)

These work best since they match the SDK's native format:

| Model | OpenRouter Model ID | Notes |
|-------|-------------------|-------|
| Claude Opus 4 | `anthropic/claude-opus-4` | Most capable, highest cost |
| Claude Sonnet 4 | `anthropic/claude-sonnet-4` | Good balance of capability and cost |
| Claude Haiku 3.5 | `anthropic/claude-3.5-haiku` | Fastest, lowest cost |
| Claude Sonnet 3.5 | `anthropic/claude-3.5-sonnet` | Previous generation |

### Model Selection

The Claude Agent SDK selects the model internally. To control which model OpenRouter uses, you can set the `CLAUDE_MODEL` environment variable (if supported by your SDK version), or configure model routing on the OpenRouter dashboard under your API key settings.

Check OpenRouter's [model list](https://openrouter.ai/models) for current pricing and availability.

---

## Limitations vs Direct Anthropic API

| Feature | Direct Anthropic | Via OpenRouter |
|---------|-----------------|----------------|
| **Latency** | Lowest | Slightly higher (extra hop) |
| **Streaming** | Full support | Supported but may have minor differences |
| **Tool use** | Full support | Supported for Anthropic models |
| **Extended thinking** | Full support | May not be supported for all models |
| **Prompt caching** | Supported | Supported for Anthropic models |
| **Rate limits** | Per your Anthropic plan | Per your OpenRouter plan |
| **Billing** | Direct to Anthropic | Through OpenRouter (may add margin) |
| **OAuth auth** | Supported (`CLAUDE_CODE_OAUTH_TOKEN`) | Not supported - must use API key |
| **Max output tokens** | Model default | May vary by OpenRouter tier |
| **Beta features** | Immediate access | May lag behind direct API |

### Important Notes

1. **OAuth not supported**: OpenRouter uses API keys only. If you currently authenticate with `CLAUDE_CODE_OAUTH_TOKEN` (Claude subscription), you must switch to API key auth.

2. **Cost tracking**: Monitor usage at [https://openrouter.ai/activity](https://openrouter.ai/activity). OpenRouter charges per token with a small markup over direct API pricing.

3. **Reliability**: Adding OpenRouter as a middleman introduces an additional point of failure. If OpenRouter is down, your agent will not work even if Anthropic's API is healthy.

4. **Privacy**: Your prompts and responses pass through OpenRouter's infrastructure. Review their [privacy policy](https://openrouter.ai/privacy) if this is a concern.

5. **Non-Anthropic models**: While OpenRouter supports many providers, the Claude Agent SDK sends Anthropic-formatted requests. Non-Anthropic models may work for simple prompts but tool use, system prompts, and other Claude-specific features may not translate correctly.

---

## Switching Back to Direct Anthropic

To revert to direct Anthropic API access:

```bash
# Remove the base URL override
sed -i '' '/^ANTHROPIC_BASE_URL=/d' .env

# Restore your Anthropic API key (from backup or console.anthropic.com)
sed -i '' "s|^ANTHROPIC_API_KEY=.*|ANTHROPIC_API_KEY=sk-ant-api03-your-key-here|" .env
```

Or if you backed up your `.env`:

```bash
cp .env.backup .env
```

The `ANTHROPIC_BASE_URL` allowlist addition in `container-runner.ts` is harmless to keep - it will only be used if the variable is set.

Rebuild after changes:

```bash
npm run build
```

---

## Troubleshooting

### "401 Unauthorized" errors

```bash
# Verify key format (should start with sk-or-v1-)
grep "^ANTHROPIC_API_KEY=" .env

# Test key directly
curl -s https://openrouter.ai/api/v1/models \
  -H "Authorization: Bearer $(grep '^ANTHROPIC_API_KEY=' .env | cut -d= -f2)" | jq '.data | length'
```

### "Connection refused" or timeout

```bash
# Verify base URL is correct
grep "^ANTHROPIC_BASE_URL=" .env
# Must be exactly: https://openrouter.ai/api/v1

# Test connectivity
curl -s -o /dev/null -w "%{http_code}" https://openrouter.ai/api/v1/models
# Should return 200
```

### Agent container doesn't use OpenRouter

```bash
# Verify ANTHROPIC_BASE_URL is in the allowlist
grep "allowedVars" src/container-runner.ts
# Should include 'ANTHROPIC_BASE_URL'

# Verify env file passed to container includes the base URL
cat data/env/env | grep ANTHROPIC_BASE_URL
```

If `data/env/env` does not contain `ANTHROPIC_BASE_URL`, rebuild and restart:

```bash
npm run build && npm run dev
```

### Responses are slow

OpenRouter adds a routing layer on top of the provider. Expected additional latency is 100-500ms per request. If latency is significantly higher:

- Check [OpenRouter status](https://status.openrouter.ai/)
- Try a different model (some have longer queue times)
- Check your OpenRouter tier - free tier may have throttling

### "Model not found" errors

OpenRouter model IDs use the format `provider/model-name`. The Claude Agent SDK may request a model ID that doesn't match OpenRouter's naming. Check the [OpenRouter models page](https://openrouter.ai/models) for the exact ID.

---

## Security Considerations

- Your OpenRouter API key has access to your OpenRouter balance. Treat it like any other API key.
- The key is exposed inside agent containers (same limitation as direct Anthropic keys - see `docs/SECURITY.md`).
- OpenRouter supports key-level spending limits. Set one at [https://openrouter.ai/keys](https://openrouter.ai/keys) to prevent unexpected charges.
