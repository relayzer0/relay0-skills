# Relay0 Coding Tool Setup

Configure coding agents to route through a Relay0 gateway. Every tool needs two things: the gateway base URL (`$RELAY0_BASE_URL`, e.g. `https://api.userelay0.com/v1`) and a Relay0 gateway key (`$RELAY0_API_KEY`).

Always pick the default model from `/models` and use the alias verbatim (for example `cx/gpt-5.5`, not `gpt-5.5`):

```bash
curl "$RELAY0_BASE_URL/models" -H "Authorization: Bearer $RELAY0_API_KEY"
```

Never put an upstream provider key anywhere in these configs. Relay0 only uses Relay0 gateway keys.

## Codex

Codex splits its config across two files and speaks the Responses API.

`~/.codex/config.toml` — routing (no key here):

```toml
model = "cx/gpt-5.5"
model_provider = "relay0"

[model_providers.relay0]
name = "Relay0"
base_url = "https://api.userelay0.com/v1"
wire_api = "responses"

[agents.subagent]
model = "cx/gpt-5.5"
```

`~/.codex/auth.json` — the key. The field is named `OPENAI_API_KEY`, but the value is the **Relay0 gateway key**:

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-...relay0-gateway-key"
}
```

Notes:

- `wire_api = "responses"` is required — Codex will not work against plain chat completions.
- Restart Codex after editing these files.
- Leave any other `[model_providers.*]` blocks in place if the user wants to switch back; only `model_provider` selects the active one.
- Verify with a Responses call before declaring success:

```bash
curl -X POST "$RELAY0_BASE_URL/responses" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"cx/gpt-5.5","input":"Reply with exactly: Codex via Relay0 OK","max_output_tokens":20}'
```

## Claude Code

Claude Code uses Anthropic-style env vars / `settings.json`. Point the base URL at the Relay0 gateway and use the gateway key as the auth token:

```json
{
  "hasCompletedOnboarding": true,
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.userelay0.com/v1",
    "ANTHROPIC_AUTH_TOKEN": "sk-...relay0-gateway-key",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "cx/gpt-5.5",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "cx/gpt-5.5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "cx/gpt-5.4-mini"
  }
}
```

Map the Opus/Sonnet/Haiku slots to whatever aliases `/models` exposes. Requests use the Anthropic `/messages` shape, which Relay0 accepts.

## Cursor

Cursor is OpenAI-compatible. In Settings -> Models:

1. Enable "Override OpenAI Base URL" and set it to `https://api.userelay0.com/v1`.
2. Set the OpenAI API key to your Relay0 gateway key.
3. Add a custom model whose name matches a Relay0 alias from `/models` (for example `cx/gpt-5.5`), then verify the connection.

The same pattern works for any OpenAI-compatible client: base URL + bearer key + an exact alias.

## OpenClaw

OpenClaw takes a provider block in its JSON config:

```json
{
  "agents": {
    "defaults": { "model": { "primary": "relay0/cx/gpt-5.5" } }
  },
  "models": {
    "providers": {
      "relay0": {
        "baseUrl": "https://api.userelay0.com/v1",
        "apiKey": "sk-...relay0-gateway-key",
        "api": "openai-completions",
        "models": [
          { "id": "cx/gpt-5.5", "name": "cx/gpt-5.5" }
        ]
      }
    }
  }
}
```

Populate `models[]` from `/models`. Reference models elsewhere as `relay0/<alias>`.

## Verify any tool

After configuring, confirm the gateway answers before handing back to the user:

```bash
curl -X POST "$RELAY0_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"cx/gpt-5.5","messages":[{"role":"user","content":"ping"}],"stream":false}'
```

If it fails, see the troubleshooting decision tree in `SKILL.md`.
