# Chat API Credentials And Test Values Guide

This guide explains how to get every value a frontend or backend developer commonly asks for when integrating the single-prompt chat API.

Use this Render base URL:

```text
https://nexiflowai-single-prompt-agent-tool.onrender.com
```

Management/session API routes use:

```text
https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1
```

Public website widget routes use:

```text
https://nexiflowai-single-prompt-agent-tool.onrender.com/widget
```

## Which Credential Do I Need?

| Route type | Example routes | Credential |
|---|---|---|
| Admin/sandbox agent management | `/api/v1/agents`, `/api/v1/tools`, `/api/v1/api-keys`, `/api/v1/public-keys` | `Authorization: Bearer <member_jwt>` or `X-API-Key: <tenant_api_key>` |
| Internal sandbox chat testing | `/api/v1/agents/{agent_id}/test-session`, `/api/v1/sessions/{session_id}/stream` | API key with `sessions:write`, or member JWT |
| Production server-to-server chat | `/api/v1/sessions`, `/api/v1/sessions/{session_id}/stream` | API key with `sessions:write`, or member JWT |
| Public browser website widget | `/widget/init`, `/widget/sessions/{session_id}/message` | Public key `nxf_pk_...`, then widget `session_token` |

Do not expose tenant API keys or member JWTs in public browser code.

## Known Example Values

These values are examples from test/QA data and show the expected shape. Do not assume these are the right values for every tenant or agent.

```text
Tenant ID: 00000000-0000-0000-0000-000000000001
Agent ID: ea517389-f9d8-448a-8080-34a6225509fa
Session ID: fc8c31b6-96cd-4ee8-a65f-d9311ed4f4f1
Channel: chat
Engine: single_prompt
```

Example placeholder tenant API key shape:

```text
nxf_111111111111111111111111111111111111111111111111
```

Example placeholder public widget key shape:

```text
nxf_pk_6fc2f7ef0bcb03e59307f278f2f51fb57b44fedc4a1b9ed2
```

## Tenant ID

The tenant ID scopes all management/session data.

Use the tenant ID that owns the agent. If the API key already belongs to one tenant, you can omit `X-Tenant-ID`; if you send it, it must match the API key tenant.

Example header with member JWT:

```http
Authorization: Bearer <member_jwt>
X-Tenant-ID: 00000000-0000-0000-0000-000000000001
Content-Type: application/json
```

Example header with API key:

```http
X-API-Key: nxf_111111111111111111111111111111111111111111111111
Content-Type: application/json
```

## Member JWT

A member JWT is for trusted admin/backend code, not public websites.

You normally get it from the identity/login service. For dev/testing, a backend operator can mint one with the same `SECRET_KEY` used by the Render deployment:

```bash
python -c "from src.core.config.security import create_access_token; print(create_access_token('22222222-2222-4222-8222-222222222222', '00000000-0000-0000-0000-000000000001'))"
```

Important: a locally minted JWT works on Render only if local `SECRET_KEY` is identical to Render's `SECRET_KEY`.

Use it like:

```http
Authorization: Bearer <member_jwt>
X-Tenant-ID: 00000000-0000-0000-0000-000000000001
```

If you see this error, the request is missing a valid JWT/API key:

```json
{
  "error": "AUTHENTICATION_ERROR",
  "message": "A valid API key or member Bearer token is required",
  "details": {
    "reason": "missing_authenticated_tenant"
  }
}
```

## Tenant API Key

Use tenant API keys for trusted server-to-server calls or an internal admin/sandbox UI.

The full API key is returned only once when it is created. Store it securely. The database stores only its hash, so you cannot retrieve the full key later.

### Create A Sessions-Only API Key

Requires an existing admin/member JWT or full-access/admin API key.

```bash
curl -sS -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/api-keys" \
  -H "Authorization: Bearer <admin_member_jwt>" \
  -H "X-Tenant-ID: 00000000-0000-0000-0000-000000000001" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "name": "Frontend sandbox session key",
    "permissions": {
      "scopes": ["sessions:write", "sessions:read"]
    },
    "rate_limit_requests": 300,
    "rate_limit_window": 60
  }'
```

Example response:

```json
{
  "id": "63a7fd44-5850-4be2-8c88-3068bd6b5f54",
  "key": "nxf_111111111111111111111111111111111111111111111111",
  "key_prefix": "nxf_1111",
  "name": "Frontend sandbox session key",
  "permissions": {
    "scopes": ["sessions:write", "sessions:read"]
  },
  "rate_limit_requests": 300,
  "rate_limit_window": 60,
  "created_at": "2026-05-07T06:00:00Z",
  "expires_at": null
}
```

Use the returned `key` like:

```http
X-API-Key: nxf_111111111111111111111111111111111111111111111111
Content-Type: application/json
```

## Public Widget Key

Use public keys for browser website integrations. Public keys are safe to place in frontend configuration because they can only initialize widget sessions for configured agent IDs and allowed domains.

Public key format:

