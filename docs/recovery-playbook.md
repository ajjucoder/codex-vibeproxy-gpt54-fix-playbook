# Recovery Playbook

## When to Use This

Use this playbook if Codex or Godex suddenly starts failing through the local proxy with messages like:

```text
ERROR: Bad Request
HTTP error: 400 Bad Request, url: ws://127.0.0.1:8328/v1/responses
```

## Quick Diagnosis

### 1. Check that the proxy is listening

```bash
lsof -iTCP:8317 -sTCP:LISTEN
lsof -iTCP:8328 -sTCP:LISTEN
```

### 2. Check that models are still exposed

```bash
curl -sS http://127.0.0.1:8317/v1/models
curl -sS http://127.0.0.1:8328/v1/models
```

You should see model IDs including:

- `gpt-5.4`
- `gpt-5.4-mini`

### 3. Reproduce the actual client failure

```bash
codex exec --skip-git-repo-check -C "/Users/aejjusingh" "Reply with OK only."
```

If the output mentions websocket connection failure on `/v1/responses`, this is the same issue.

## Fix Steps

Edit:

```text
~/.codex/config.toml
```

Use a custom provider instead of the built-in `openai` provider:

```toml
model = "gpt-5.4"
model_provider = "localproxy"
openai_base_url = "http://127.0.0.1:8328/v1"
model_reasoning_effort = "high"
service_tier = "fast"

[model_providers.localproxy]
name = "Local Proxy"
base_url = "http://127.0.0.1:8328/v1"
wire_api = "responses"
supports_websockets = false
experimental_bearer_token = "<your-local-proxy-token>"
```

## Why This Is the Right Fix

The problem is not that GPT-5.4 is missing.
The problem is that the client is choosing websocket transport against a proxy that only supports HTTP.

Disabling websocket support at the provider layer keeps the client on the working HTTP path.

## Validation

Run:

```bash
codex exec --skip-git-repo-check -C "/Users/aejjusingh" "Reply with OK only."
codex exec --skip-git-repo-check -C "/Users/aejjusingh" -m gpt-5.4-mini "Reply with MINI only."
```

Expected:

- first command prints `OK`
- second command prints `MINI`

## App Reload Step

If Godex or Codex Desktop was already open:

1. fully quit the app
2. reopen it
3. retry the model call

## If It Breaks Again Later

Check these in order:

1. `~/.codex/config.toml` no longer contains `model_provider = "localproxy"`
2. the custom provider block was removed
3. `supports_websockets = false` was removed
4. the proxy port changed from `8328`
5. the local proxy itself stopped listening

## Safe Public-Repo Guidance

If documenting this publicly:

- never commit real API keys
- never commit real bearer tokens
- replace secrets with placeholders like `<your-local-proxy-token>`
