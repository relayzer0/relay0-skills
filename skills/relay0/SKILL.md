---
name: relay0
description: Use Relay0 as a scoped multi-tenant AI gateway and agent-ops API. Trigger when the user mentions Relay0, wants to call AI models through a Relay0 /v1 endpoint, needs to configure a coding agent (Codex, Claude Code, Cursor, OpenClaw) against Relay0, set a base URL or gateway key, discover tenant-visible models, check usage/quotas, manage gateway keys or provider connections, debug a model route, or sees a connection marked unhealthy — all without exposing upstream provider secrets.
---

# Relay0

Relay0 is an OpenAI-compatible gateway plus an agent-safe management API. Use it to route model calls through the user's Relay0 gateway key, discover tenant-visible models, inspect usage/quotas, manage assigned gateway keys or provider connections, and wire coding agents (Codex, Claude Code, Cursor, OpenClaw) to the gateway.

## Install

```bash
npx skills add relayzer0/relay0-skills --skill relay0
```

## Setup

Read config from environment first. The hosted SaaS defaults are:

```bash
export RELAY0_BASE_URL="https://api.userelay0.com/v1"   # gateway, OpenAI-compatible
export RELAY0_APP_URL="https://app.userelay0.com"       # management API host
export RELAY0_API_KEY="sk-..."                          # Relay0 gateway key, not an upstream provider key
export RELAY0_AGENT_KEY="$RELAY0_API_KEY"               # agent-ops key (often the same key, different scope)
```

Users are frequently handed an env file (for example `relay0-agent-cursor.env`). Source it instead of retyping:

```bash
set -a && source /path/to/relay0-agent.env && set +a
```

`RELAY0_API_KEY` authenticates gateway calls (`$RELAY0_BASE_URL/...`). `RELAY0_AGENT_KEY` authenticates management calls (`$RELAY0_APP_URL/api/cloud/agent/...`). They are commonly the same Relay0 gateway key assigned to an agent user, but keep both names so either scope works.

If only `RELAY0_BASE_URL` exists, derive `RELAY0_APP_URL` only when obvious (for example `api.userelay0.com` -> `app.userelay0.com`). Prefer asking for the app URL before calling `/api/cloud/agent/*`.

Never ask for OpenAI, Anthropic, Google, or other upstream provider secrets. Relay0 only needs Relay0 gateway keys.

## Safety Rules

- Treat `RELAY0_API_KEY` and `RELAY0_AGENT_KEY` as secrets. Do not print them.
- Do not expose full gateway keys except when the user explicitly asks to reveal a newly-created key returned once by Relay0.
- Never request or log upstream provider API keys, OAuth access tokens, refresh tokens, or ID tokens.
- Inspect current state before mutating keys or connections.
- Ask before disabling a key or provider connection unless the user explicitly asked for an automated remediation.
- Summarize mutations with endpoint, object id, old state, new state, and reason.

## Health-Check Playbook

When asked to "test", "verify", or "check" Relay0, run this sequence and report which connection actually served the request:

1. `GET $RELAY0_APP_URL/api/cloud/agent/summary` — actor role, permissions, key active, workspace scope.
2. `GET $RELAY0_BASE_URL/models` — count of tenant-visible models.
3. `GET $RELAY0_APP_URL/api/cloud/agent/connections` — healthy vs unhealthy upstreams.
4. One live call — `POST /responses` or `/chat/completions` (see below).
5. `GET $RELAY0_APP_URL/api/cloud/agent/usage?period=7d&pageSize=5` — confirm the request landed and on which `connectionId`.

Step 5 matters: requests can fail on an unhealthy connection and then succeed on a healthy one via failover. Read `details[].connectionId` and `details[].status` to see what really happened, rather than trusting the first error.

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

## Model Selection

Always discover tenant-visible models before choosing:

```bash
curl "$RELAY0_BASE_URL/models" -H "Authorization: Bearer $RELAY0_API_KEY"
```

