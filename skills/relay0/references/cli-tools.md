# Relay0 Coding Tool Setup

Every tool needs:

1. **Base URL** — Workspace `https://api.userelay0.com/v1` or Grid `https://grid.userelay0.com/v1` (must match key type).
2. **Relay0 gateway key** — never an upstream provider key.
3. **Model id(s)** — exact ids from discovery (never invent).

### Prefer the `relay0` CLI (any coding tool)

```bash
export PATH="$HOME/.local/bin:$HOME/.relay0/bin:$PATH"

relay0 models                                 # source of truth for ids
relay0 set <tool> model <exact-id>            # change default model for a tool
relay0 set claude model <id> --slot haiku     # Claude slot: all|opus|sonnet|haiku
relay0 get <tool> model
relay0 doctor
```

Examples:

```bash
relay0 set codex model cx/gpt-5.5
relay0 set claude model xai/grok-4.5
relay0 set openclaw model cx/gpt-5.4-mini
relay0 set opencode model xai/grok-4.5
```

Agents should run `relay0 set` / `relay0 models` instead of scraping tool configs or hand-rolling curl.

Fallback without CLI:

```bash
export RELAY0_BASE_URL="https://api.userelay0.com/v1"   # or grid.userelay0.com/v1
export RELAY0_API_KEY="sk-..."

curl -sS "$RELAY0_BASE_URL/models" \
  -H "Authorization: Bearer $RELAY0_API_KEY" | jq -r '.data[].id'
```

**Workspace ids** are often prefixed (`xai/grok-4.5`, `cx/gpt-5.5`). **Grid retail ids** are often bare (`grok-4.5`). Always copy what discovery returns for *this* host + key.

The hosted installer (`/setup?token=&capacity=byok|grid&tools=all`) writes these files for you. This doc is for agents and manual edits — especially **changing models per tool**.

---

## How to change models (agent recipe)

```bash
relay0 models
relay0 set <tool> model <exact-id-from-models>
# restart the tool if it is already running
```

That is enough for codex, claude, openclaw, opencode, cline, qwen, hermes, droid, jcode, shell.
Claude slots: `--slot opus|sonnet|haiku|all` (default all).

Cursor / Continue: CLI prints UI instructions (no reliable file API).

You may assign **different** ids to different tools or slots.

---

## Codex

Files:

- `~/.codex/config.toml` — routing + **preferred** auth via `experimental_bearer_token`
- `~/.codex/auth.json` — keep in sync (`auth_mode = "apikey"`, `OPENAI_API_KEY` = Relay0 key)

### Installer-correct config.toml

```toml
model = "xai/grok-4.5"
model_provider = "relay0"

[model_providers.relay0]
name = "Relay0"
base_url = "https://api.userelay0.com/v1"
experimental_bearer_token = "sk-...relay0-gateway-key"
wire_api = "responses"
requires_openai_auth = false

[agents.subagent]
model = "xai/grok-4.5"
```

### auth.json

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-...relay0-gateway-key"
}
```

Clear stale ChatGPT login fields (`tokens`, `access_token`, `refresh_token`, `id_token`) if present so subscription auth does not override Relay0.

### Change the model

1. List ids from `/models`.
2. Set root `model = "<id>"`.
3. Optionally set `[agents.subagent] model = "<other-id>"` for a cheaper subagent.
4. Do **not** invent bare names if the catalog shows prefixes.
5. Restart Codex.

### Why not `env_key` alone?

Codex `env_key = "OPENAI_API_KEY"` requires that variable in the **shell environment**. Many users only have the key in `auth.json` or the TOML bearer field → error **Missing OPENAI_API_KEY**. Prefer `experimental_bearer_token` (installer default) and still sync `auth.json`.

### Rules

- `wire_api = "responses"` is required.
- Codex must call `/responses` semantics, not only chat completions.
- Leave other `[model_providers.*]` blocks if the user switches providers; only `model_provider` selects the active one.
- Grid: set `base_url` to `https://grid.userelay0.com/v1` and use Grid retail ids.

