# Codex + VibeProxy GPT-5.4 Fix Playbook

This repository documents a real incident where Codex/Godex stopped working with a local VibeProxy setup and started failing with `400 Bad Request`.

It explains:

- what broke
- why it broke
- how it was fixed
- how to verify the fix
- how to recover quickly if it happens again

## Incident Summary

The break happened after the local Codex client began using the Responses API websocket transport for the built-in `openai` provider.

The local proxy stack in this setup exposed normal HTTP Responses endpoints, but it did not support the websocket upgrade path Codex tried to use. That mismatch caused requests to fail with:

- `HTTP error: 400 Bad Request`
- `ERROR: Bad Request`

## Root Cause

Codex was configured to use the built-in OpenAI provider against a local proxy URL:

```toml
openai_base_url = "http://127.0.0.1:8328/v1"
```

After the client behavior changed, Codex attempted a websocket connection to:

```text
ws://127.0.0.1:8328/v1/responses
```

The local proxy only supported HTTP request/response handling, not websocket transport, so the handshake failed and surfaced as `400 Bad Request`.

## Fix

The durable fix was to stop using the built-in `openai` provider and instead define a custom local provider with websocket support explicitly disabled:

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

This forces Codex to stay on the normal HTTP Responses path.

## Files in This Repo

- `docs/incident-2026-04-02.md` — full incident writeup
- `docs/recovery-playbook.md` — step-by-step recovery guide

## Validation

After applying the fix, these commands succeeded:

```bash
codex exec --skip-git-repo-check -C "/Users/aejjusingh" "Reply with OK only."
codex exec --skip-git-repo-check -C "/Users/aejjusingh" -m gpt-5.4-mini "Reply with MINI only."
```

Expected output:

- `OK`
- `MINI`

## Notes

- Never commit real proxy tokens or provider API keys.
- If Godex or Codex Desktop is already open, fully quit and reopen it after editing config.