```text
nxf_pk_<48 hex chars>
```

### Create A Public Widget Key

Requires admin scope. Call this from backend/admin code, never from public browser code.

```bash
curl -sS -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/public-keys" \
  -H "Authorization: Bearer <admin_member_jwt>" \
  -H "X-Tenant-ID: 00000000-0000-0000-0000-000000000001" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "name": "Website widget key",
    "agent_ids": ["ea517389-f9d8-448a-8080-34a6225509fa"],
    "allowed_domains": ["example.com", "www.example.com", "localhost:*"],
    "rate_limit_requests": 120,
    "rate_limit_window": 60,
    "expires_at": null
  }'
```

Example response:

```json
{
  "id": "8e39464a-1d1b-4d9f-bf5b-91a7214d7ed2",
  "key": "nxf_pk_6fc2f7ef0bcb03e59307f278f2f51fb57b44fedc4a1b9ed2",
  "key_prefix": "nxf_pk_6fc2",
  "name": "Website widget key",
  "agent_ids": ["ea517389-f9d8-448a-8080-34a6225509fa"],
  "allowed_domains": ["example.com", "www.example.com", "localhost:*"],
  "rate_limit_requests": 120,
  "rate_limit_window": 60,
  "created_at": "2026-05-07T06:05:00Z",
  "expires_at": null
}
```

The full public key is also returned only once.

## Create A Sandbox Test Session

Use this for internal admin/sandbox UI testing against the current draft agent version.

```bash
curl -sS -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/agents/ea517389-f9d8-448a-8080-34a6225509fa/test-session" \
  -H "X-API-Key: nxf_111111111111111111111111111111111111111111111111" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "initial_variables": {
      "customer_name": "Ava",
      "plan_name": "Growth"
    }
  }'
```

Example response:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "current_node_id": "prompt_mode",
  "agent_version": 1,
  "begin_message": "Hi Ava, how can I help today?",
  "mode": "test"
}
```

Then stream a message:

```bash
curl -N -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/sessions/65c86045-32b4-4d9a-a4df-0fd79683bb74/stream" \
  -H "X-API-Key: nxf_111111111111111111111111111111111111111111111111" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"user_input":"Can you explain the Growth plan?"}'
```

## Create A Public Website Session

Use this for public visitor chat.

```bash
curl -sS -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/widget/init" \
  -H "Origin: https://www.example.com" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "public_key": "nxf_pk_6fc2f7ef0bcb03e59307f278f2f51fb57b44fedc4a1b9ed2",
    "agent_id": "ea517389-f9d8-448a-8080-34a6225509fa",
    "visitor_id": "visitor-123",
    "metadata": {
      "page": "/pricing",
      "locale": "en-US"
    }
  }'
```

Example response:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "session_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "agent_name": "Website Support Agent",
  "greeting": "Hi Ava, how can I help today?"
}
```

Then send a visitor message:

```bash
curl -N -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/widget/sessions/65c86045-32b4-4d9a-a4df-0fd79683bb74/message" \
  -H "Authorization: Bearer <session_token_from_widget_init>" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"message":"Can you explain the Growth plan?"}'
```

## Value Checklist

Before asking another developer to integrate, give them this set:

For internal sandbox/admin UI:

```text
API base URL: https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1
Agent ID: <agent_uuid>
Tenant API key: nxf_...
Test session endpoint: POST /agents/{agent_id}/test-session
Stream endpoint: POST /sessions/{session_id}/stream
```

For public website widget:

```text
Widget base URL: https://nexiflowai-single-prompt-agent-tool.onrender.com
Agent ID: <published_chat_agent_uuid>
Public key: nxf_pk_...
Allowed domain: <frontend_origin_domain>
Init endpoint: POST /widget/init
Message endpoint: POST /widget/sessions/{session_id}/message
```

## Troubleshooting

| Error | Meaning | Fix |
|---|---|---|
| `missing_authenticated_tenant` | `/api/v1/*` request has no valid member JWT or tenant API key | Send `Authorization: Bearer <member_jwt>` or `X-API-Key: nxf_...`. |
| `API key does not grant required scope` | API key exists but lacks route permission | Create/update key with needed scopes such as `sessions:write`, `agents:write`, or `admin`. |
| `Invalid public key` | `/widget/init` public key is wrong, revoked, expired, or not saved | Create a new public key and use the full `key` returned once. |
| `Origin domain is not allowed` | Browser `Origin` is not in `allowed_domains` | Add the frontend domain, for example `localhost:*` or `www.example.com`. |
| `Agent has no published version` | Widget/production session tried to use an unpublished agent | Publish the agent first with `POST /api/v1/agents/{agent_id}/publish`. |
| `Agent not found` | Wrong tenant/agent ID or flow-engine agent | Use an agent owned by the same tenant with `engine = "single_prompt"` and `channel = "chat"`. |
| `422` JSON error | Invalid JSON or wrong body shape | Remove trailing commas and match the examples exactly. |
