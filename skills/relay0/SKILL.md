---
name: relay0
description: >
  Use Relay0 as a scoped multi-tenant AI gateway and agent-ops API. Trigger when the user
  mentions Relay0; wants different models on Codex, Claude Code, Cursor, OpenClaw, or other
  coding tools; needs to discover models via GET /v1/models; configure base URL or gateway key;
  run the Relay0 setup installer or cleanup; check usage/quotas; manage keys or provider
  connections; debug a model route or empty catalog; or pick Workspace (api.*) vs Grid (grid.*)
  hosts — all without exposing upstream provider secrets.
---

# Relay0

Relay0 is an OpenAI-compatible gateway plus an agent-safe management API. Use it to:

1. Discover which models a gateway key can actually call (`GET /v1/models`).
2. Wire coding tools (Codex, Claude Code, Cursor, OpenClaw, …) to Relay0 **with the correct model ids per tool**.
3. Call chat / responses / messages endpoints.
4. Inspect usage, quotas, keys, and provider health via the Agent API.

**Critical rule:** never invent model names. Always use exact `id` values from `/v1/models` for that key + host.

## Install this skill

```bash
npx skills add relayzer0/relay0-skills --skill relay0
```

## Setup (env)

Read config from environment first. Hosted SaaS defaults:

```bash
# Workspace (BYOK) — providers the user connects under /providers
export RELAY0_BASE_URL="https://api.userelay0.com/v1"

# Grid (prepaid pool) — admin-built capacity; use grid host + a Grid key
# export RELAY0_BASE_URL="https://grid.userelay0.com/v1"

export RELAY0_APP_URL="https://app.userelay0.com"
export RELAY0_API_KEY="sk-..."            # Relay0 gateway key (not OpenAI/Anthropic/xAI)
export RELAY0_AGENT_KEY="$RELAY0_API_KEY" # same key often works for agent-ops
```

Source an env file when the user has one:

```bash
set -a && source /path/to/relay0-agent.env && set +a
```

| Variable | Used for |
|---|---|
| `RELAY0_BASE_URL` | Gateway OpenAI-compatible API (`/models`, `/chat/completions`, `/responses`, `/messages`) |
| `RELAY0_API_KEY` | `Authorization: Bearer` on the gateway |
| `RELAY0_APP_URL` | Management host for `/api/cloud/agent/*` |
| `RELAY0_AGENT_KEY` | Bearer for agent-ops (often same as gateway key) |

Never ask for upstream provider secrets (OpenAI, Anthropic, Google, xAI, OAuth tokens). Only Relay0 gateway keys.

### Two products, not brand lanes

| Lane | Host | Who connects capacity | Typical model ids |
|---|---|---|---|
| **Workspace (BYOK)** | `https://api.userelay0.com/v1` | User/admin under `/providers` | Prefixed aliases: `xai/grok-4.5`, `cx/gpt-5.5`, `cc/claude-sonnet-…` |
| **Grid (prepaid)** | `https://grid.userelay0.com/v1` | Platform admin Grid pool only | Retail bare ids: `grok-4.5` (not `xai/grok-4.5`) |

- Brands ≠ capacity. xAI / ChatGPT / Claude can be **workspace BYOK** if the user connected them, or **Grid** if an admin put them in the pool.
- Wrong host for the key type → **403**. Grid keys must hit `grid.*`; workspace keys hit `api.*`.
- Console: Keys + Providers at `https://app.userelay0.com`.

If only `RELAY0_BASE_URL` is set, derive `RELAY0_APP_URL` only when obvious (`api.` / `grid.` → `app.`). Prefer asking for the app URL before calling agent-ops.

---

## Agent playbook: figure out models for any tool

When the user asks “what models can I use?”, “set Claude to Grok”, “switch Codex model”, or you need to configure a tool:

### Step 1 — Discover (mandatory)

```bash
curl -sS "$RELAY0_BASE_URL/models" \
  -H "Authorization: Bearer $RELAY0_API_KEY" | jq -r '.data[].id'
```

- Use **exact** `data[].id` strings. Do not strip prefixes. Do not invent names from memory.
- Empty list → no visible capacity for this key/host. Tell the user:
  - Workspace: connect providers at `/providers` and ensure the key can see them.
  - Grid: ask an admin to publish pool models; confirm Grid key + `grid.*` host.
- Usage logs may show a bare upstream name while the request used a Relay0 alias — always configure tools with the **alias/`id` from `/models`**.

### Step 2 — Pick ids by intent

