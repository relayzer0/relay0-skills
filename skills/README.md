# Relay0 Skills

Agent-ready skills for Relay0. The entry skill teaches an agent how to call the Relay0 `/v1` gateway, discover scoped models, inspect usage, check quotas, and manage agent-safe keys or provider connections without exposing upstream provider secrets.

## Install With skills.sh

After this repo is published, install the Relay0 skill with:

```bash
npx skills add relayzer0/relay0-skills --skill relay0
```

Install globally for Codex without prompts:

```bash
npx skills add relayzer0/relay0-skills --skill relay0 -g -a codex -y
```

List available skills first:

```bash
npx skills add relayzer0/relay0-skills --list
```

For a direct Git URL:

```bash
npx skills add https://github.com/relayzer0/relay0-skills --skill relay0
```

## Configure

```bash
export RELAY0_BASE_URL="https://api.example.com/v1"
export RELAY0_API_KEY="sk-..."          # Relay0 gateway key
export RELAY0_APP_URL="https://app.example.com"
export RELAY0_AGENT_KEY="$RELAY0_API_KEY"
```

Use a Relay0 gateway key assigned to an owner, admin, or agent sub-account for `/api/cloud/agent/*` endpoints.

## Agent Prompt

```text
Use $relay0 to route requests through my Relay0 gateway. Discover visible models first. Never ask for upstream provider secrets.
```

## Manual Use

If an agent cannot use `skills.sh`, paste this once the repo is public:

```text
Read this skill and use it: https://raw.githubusercontent.com/relayzer0/relay0-skills/main/skills/relay0/SKILL.md
```

## Files

| Path | Purpose |
|---|---|
| `relay0/SKILL.md` | Core agent instructions |
| `relay0/references/agent-api.md` | Detailed agent endpoint examples |
| `relay0/agents/openai.yaml` | Agent marketplace metadata |
