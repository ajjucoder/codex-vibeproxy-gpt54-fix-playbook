# Cursor Kimi/GLM Ollama Cloud Streaming Playbook

## Short Version

Kimi K2.6 and GLM 5.1 were not broken models.

They looked stuck in Cursor because the Ollama Cloud stream sends thinking chunks first, usually as `delta.reasoning`, while `delta.content` stays empty for a while. Cursor looked like it was doing nothing even though the model was thinking.

GPT worked because it streamed visible `content` in the shape Cursor already handled well.

## Working Local Chain

```text
Cursor
  -> Cloudflare tunnel / Cursor auth shim on 127.0.0.1:8331
  -> Python compatibility shim on 127.0.0.1:8329
  -> VibeProxy on 127.0.0.1:8318
  -> Ollama Cloud
```

The compatibility logic lives in:

```text
~/.cli-proxy-api/shim/factory_responses_compat.py
```

## Final Model Surface

Only advertise these two Cursor models:

- `kimi-k2.6-cloud`
- `glm-5.1-cloud`

Do not advertise the raw upstream IDs:

- `kimi-k2.6:cloud`
- `glm-5.1:cloud`

Also hide unrelated Kimi routes from Cursor if the goal is one clean Ollama Cloud Kimi choice:

- `accounts/fireworks/routers/kimi-k2p5-turbo`

If an old Cursor chat calls a raw upstream ID directly, the shim should still apply the same compatibility behavior as the safe alias.

Both should run with thinking on:

```json
{
  "reasoning": {
    "effort": "high"
  }
}
```

Keep these old aliases accepted but hidden:

- `kimi-k2.6-cloud-high`
- `glm-5.1-cloud-high`

Reason: Cursor can cache old model IDs in existing chats. Hidden aliases let old chats keep working without showing duplicate models in the picker.

## Root Cause

The bad behavior was a compatibility problem, not a model-quality problem.

The failing pattern was:

1. Cursor requested Kimi or GLM through the proxy.
2. Ollama Cloud streamed reasoning first.
3. The stream used `reasoning`, but Cursor expected visible output from `content` or a compatible reasoning field.
4. With a tiny output budget, the model could spend the whole budget thinking and never reach visible text.
5. Cursor looked stuck or abruptly stopped.

## Fix

The compatibility shim must do three things for the Ollama Cloud aliases.

### 1. Force Thinking On

Do not set reasoning to `none`.

For these aliases, pin reasoning to high:

```text
kimi-k2.6-cloud
glm-5.1-cloud
kimi-k2.6-cloud-high
glm-5.1-cloud-high
```

### 2. Give the Model Enough Output Budget

For these aliases, bump both token fields to at least:

```text
FACTORY_OLLAMA_CLOUD_MIN_OUTPUT_TOKENS=1024
```

If the env var is not set, the default should be `1024`.

Apply the minimum to both:

- `max_tokens`
- `max_completion_tokens`

This prevents Cursor's tiny test requests from burning all tokens on thinking.

### 3. Normalize Reasoning Fields

When a streamed chunk has:

```json
{
  "delta": {
    "reasoning": "..."
  }
}
```

also add:

```json
{
  "delta": {
    "reasoning_content": "..."
  }
}
```

Do the same for non-stream messages:

```json
{
  "message": {
    "reasoning": "...",
    "reasoning_content": "..."
  }
}
```

This keeps Cursor and other OpenAI-compatible clients from treating thinking chunks as invisible/no-output chunks.

## Restart After Changing the Shim

Restart the local compatibility service:

```bash
launchctl kickstart -k gui/$(id -u)/com.aejjusingh.vibe-cursor-alias
```

Then fully restart Cursor so the model picker refreshes.

## Verification

### Check the advertised model list

```bash
curl -sS http://127.0.0.1:8329/v1/models \
  | jq -r '.data[].id' \
  | rg 'kimi-k2\.6-cloud|glm-5\.1-cloud'
```

Expected visible models:

```text
kimi-k2.6-cloud
glm-5.1-cloud
```

Do not leave another Kimi route visible unless you intentionally want Cursor to offer it. During this setup, `accounts/fireworks/routers/kimi-k2p5-turbo` was hidden so Cursor only shows the intended Ollama Cloud Kimi K2.6 alias.

These old aliases should not appear in `/v1/models`:

```text
kimi-k2.6-cloud-high
glm-5.1-cloud-high
```

They should still work if called directly, because old Cursor chats can cache them.

### Check Kimi direct through the compatibility shim

```bash
curl -sS http://127.0.0.1:8329/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{
    "model": "kimi-k2.6-cloud",
    "messages": [{"role": "user", "content": "Think briefly, then reply exactly OK_DONE"}],
    "max_tokens": 16,
    "stream": false
  }' \
  | jq '.choices[0].message | {content, has_reasoning: has("reasoning"), has_reasoning_content: has("reasoning_content")}'
```

Expected:

```json
{
  "content": "OK_DONE",
  "has_reasoning": true,
  "has_reasoning_content": true
}
```

### Check GLM direct through the compatibility shim

```bash
curl -sS http://127.0.0.1:8329/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{
    "model": "glm-5.1-cloud",
    "messages": [{"role": "user", "content": "Think briefly, then reply exactly OK_DONE"}],
    "max_tokens": 16,
    "stream": false
  }' \
  | jq '.choices[0].message | {content, has_reasoning: has("reasoning"), has_reasoning_content: has("reasoning_content")}'
```

Expected:

```json
{
  "content": "OK_DONE",
  "has_reasoning": true,
  "has_reasoning_content": true
}
```

### Check through the Cursor auth shim

Use the local origin key header, but never commit the real key:

```bash
curl -sS http://127.0.0.1:8331/v1/chat/completions \
  -H 'content-type: application/json' \
  -H 'x-vibeproxy-origin-key: <local-origin-key>' \
  -d '{
    "model": "kimi-k2.6-cloud",
    "messages": [{"role": "user", "content": "Think briefly, then reply exactly OK_DONE"}],
    "max_tokens": 16,
    "stream": false
  }' \
  | jq '.choices[0].message | {content, has_reasoning: has("reasoning"), has_reasoning_content: has("reasoning_content")}'
```

Repeat with:

```json
{
  "model": "glm-5.1-cloud"
}
```

## If This Breaks Again

Check these before blaming the model:

1. `/v1/models` advertises only one Kimi model and one GLM model.
2. Raw IDs like `kimi-k2.6:cloud` and `glm-5.1:cloud` are hidden from `/v1/models`.
3. Unrelated Kimi routes like `accounts/fireworks/routers/kimi-k2p5-turbo` are hidden unless intentionally enabled.
4. `factory_responses_compat.py` still maps Kimi and GLM aliases to the Ollama Cloud model IDs.
5. Direct raw-ID calls still receive the same high-reasoning compatibility handling.
6. `reasoning.effort` is still `high`, not `none`.
7. The shim still bumps output tokens to at least `1024`.
8. Streaming chunks with `reasoning` are still copied to `reasoning_content`.
9. Cursor was fully restarted after model-list changes.
10. Direct calls to `127.0.0.1:8329` fail even with a 1024 token budget.

Only after those checks fail should this be treated as a real upstream model/provider problem.

## Do Not Repeat This Mistake

- Do not turn thinking off to make the stream look faster.
- Do not show duplicate `cloud` and `cloud-high` models in Cursor.
- Do not assume blank early stream chunks mean the model is broken.
- Do not commit real keys, bearer tokens, or private tunnel URLs.
