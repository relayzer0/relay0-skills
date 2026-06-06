---
name: relay0
description: Use Relay0 as a scoped multi-tenant AI gateway and agent-ops API. Trigger when the user mentions Relay0, wants to call AI models through a Relay0 /v1 endpoint, needs model discovery, coding-agent configuration, usage/quota checks, gateway key management, provider connection status, or asks an LLM agent to manage Relay0 accounts without exposing upstream provider secrets.
---

# Relay0

Relay0 is an OpenAI-compatible gateway plus an agent-safe management API. Use it to route model calls through the user's Relay0 gateway key, discover tenant-visible models, inspect usage/quotas, and manage assigned gateway keys or provider connections.

## Setup

Read config from environment first:

```bash
export RELAY0_BASE_URL="https://api.example.com/v1"
export RELAY0_API_KEY="sk-..."          # Relay0 gateway key, not an upstream provider key
export RELAY0_APP_URL="https://app.example.com"
export RELAY0_AGENT_KEY="$RELAY0_API_KEY"
```

If only `RELAY0_BASE_URL` exists, derive `RELAY0_APP_URL` only when obvious. Prefer asking for the app URL before calling `/api/cloud/agent/*`.

Never ask for OpenAI, Anthropic, Google, or other upstream provider secrets. Relay0 only needs Relay0 gateway keys.

## Safety Rules

- Treat `RELAY0_API_KEY` and `RELAY0_AGENT_KEY` as secrets. Do not print them.
- Do not expose full gateway keys except when the user explicitly asks to reveal a newly-created key returned once by Relay0.
- Never request or log upstream provider API keys, OAuth access tokens, refresh tokens, or ID tokens.
- Inspect current state before mutating keys or connections.
- Ask before disabling a key or provider connection unless the user explicitly asked for an automated remediation.
- Summarize mutations with endpoint, object id, old state, new state, and reason.

## Verify Access

```bash
curl "$RELAY0_BASE_URL/models" \
  -H "Authorization: Bearer $RELAY0_API_KEY"
```

Expected shape:

```json
{ "object": "list", "data": [{ "id": "cx/gpt-5.5", "object": "model" }] }
```

If `data` is empty, tell the user to connect or assign providers in Relay0. Do not invent global model names.

## Chat And Coding Requests

Use OpenAI-compatible chat unless the user requests another wire format:

```bash
curl -X POST "$RELAY0_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"MODEL_ID","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

For Anthropic-shaped clients, use:

```bash
curl -X POST "$RELAY0_BASE_URL/messages" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"MODEL_ID","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'
```

For Codex/Responses clients, use `/responses` when the target tool requires Responses API semantics.

## Model Selection

Always discover tenant-visible models before choosing:

```bash
curl "$RELAY0_BASE_URL/models" -H "Authorization: Bearer $RELAY0_API_KEY"
```

Use `data[].id` exactly as returned. Model visibility is scoped by workspace, user, gateway key, provider assignment, and hosted OSS availability.

## Agent Operations

Use `RELAY0_APP_URL` plus `RELAY0_AGENT_KEY`:

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/summary" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

Core endpoints:

| Task | Method + path |
|---|---|
| Workspace/actor summary | `GET /api/cloud/agent/summary` |
| Usage + recent requests | `GET /api/cloud/agent/usage?period=7d&pageSize=20` |
| Visible gateway keys | `GET /api/cloud/agent/keys` |
| Create gateway key | `POST /api/cloud/agent/keys` |
| Rename/enable/disable key | `PATCH /api/cloud/agent/keys` |
| Visible provider connections | `GET /api/cloud/agent/connections` |
| Rename/enable/disable connection | `PATCH /api/cloud/agent/connections` |
| Provider quota/status | `GET /api/cloud/agent/quotas` |

For full examples, read `references/agent-api.md`.

## Error Handling

- `401`: wrong/missing Relay0 key, expired login/session, or bearer key not assigned to an agent/admin user.
- `403`: role or agent permission denied. Do not retry with guessed headers.
- Empty `/models`: no visible provider/model connections for this user/key.
- Provider quota unavailable: upstream does not expose quota API or connection is API-key-only without quota support.
- Model route failure: inspect `/api/cloud/agent/connections` and `/api/cloud/agent/usage` before suggesting provider changes.

## User-Facing Summary Pattern

When done, report:

```text
Relay0: checked <summary/usage/models/etc>.
Visible models: <count or selected model>.
Usage: <requests/tokens/spend if checked>.
Actions: <none or concise mutation summary>.
Risk: <missing key/no providers/permission issue/etc>.
```
