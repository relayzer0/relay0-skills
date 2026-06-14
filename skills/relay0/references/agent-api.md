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

Response shape (the real payload — `details[]` carries the routing connection and the full request/response):

```json
{
  "stats": {
    "totalRequests": 42,
    "totalPromptTokens": 80000,
    "totalCompletionTokens": 40000,
    "totalTokens": 120000,
    "totalCost": 1.23,
    "successRate": 98,
    "byProvider": {},
    "byModel": {},
    "byApiKey": {},
    "byEndpoint": {},
    "last10Minutes": []
  },
  "details": [
    {
      "id": "2026-06-06T00:00:00.000Z-abc123-gpt-5-5",
      "timestamp": "2026-06-06T00:00:00.000Z",
      "provider": "codex",
      "model": "gpt-5.5",
      "connectionId": "0773d6fb-2cf8-4cdf-a222-076232759bc4",
      "status": "success",
      "latency": { "ttft": 800, "total": 1200 },
      "tokens": { "prompt_tokens": 1000, "completion_tokens": 500 },
      "apiKeyId": "key-id",
      "userId": "user-id",
      "request": { "model": "codex/gpt-5.5", "messages": [], "stream": true },
      "response": { "content": "...", "finish_reason": "completed", "thinking": null }
    },
    {
      "id": "2026-06-06T00:00:00.000Z-def456-gpt-5-5",
      "timestamp": "2026-06-06T00:00:00.000Z",
      "provider": "codex",
      "model": "gpt-5.5",
      "connectionId": "59208fe6-985b-4f28-a777-164e5ddcb04c",
      "status": "error",
      "latency": { "ttft": 0, "total": 3153 },
      "tokens": { "prompt_tokens": 0, "completion_tokens": 0 },
      "response": {
        "error": "{\"error\":{\"message\":\"Your authentication token has been invalidated...\",\"code\":\"token_invalidated\"},\"status\":401}",
        "status": 401
      }
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "totalItems": 42, "totalPages": 3, "hasNext": true, "hasPrev": false },
  "scoped": true
}
```

Owners/admins see workspace-scoped usage. Agent users see only rows tied to their assigned user/key scope.

When debugging a flaky model route, read each `details[]` row's `connectionId` and `status`: a single user prompt can produce an `error` row on one connection followed by a `success` row on another when Relay0 fails over. Large `request`/`response`/`providerRequest` bodies may be truncated with `_truncated`, `_originalSize`, and `_preview` fields.

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
      "provider": "codex",
      "authType": "oauth",
      "name": "li***s@p***n.me",
      "email": "li***s@p***n.me",
      "priority": 1,
      "isActive": true,
      "testStatus": "active",
      "healthStatus": "healthy",
      "lastHealthResult": "success",
      "lastCheckedAt": "2026-06-14T20:39:03.049Z",
      "lastError": null,
      "lastErrorAt": null,
      "consecutiveFailures": 0,
      "expiresAt": "2026-06-16T22:53:07.637Z",
      "providerSpecificData": { "chatgptPlanType": "pro" },
      "tenantAccess": [{ "tenantId": "tenant-id", "userId": "user-id", "projectId": "" }]
    }
  ]
}
```

Provider secrets are never returned. OAuth emails and API keys are redacted.

Health/routing fields to read before acting:

- `healthStatus`: `healthy` or `unhealthy`.
- `lastHealthResult`: for example `success` or `auth_failed`.
- `lastError` / `lastErrorAt`: for example `Token invalid or revoked`.
- `priority`: Relay0 tries connections in ascending priority order; a healthy lower-priority connection backs up a failed higher-priority one.
- `modelLock_<model>`: a timestamp means that model is temporarily rate-limited / locked on this upstream.
- `consecutiveFailures` and `backoffLevel`: rising values indicate Relay0 is backing off this connection.

An OAuth connection reporting `token_invalidated` / `auth_failed` needs a human re-auth in the Relay0 app. An agent cannot repair it via the API — do not auto-disable it; tell the user to reconnect.

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

Connection mutation limits: an agent can rename, enable, or disable a **visible** connection via `PATCH /api/cloud/agent/connections`. An agent **cannot** re-authenticate an OAuth connection — expired or revoked tokens (`token_invalidated`, `auth_failed`) require the user to reconnect in the Relay0 dashboard. Never disable a connection on a transient auth error without asking.

## Errors

- `401`: missing/invalid Relay0 key or expired dashboard session.
- `403`: actor role or agent permission denied.
- `404`: key or connection is not visible to this actor.
- `400`: bad period or malformed JSON body.