### Verify

```bash
curl -X POST "$RELAY0_BASE_URL/responses" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"<id-from-/models>","input":"Reply with exactly: Codex via Relay0 OK","max_output_tokens":20}'
```

---

## Claude Code

File: `~/.claude/settings.json` (env map). Claude Code has **three model slots** that can each be a different Relay0 id.

```json
{
  "hasCompletedOnboarding": true,
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.userelay0.com/v1",
    "ANTHROPIC_AUTH_TOKEN": "sk-...relay0-gateway-key",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "cc/claude-opus-4-6",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "xai/grok-4.5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "cx/gpt-5.4-mini"
  }
}
```

### Map slots from `/models`

| Slot | Env key | Prefer ids matching |
|---|---|---|
| Opus (strong) | `ANTHROPIC_DEFAULT_OPUS_MODEL` | `opus`, `thinking`, top-tier `gpt-5`, `grok-4` |
| Sonnet (default) | `ANTHROPIC_DEFAULT_SONNET_MODEL` | `sonnet`, mid-tier coding, `gpt-5.4` / `gpt-5.5` |
| Haiku (fast) | `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `haiku`, `mini`, `flash`, `lite` |

If only one model is visible, set all three slots to that same id (installer behavior). Later, re-split when more ids appear.

Optional: `ANTHROPIC_MODEL` for a single default override if the user’s Claude build respects it.

Requests use Anthropic `/messages`; Relay0 accepts that shape with the same model ids.

### Change one slot

Edit only that env key in `settings.json`, save, restart Claude Code. Confirm with:

```bash
curl -X POST "$RELAY0_BASE_URL/messages" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"<slot-id>","max_tokens":64,"messages":[{"role":"user","content":"ping"}]}'
```

---

## Cursor

Manual UI (Settings → Models):

1. Enable OpenAI-compatible / override base URL → `$RELAY0_BASE_URL` (include `/v1`).
2. API key → Relay0 gateway key.
3. Add a **custom model** whose name is the exact `/models` id.
4. To use multiple models: add multiple custom models (one id each); pick in the chat model menu.

There is no separate “Relay0 model list” inside Cursor — the custom model string is what gets sent as `model`.

---

## OpenClaw

Typical file: `~/.openclaw/openclaw.json`

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "relay0/xai/grok-4.5" }
    }
  },
  "models": {
    "providers": {
      "relay0": {
        "baseUrl": "https://api.userelay0.com/v1",
        "apiKey": "sk-...relay0-gateway-key",
        "api": "openai-completions",
        "models": [
          { "id": "xai/grok-4.5", "name": "xai/grok-4.5" },
          { "id": "cx/gpt-5.4-mini", "name": "cx/gpt-5.4-mini" }
        ]
      }
    }
  }
}
```

### Model reference rules

- Provider registry lists **raw** ids in `models[]` (same as `/models`).
- Default / agent refs use **`relay0/<raw-id>`** (provider slug + slash + raw id).
- Populate `models[]` from the full catalog (or the first ~20 coding-relevant ids).
- Change primary: set `agents.defaults.model.primary` to `relay0/<new-id>` and ensure that id exists in `models[]`.

---

## OpenCode

