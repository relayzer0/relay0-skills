# Relay0 Agent API Reference

Use these endpoints only with `RELAY0_APP_URL` and `Authorization: Bearer $RELAY0_AGENT_KEY`. The bearer key is a Relay0 gateway key assigned to an owner, admin, or agent user.

Never log bearer keys. Never ask for upstream provider API keys or OAuth tokens.

## Common headers

```bash
-H "Authorization: Bearer $RELAY0_AGENT_KEY"
-H "Content-Type: application/json"
```

## Summary

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/summary" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

Response shape:

```json
{
  "tenant": { "id": "tenant-id", "name": "Workspace", "slug": "workspace" },
  "actor": { "id": "user-id", "email": "agent@example.com", "role": "agent", "name": "Ops Agent" },
  "authMode": "bearer",
  "apiKey": { "id": "key-id", "name": "agent key", "masked": "sk-abc...xyz12" },
  "permissions": { "viewUsage": true, "viewQuotas": true, "manageKeys": true, "manageConnections": true, "manageUsers": false },
  "counts": { "keys": 2, "providers": 3, "users": 0 },
  "usage30d": { "totalRequests": 42, "totalTokens": 120000, "totalCost": 1.23, "successRate": 98 }
}
```

Use this first for health checks and to decide what the agent is allowed to do.

## Usage

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/usage?period=7d&pageSize=20" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

`period` may be `today`, `24h`, `7d`, `30d`, `60d`, or `all`. `pageSize` is capped at `100`.

Response shape:

```json
{
  "stats": {
    "totalRequests": 42,
    "totalPromptTokens": 80000,
    "totalCompletionTokens": 40000,
    "totalTokens": 120000,
    "totalCost": 1.23,
    "successRate": 98
  },
  "details": [
    {
      "timestamp": "2026-06-06T00:00:00.000Z",
      "provider": "openai",
      "model": "cx/gpt-5.5",
      "status": "success",
      "promptTokens": 1000,
      "completionTokens": 500,
      "latency": 1200,
      "apiKeyId": "key-id",
      "userId": "user-id"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 42 },
  "scoped": true
}
```

Owners/admins see workspace-scoped usage. Agent users see only rows tied to their assigned user/key scope.

## Gateway Keys

List visible keys:

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/keys" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

Response shape:

```json
{
  "keys": [
    {
      "id": "key-id",
      "name": "agent key",
      "masked": "sk-abc...xyz12",
      "isActive": true,
      "tenantAccess": [{ "tenantId": "tenant-id", "userId": "user-id", "projectId": "" }]
    }
  ]
}
```

Create a key:

```bash
curl -X POST "$RELAY0_APP_URL/api/cloud/agent/keys" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"agent rollout key"}'
```

Admin-only tenant-wide key:

```bash
curl -X POST "$RELAY0_APP_URL/api/cloud/agent/keys" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"workspace automation key","scope":"tenant"}'
```

Create response includes the full new Relay0 gateway key once. After that, keys are masked only.

Patch a key:

```bash
curl -X PATCH "$RELAY0_APP_URL/api/cloud/agent/keys" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"id":"key-id","isActive":false,"name":"disabled after quota spike"}'
```

Before disabling a key, list keys and usage, then report the reason.

## Provider Connections

List visible connections:

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/connections" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

Response shape:

```json
{
  "connections": [
    {
      "id": "connection-id",
      "provider": "openai",
      "authType": "oauth",
      "name": "li***s@p***n.me",
      "email": "li***s@p***n.me",
      "isActive": true,
      "testStatus": "ok",
      "tenantAccess": [{ "tenantId": "tenant-id", "userId": "user-id", "projectId": "" }]
    }
  ]
}
```

Provider secrets are never returned. OAuth emails and API keys are redacted.

Patch a connection:

```bash
curl -X PATCH "$RELAY0_APP_URL/api/cloud/agent/connections" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"id":"connection-id","isActive":true,"name":"primary codex account"}'
```

Only mutate visible connections. Ask before disabling unless the user explicitly authorized automatic mitigation.

## Quotas

Fetch quotas for up to 12 visible connections:

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/quotas" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

Fetch one connection:

```bash
curl "$RELAY0_APP_URL/api/cloud/agent/quotas?connectionId=connection-id" \
  -H "Authorization: Bearer $RELAY0_AGENT_KEY"
```

Response shape:

```json
{
  "quotas": [
    {
      "connectionId": "connection-id",
      "provider": "openai",
      "name": "primary codex account",
      "available": true,
      "usage": { "remaining": 123, "limit": 500, "resetAt": "2026-06-07T00:00:00.000Z" }
    },
    {
      "connectionId": "connection-id-2",
      "provider": "anthropic",
      "name": "byok key",
      "available": false,
      "message": "Quota API unavailable for this connection type"
    }
  ],
  "truncated": false
}
```

Quota shape depends on upstream provider support. If unavailable, report status without guessing limits.

## Mutation Guidance

For every mutation:

1. Read current state first.
2. Confirm the key or connection is visible to this actor.
3. Mutate only allowed fields: `name`, `isActive`, and backend-supported status fields.
4. Do not include unknown fields, upstream secrets, or tenant ids in patches.
5. Summarize object id, old state, new state, and reason.

## Errors

- `401`: missing/invalid Relay0 key or expired dashboard session.
- `403`: actor role or agent permission denied.
- `404`: key or connection is not visible to this actor.
- `400`: bad period or malformed JSON body.