- Use `data[].id` **exactly** as returned. The Relay0 alias includes a prefix, for example `cx/gpt-5.5` — not bare `gpt-5.5`. A bare name will fail to route.
- Usage logs may show the upstream model name (for example `gpt-5.4-mini`) while the request used the Relay0 alias (`cx/gpt-5.4-mini`). This is expected; match on the alias when configuring tools.
- Model visibility is scoped by workspace, user, gateway key, provider assignment, and hosted OSS availability.

## Chat And Coding Requests

Use OpenAI-compatible chat unless the user requests another wire format:

```bash
curl -X POST "$RELAY0_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"cx/gpt-5.5","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

For Anthropic-shaped clients (Claude Code), use:

```bash
curl -X POST "$RELAY0_BASE_URL/messages" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"cx/gpt-5.5","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'
```

For Codex / Responses clients, use `/responses`. Codex will not work against `/chat/completions` — it requires Responses API semantics:

```bash
curl -X POST "$RELAY0_BASE_URL/responses" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"cx/gpt-5.5","input":"Reply with exactly: Relay0 OK","max_output_tokens":20}'
```

## Coding Tool Setup

To wire Codex, Claude Code, Cursor, or OpenClaw to Relay0, read `references/cli-tools.md` for copy-paste config. Key rules:

- **Codex** splits config across `~/.codex/config.toml` (routing, `wire_api = "responses"`) and `~/.codex/auth.json` (the field is named `OPENAI_API_KEY` but the value is the **Relay0 gateway key**).
- **Claude Code** uses `ANTHROPIC_BASE_URL` + `ANTHROPIC_AUTH_TOKEN`.
- **Cursor** and other OpenAI-compatible clients use the base URL + bearer key directly.
- Always pick the default model from `/models`, using the alias verbatim.

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

For full examples and response schemas, read `references/agent-api.md`.

## Connection Health And Failover

`GET /api/cloud/agent/connections` returns health signals beyond `isActive`:

- `healthStatus`: `healthy` or `unhealthy`.
- `lastHealthResult`: for example `success` or `auth_failed`.
- `lastError` / `lastErrorAt`: for example `Token invalid or revoked`.
- `priority`: Relay0 tries connections in ascending priority order, so a low-priority healthy connection covers a failed high-priority one.
- `modelLock_*`: a timestamp means that model is temporarily rate-limited / locked on that upstream.

Rules:

- An OAuth connection showing `token_invalidated` / `Token invalid or revoked` needs a **human re-auth in the Relay0 app**. An agent cannot fix this. Do **not** auto-disable it on these errors — tell the user to reconnect.
- Routing can still succeed while some connections are unhealthy; report which connection served the request rather than declaring the whole gateway broken.

## Error Handling

- `401`: wrong/missing Relay0 key, expired login/session, or bearer key not assigned to an agent/admin user.
- `403`: role or agent permission denied (actor lacks `manageKeys` / `manageConnections`). Do not retry with guessed headers.
- Empty `/models`: no visible provider/model connections for this user/key.
- Provider quota unavailable: upstream does not expose a quota API or the connection is API-key-only without quota support.
- Model route failure: inspect `/api/cloud/agent/connections` and `/api/cloud/agent/usage` before suggesting provider changes.

### Troubleshooting decision tree

```
401 on gateway        -> check RELAY0_API_KEY and (for Codex) auth.json auth_mode = "apikey"
empty /models         -> no providers assigned to this key/workspace; assign in the app
model not found        -> use the exact alias from /models (cx/...), not the bare name
intermittent failures -> GET /connections; look for unhealthy OAuth (token_invalidated)
quota / 429 errors    -> GET /quotas?connectionId=...; check session/weekly remaining
403 on agent ops      -> actor lacks manageKeys / manageConnections permission
Codex won't respond   -> ensure wire_api = "responses" and you are hitting /responses
```

## User-Facing Summary Pattern

When done, report:

```text
Relay0: checked <summary/usage/models/connections/etc>.
Visible models: <count or selected model>.
Usage: <requests/tokens/spend if checked>.
Routed via: <connection name/id if a live call was made>.
Actions: <none or concise mutation summary>.
Risk: <missing key/no providers/unhealthy connection/permission issue/etc>.
```