File: `~/.config/opencode/opencode.json` (path may vary)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "relay0": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Relay0",
      "options": {
        "baseURL": "https://api.userelay0.com/v1",
        "apiKey": "sk-...relay0-gateway-key"
      },
      "models": {
        "xai/grok-4.5": { "name": "xai/grok-4.5" },
        "cx/gpt-5.4-mini": { "name": "cx/gpt-5.4-mini" }
      }
    }
  },
  "model": "relay0/xai/grok-4.5",
  "agent": {
    "explorer": {
      "description": "Fast codebase exploration",
      "mode": "subagent",
      "model": "relay0/cx/gpt-5.4-mini"
    }
  }
}
```

Use different `model` vs `agent.*.model` for strong primary + cheap explorer.

---

## Cline

- Global state (often `~/.cline/data/globalState.json`): OpenAI provider, **base URL without `/v1`** in some builds, `openAiModelId` / plan-mode model id.
- Secrets: `openAiApiKey` = Relay0 key.

Set act vs plan models to different Relay0 ids if desired.

---

## Roo / Kilo / generic OpenAI-compatible UIs

```text
Provider: OpenAI Compatible
Base URL: https://api.userelay0.com/v1   # some want without /v1 — try both if 404
API Key:  sk-...relay0-gateway-key
Model:    <exact id from /models>
```

If the UI has separate “plan” and “act” models, assign two ids from the catalog.

---

## Continue (VS Code / JetBrains)

Add one object per model to the Continue `models` array:

```json
{
  "apiBase": "https://api.userelay0.com/v1",
  "title": "Relay0 Grok",
  "model": "xai/grok-4.5",
  "provider": "openai",
  "apiKey": "sk-...relay0-gateway-key"
}
```

Duplicate and change `model` / `title` for each catalog entry you want in the picker.

---

## Amp CLI / shell-first tools

```bash
export OPENAI_API_KEY="sk-...relay0-gateway-key"
export OPENAI_BASE_URL="https://api.userelay0.com/v1"
amp --model "xai/grok-4.5"
```

Pass the exact id on the CLI or in the tool’s model flag.

---

## Qwen Code / DeepSeek TUI / Factory Droid / Hermes / jcode

Pattern is always the same:

| Field | Value |
|---|---|
| Provider mode | OpenAI-compatible |
| Base URL | `$RELAY0_BASE_URL` |
| API key | Relay0 gateway key |
| Model | Exact `/models` id |

Example DeepSeek TUI `~/.deepseek/config.toml`:

```toml
provider = "openai"
base_url = "https://api.userelay0.com/v1"
api_key = "sk-...relay0-gateway-key"
model = "xai/grok-4.5"
```

Example jcode multi-model profiles:

```toml
[providers.relay0]
type = "openai"
base_url = "https://api.userelay0.com/v1"
api_key = "sk-...relay0-gateway-key"

[agents.default]
model = "xai/grok-4.5"
provider = "relay0"

[agents.fast]
model = "cx/gpt-5.4-mini"
provider = "relay0"
```

---

## Hosted installer and cleanup

```text
# Configure all supported tools (prompts capacity if omitted)
https://app.userelay0.com/setup?token=sk-...&capacity=byok&tools=all
https://app.userelay0.com/setup?token=sk-...&capacity=grid&tools=all

# Cleanup Relay0 configuration
https://app.userelay0.com/setup?tools=cleanup
```

Installer behavior agents should know:

- Resolves default model from live `/models` (Grid fallback may be `grok-4.5` if catalog empty).
- Codex: writes `experimental_bearer_token` + syncs `auth.json`.
- Claude: may set all three slots to the same default; re-split manually for multi-model.
- Offers to install the **Relay0 agent skill** (`npx skills add relayzer0/relay0-skills --skill relay0 -g -y`). Interactive default is yes; non-interactive needs `skill=yes` or `RELAY0_INSTALL_SKILL=1`.
- Wrong capacity/host → 403; re-run with correct `capacity=` or new key from the matching Keys page.

---

## Verify any tool

```bash
MODEL_ID="<paste-from-/models>"

curl -sS -X POST "$RELAY0_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $RELAY0_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"$MODEL_ID\",\"messages\":[{\"role\":\"user\",\"content\":\"ping\"}],\"stream\":false}"
```

Codex: also verify `/responses`. Claude: also verify `/messages`.

If it fails, see the troubleshooting tree in `SKILL.md` (host mismatch, empty catalog, bad id, Codex bearer).