Heuristics (apply only to ids actually returned):

| Intent | Prefer ids matching |
|---|---|
| Strong coding / default | `grok-4.5`, `gpt-5`, `sonnet`, `opus`, `codex` |
| Fast / cheap | `mini`, `haiku`, `flash`, `lite` |
| Avoid for chat agents | `embed`, `tts`, `whisper`, `image`, `dall`, `audio`, smoke/test ids |

If the user names a brand (“use Grok”, “use Claude”), filter `/models` for that substring; if multiple, prefer the highest tier visible; if none, report available ids and stop.

### Step 3 — Map id → tool-specific knobs

Each tool stores the model in a different place. Full copy-paste configs: `references/cli-tools.md`.

| Tool | Where the model is set | Notes |
|---|---|---|
| **Codex** | `~/.codex/config.toml` → `model = "<id>"` (and optional `[agents.subagent] model`) | Provider: `experimental_bearer_token` + `wire_api = "responses"`. Also keep `auth.json` `OPENAI_API_KEY` = Relay0 key. |
| **Claude Code** | `~/.claude/settings.json` → `env.ANTHROPIC_DEFAULT_{OPUS,SONNET,HAIKU}_MODEL` | Three slots can point at **different** Relay0 ids. Optional `ANTHROPIC_MODEL`. |
| **Cursor** | Settings → Models → custom model name = exact id | Base URL + OpenAI-compatible key; custom model string must match `/models`. |
| **OpenClaw** | `openclaw.json` → `agents.defaults.model.primary` = `relay0/<id>` and list entries under `models.providers.relay0.models[]` | Provider key is `relay0`; primary uses `relay0/` prefix + raw id. |
| **OpenCode** | `opencode.json` → `model`: `relay0/<id>`; more under `provider.relay0.models` | Can set explorer/subagent to a cheaper id. |
| **Cline / Kilo / Roo** | OpenAI-compatible model id field | Some UIs want base URL **without** trailing `/v1`. |
| **Continue** | `models[]` entry: `model` + `apiBase` + `apiKey` | Duplicate objects for multiple Relay0 models. |
| **Generic OpenAI SDK** | request body `model` | Same id as `/models`. |
| **Anthropic-shaped client** | request body `model` + `/messages` | Same id; Relay0 accepts Anthropic wire format. |

### Step 4 — Apply and verify

1. Edit only the target tool’s config (or re-run installer — see below).
2. Restart the tool if it caches config.
3. Smoke-test the **wire format that tool uses**:

```bash
# OpenAI chat (Cursor, many CLIs)
curl -sS -X POST "$RELAY0_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL_ID\",\"messages\":[{\"role\":\"user\",\"content\":\"ping\"}],\"stream\":false}"

# Codex / Responses
curl -sS -X POST "$RELAY0_BASE_URL/responses" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL_ID\",\"input\":\"Reply with exactly: OK\",\"max_output_tokens\":20}"

# Claude Code /messages
curl -sS -X POST "$RELAY0_BASE_URL/messages" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL_ID\",\"max_tokens\":64,\"messages\":[{\"role\":\"user\",\"content\":\"ping\"}]}"
```

4. Optional: confirm routing via Agent API usage (`details[].connectionId`, `status`).

### Different models per tool (and per slot)

This is supported and expected:

- Codex default `model = "xai/grok-4.5"` while Claude Sonnet slot is `cc/claude-sonnet-…` and Haiku is a mini id.
- OpenClaw primary expensive, explorer/subagent cheap.
- Cursor custom models: add **multiple** custom model names, each an exact `/models` id; user picks in the UI.

Do **not** assume one global “Relay0 model” for all tools. Discover once, then set each tool independently.

---

## One-shot installer (recommended for humans)

Hosted setup script configures tools from a live catalog:

```text
https://app.userelay0.com/setup?token=sk-...&capacity=byok|grid&tools=all
```

- Prompts or uses `capacity=byok|grid` → correct host.
- Fetches `/models` and picks a sensible default.
- Writes Codex with `experimental_bearer_token` (not bare `env_key` alone — `env_key` fails if `OPENAI_API_KEY` is unset in the shell).
- Maps Claude’s three slots (installer may set all three to the same default; you can re-split later).
- Cleanup: `tools=cleanup` or installer cleanup mode removes Relay0 blocks without nuking the whole user config when possible.

After install, agents should still re-list `/models` before recommending a different default.

---

## Safety rules

- Treat `RELAY0_API_KEY` / `RELAY0_AGENT_KEY` as secrets; do not print them.
- Reveal a full key only when the user asks and Relay0 just returned a one-time create response.
- Never request or log upstream provider API keys or OAuth tokens.
- Inspect before mutating keys/connections; ask before disable unless user requested automated remediation.
- Summarize mutations: endpoint, object id, old → new, reason.

---

## Health-check playbook

When asked to test/verify/check Relay0:

1. `GET $RELAY0_APP_URL/api/cloud/agent/summary` — role, permissions, key active.
2. `GET $RELAY0_BASE_URL/models` — visible model count + sample ids.
3. `GET $RELAY0_APP_URL/api/cloud/agent/connections` — healthy vs unhealthy.
4. Live call on `/responses` or `/chat/completions` with a real `/models` id.
5. `GET .../usage?period=7d&pageSize=5` — which `connectionId` served the request (failover may succeed after an error row).

---

## Chat and coding requests

```bash
# Chat completions
curl -X POST "$RELAY0_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"<id-from-/models>","messages":[{"role":"user","content":"Hello"}],"stream":false}'

# Anthropic Messages (Claude Code)
curl -X POST "$RELAY0_BASE_URL/messages" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"<id-from-/models>","max_tokens":1024,"messages":[{"role":"user","content":"Hello"}]}'

# Responses (Codex) — required wire; chat completions alone is not enough for Codex
curl -X POST "$RELAY0_BASE_URL/responses" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"<id-from-/models>","input":"Reply with exactly: Relay0 OK","max_output_tokens":20}'
```

---

## Coding tool setup

Read **`references/cli-tools.md`** for full configs and “change model later” recipes.

Quick Codex shape (current installer truth):

```toml
model = "<id-from-/models>"
model_provider = "relay0"

[model_providers.relay0]
name = "Relay0"
base_url = "https://api.userelay0.com/v1"   # or grid.userelay0.com/v1
experimental_bearer_token = "sk-..."      # Relay0 gateway key
wire_api = "responses"
requires_openai_auth = false
```

Also set `~/.codex/auth.json`:

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-...relay0-gateway-key"
}
```

---

## Agent operations

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/summary" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

| Task | Method + path |
|---|---|
| Summary | `GET /api/cloud/agent/summary` |
| Usage | `GET /api/cloud/agent/usage?period=7d&pageSize=20` |
| Keys | `GET` / `POST` / `PATCH /api/cloud/agent/keys` |
| Connections | `GET` / `PATCH /api/cloud/agent/connections` |
| Quotas | `GET /api/cloud/agent/quotas` |

Schemas and examples: `references/agent-api.md`.

### Connection health

From `GET .../connections`:

- `healthStatus`: `healthy` | `unhealthy`
- `lastHealthResult`, `lastError`, `priority`, model lock timestamps
- OAuth `token_invalidated` → **user must re-auth in the app**; do not auto-disable; routing may still succeed via another connection

---

## Error handling

| Signal | Action |
|---|---|
| `401` gateway | Check `RELAY0_API_KEY`; Codex auth_mode / bearer token |
| `403` gateway | Host/key capacity mismatch (api vs grid) |
| `403` agent-ops | Missing `manageKeys` / `manageConnections` |
| Empty `/models` | No visible providers for this key/lane |
| Model not found | Wrong id — re-fetch `/models`; do not strip prefixes |
| Codex silent/fail | `wire_api = "responses"`, hit `/responses`, bearer present |
| Intermittent fail | Check connections + usage failover rows |

```
401 on gateway        -> RELAY0_API_KEY; Codex experimental_bearer_token + auth.json
403 on gateway        -> api.* vs grid.* mismatch for key type
empty /models         -> connect providers (BYOK) or publish Grid pool; wrong host
model not found       -> exact id from /models for this host
Codex Missing OPENAI_API_KEY -> use experimental_bearer_token (not env_key alone)
OAuth token_invalidated -> human reconnect in app; do not auto-disable
```

---

## User-facing summary pattern

```text
Relay0: checked <summary/usage/models/connections/tools>.
Host/lane: <api workspace | grid> <base url>.
Visible models: <count>; using <id> for <tool>.
Per-tool models: Codex=<id>, Claude opus/sonnet/haiku=<ids>, …
Usage: <requests/tokens if checked>; routed via <connection if known>.
Actions: <none or mutation summary>.
Risk: <missing key / empty catalog / unhealthy OAuth / host mismatch / …>.
```
