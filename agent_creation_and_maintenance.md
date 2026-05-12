# Chat Agent Creation And Maintenance API

This document explains how to create, edit, version, publish, test, and maintain a single-prompt chat agent.

It is written for implementation work. A junior developer should be able to read this file and know which route to call, which headers to send, what body shape is expected, what a successful response looks like, and what errors to handle.

For credential setup and copy-paste test values, see `docs/api_docs/chat/credentials_and_test_values.md`.

## Base URL

Render deployment:

```text
https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1
```

Local development:

```text
http://localhost:8001/api/v1
```

Production:

```text
https://<your-api-host>/api/v1
```

All paths in this document are relative to `/api/v1`.

Example:

```text
POST /agents
```

means:

```text
POST https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/agents
```

## Frontend Developer Quick Start

Use this section when wiring an admin/sandbox UI.

### Which API Should I Use?

| Use case | Use these routes | Credential to send |
|---|---|---|
| Admin creates/edits/tests agents | `/api/v1/agents`, `/api/v1/tools`, `/api/v1/sessions` | `Authorization: Bearer <member_jwt>` or `X-API-Key: <tenant_api_key>` |
| Internal sandbox "Test" button | `POST /api/v1/agents/{agent_id}/test-session`, then `POST /api/v1/sessions/{session_id}/stream` | API key with `sessions:write`, or member JWT |
| Public website visitor chat | `/widget/init`, `/widget/sessions/{session_id}/message` | Public key `nxf_pk_...`, then widget `session_token` |

Do not put tenant API keys or member JWTs in public browser code. Public websites should use `/widget/*`.

### Known Real Example IDs From Render QA

These IDs are examples from a real Render QA run. They show the expected UUID shape and tenant relationship; create your own agent/session for new tests.

```text
Base URL: https://nexiflowai-single-prompt-agent-tool.onrender.com
Tenant ID: 00000000-0000-0000-0000-000000000001
Agent ID: ea517389-f9d8-448a-8080-34a6225509fa
Session ID: fc8c31b6-96cd-4ee8-a65f-d9311ed4f4f1
Engine: single_prompt
Channel: chat
```

### Common Integration Mistakes

| Symptom | Cause | Fix |
|---|---|---|
| `missing_authenticated_tenant` | Missing/invalid `Authorization` or `X-API-Key` on `/api/v1/*` route | Send a valid member JWT or tenant API key. `X-Tenant-ID` alone is not enough on Render. |
| `RESOURCE_NOT_FOUND` for an agent | Wrong tenant, wrong agent ID, or flow-engine agent ID | Use an agent owned by the same tenant and created with `engine = "single_prompt"`. |
| `422` request validation error | Invalid JSON or wrong body shape | Remove trailing commas and send the exact JSON shown in examples. |
| Widget route says token is invalid | Using member JWT/API key instead of widget session token | Call `/widget/init`, then use returned `session_token`. |

## Authentication

Management routes require one of these authentication methods:

```http
X-API-Key: <tenant_api_key>
```

or:

```http
Authorization: Bearer <member_jwt>
```

Also include:

```http
X-Tenant-ID: <tenant_uuid>
Content-Type: application/json
Accept: application/json
```

When using an API key, `X-Tenant-ID` must match the API key tenant if it is sent. If the API key already fixes the tenant, the header can be omitted.

### Auth Examples

Member JWT example:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
X-Tenant-ID: 00000000-0000-0000-0000-000000000001
Content-Type: application/json
Accept: application/json
```

API key example:

```http
X-API-Key: nxf_111111111111111111111111111111111111111111111111
Content-Type: application/json
Accept: application/json
```

API keys are managed by the **Identity Management** service. Please use the Identity Management API or dashboard to generate a tenant API key. The database stores only its hash, so you cannot retrieve the full key later.

## Required API Key Scopes

API keys can be full-access or scoped. These routes use:

| Scope | Routes |
|---|---|
| `agents:read` | list agents, get agent, versions, draft status |
| `agents:write` | create, update, publish, version, archive, restore |
| `sessions:write` | create test session |
| `tools:read` | list attached tools |
| `tools:write` | create/update/delete tools, attach/detach tools |

Full-access keys use:

```json
{
  "access": "full"
}
```

Scoped keys can use:

```json
{
  "scopes": ["agents:read", "agents:write", "sessions:write"]
}
```

## Single-Prompt Chat Invariants

These rules matter for every route in this document:

1. This service only manages agents where `engine = "single_prompt"`.
2. Chat agents must use `channel = "chat"`.
3. Chat agents must not require or configure `voice_id`.
4. `channel` and `engine` are immutable after creation.
5. Published versions are immutable. To edit a published agent, create a new version, edit the new draft, then publish it.
6. Flow-agent IDs from the companion flow-agent service return `404`.
7. Agent version config is version-scoped. Sessions keep using the version they were created with.

## Standard Error Format

Application errors normally return:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent draft is not ready to publish",
  "details": {
    "validation_errors": ["System prompt is required to publish the agent."]
  }
}
```

FastAPI request-body validation errors return:

```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "name"],
      "msg": "Field required",
      "input": {}
    }
  ]
}
```

Common status codes:

| Status | Meaning | Common cause |
|---:|---|---|
| `400` | Validation error | invalid config, published version edit, voice-only fields on chat agent |
| `401` | Authentication error | missing or invalid API key/JWT |
| `403` | Forbidden | API key does not grant the required scope |
| `404` | Not found | resource does not exist, belongs to another tenant, or belongs to the flow-agent engine |
| `422` | Request schema error | missing required field or invalid UUID/type |
| `429` | Rate or budget limit | request rate exceeded or LLM budget exceeded |
| `500` | Server error | unexpected backend failure |

## Recommended Agent Creation Flow

Use this order in a frontend or admin backend:

1. Read model options: `GET /system/llm-models`
2. Read language options: `GET /system/languages`
3. Create a draft chat agent: `POST /agents`
4. Check readiness: `GET /agents/{agent_id}/draft-status`
5. Create a test session: `POST /agents/{agent_id}/test-session`
6. Stream test messages: `POST /sessions/{session_id}/stream`
7. Publish: `POST /agents/{agent_id}/publish`
8. For website usage, create a public key and use `/widget/*`.

## Response Engine Shape

Prefer the canonical `response_engine` field for new code.

```json
{
  "type": "single_prompt",
  "general_prompt": "The system prompt used for every turn.",
  "begin_message": "Optional first assistant message.",
  "begin_message_mode": "agent_speaks_first",
  "llm_settings": {
    "model_id": "gemini-2.5-flash-lite",
    "temperature": 0.4
  },
  "chat_settings": {
    "language_code": "en-US",
    "timezone": "UTC",
    "auto_close_message": "This chat has ended.",
    "end_chat_after_silence_ms": 300000
  },
  "guardrails": {},
  "tool_calling_config": {},
  "knowledge_config": {},
  "memory_config": {},
  "post_chat_analysis_config": {
    "enabled": false,
    "items": []
  }
}
```

Field notes:

| Field | Type | Required | Notes |
|---|---|---:|---|
| `type` | string | no | Must be `single_prompt` when sent. |
| `general_prompt` | string | yes before test/publish | Main system prompt. Also returned as legacy `global_settings.system_prompt`. |
| `begin_message` | string or null | no | First assistant message. Can contain `{{variable_name}}` templates. |
| `begin_message_mode` | string | no | `agent_speaks_first`, `user_speaks_first`, or `dynamic`. |
| `llm_settings.model_id` | string | recommended | Must match `GET /system/llm-models`. |
| `llm_settings.temperature` | number | no | Usually `0.0` to `1.0`. Lower is more deterministic. |
| `chat_settings.language_code` | string | recommended | Must match `GET /system/languages` when provided. |
| `chat_settings.timezone` | string | no | IANA timezone such as `UTC` or `America/New_York`. |
| `post_chat_analysis_config.enabled` | boolean | no | Enables end-of-session extraction. |
| `post_chat_analysis_config.items` | array | no | Analysis fields extracted after session end. |

`begin_message_mode` behavior:

| Mode | Runtime behavior |
|---|---|
| `agent_speaks_first` | Store and return `begin_message` as the first assistant transcript. |
| `user_speaks_first` | Do not send an opener. Session starts waiting for the user. |
| `dynamic` | Reserved for generated openers. Treat as not required for current chat MVP. |

## Dynamic Variables

Dynamic variables are defaults or runtime values that can be rendered into prompts and begin messages using `{{key}}`.

Example:

```json
[
  {
    "key": "customer_name",
    "label": "Customer Name",
    "description": "Name shown in the greeting and prompt context.",
    "data_type": "string",
    "default_value": "there",
    "is_required": false,
    "source": "agent_default"
  }
]
```

Field rules:

| Field | Type | Required | Notes |
|---|---|---:|---|
| `key` | string | yes | Must start with a letter or underscore and contain only letters, numbers, and underscores. |
| `label` | string or null | no | Friendly label for UI. |
| `description` | string | no | Helps builders understand the variable. |
| `data_type` | string | no | `string`, `number`, `boolean`, `enum`, `object`, or `array`. Defaults to `string`. |
| `default_value` | any JSON | no | Loaded into new sessions before `initial_variables`. |
| `is_required` | boolean | no | UI hint; runtime still accepts omitted values unless the prompt requires them. |
| `source` | string | no | `agent_default`, `runtime`, `tool_extracted`, or `system`. |

## Create A Chat Agent

Creates a new draft agent and version `1`.

```http
POST /agents
```

Required scope: `agents:write`

### Request Headers

```http
X-API-Key: <tenant_api_key>
X-Tenant-ID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Content-Type: application/json
Accept: application/json
```

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `name` | string | yes | 1 to 255 chars. |
| `description` | string | no | Defaults to empty string. Max 2000 chars. |
| `channel` | string | yes | Use `chat`. |
| `engine` | string | no | Use `single_prompt`. Defaults to `single_prompt`. |
| `global_settings` | object | no | Legacy config shape. Prefer `response_engine`. |
| `response_engine` | object | recommended | Canonical single-prompt config. |
| `dynamic_variables` | array | no | Version-scoped variables. |
| `webhook_url` | string or null | no | HTTPS URL for webhook events. SSRF validation applies. |
| `webhook_events` | array or null | no | Event names such as `chat_started`, `chat_ended`. |

### Full Example Request

```json
{
  "name": "Website Support Agent",
  "description": "Answers support and pricing questions on the marketing website.",
  "channel": "chat",
  "engine": "single_prompt",
  "response_engine": {
    "type": "single_prompt",
    "general_prompt": "You are Riley, a concise support agent for NexiFlow. Help visitors understand the product, pricing, and setup. Ask one question at a time. If the visitor asks for account-specific billing data, ask for their email and say a support specialist may follow up.",
    "begin_message": "Hi {{customer_name}}, how can I help today?",
    "begin_message_mode": "agent_speaks_first",
    "llm_settings": {
      "model_id": "gemini-2.5-flash-lite",
      "temperature": 0.4
    },
    "chat_settings": {
      "language_code": "en-US",
      "timezone": "UTC",
      "auto_close_message": "Thanks for chatting with us.",
      "end_chat_after_silence_ms": 300000
    },
    "guardrails": {
      "disallowed_topics": ["legal advice", "medical advice"],
      "handoff_when_uncertain": true
    },
    "post_chat_analysis_config": {
      "enabled": true,
      "items": [
        {
          "name": "lead_intent",
          "description": "What the visitor wanted.",
          "type": "enum",
          "choices": ["pricing", "support", "demo", "other"]
        },
        {
          "name": "needs_follow_up",
          "description": "Whether a human should follow up.",
          "type": "boolean"
        }
      ]
    }
  },
  "dynamic_variables": [
    {
      "key": "customer_name",
      "label": "Customer Name",
      "description": "Visitor name if known.",
      "data_type": "string",
      "default_value": "there",
      "is_required": false,
      "source": "agent_default"
    }
  ],
  "webhook_url": "https://example.com/webhooks/nexiflow",
  "webhook_events": ["chat_started", "chat_ended", "chat_failed"]
}
```

### Success Response

Status: `201 Created`

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "status": "draft",
  "current_version": 1
}
```

### Common Errors

Invalid chat voice setting:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Chat agents cannot configure voice-only settings",
  "details": {
    "voice_only_fields": ["voice_id"]
  }
}
```

Invalid model:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Global settings contain invalid registry references",
  "details": {
    "validation_errors": [
      "Invalid model_id 'bad-model'. Valid options: gemini-3.1-flash-lite, gemini-2.5-flash-lite"
    ]
  }
}
```

Invalid variable key:

Status: `422 Unprocessable Entity`

```json
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body", "dynamic_variables", 0, "key"],
      "msg": "Value error, Variable key must start with a letter or underscore and contain only letters, numbers, and underscores"
    }
  ]
}
```

## List Agents

Returns a paginated list of non-archived single-prompt agents for the tenant.

```http
GET /agents?limit=50&offset=0&search=support&status=draft
```

Required scope: `agents:read`

### Query Parameters

| Name | Type | Required | Default | Notes |
|---|---|---:|---:|---|
| `limit` | integer | no | `50` | Min `1`, max `200`. |
| `offset` | integer | no | `0` | Pagination offset. |
| `search` | string | no | none | Matches agent name. |
| `status` | string | no | none | `draft`, `published`, or `archived`. `archived` returns only archived agents. |

### Success Response

Status: `200 OK`

```json
{
  "items": [
    {
      "id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
      "tenant_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
      "name": "Website Support Agent",
      "description": "Answers support and pricing questions on the marketing website.",
      "status": "draft",
      "channel": "chat",
      "engine": "single_prompt",
      "current_version": 1,
      "is_archived": false,
      "created_at": "2026-05-03T09:00:00Z",
      "updated_at": "2026-05-03T09:00:00Z",
      "global_settings": {
        "language_code": "en-US",
        "voice_id": null,
        "begin_message": "Hi {{customer_name}}, how can I help today?",
        "system_prompt": "You are Riley, a concise support agent for NexiFlow...",
        "llm_config": {
          "model_id": "gemini-2.5-flash-lite",
          "temperature": 0.4
        },
        "expected_dynamic_variables": [
          {
            "name": "customer_name",
            "description": "Visitor name if known.",
            "type": "string"
          }
        ],
        "post_call_analysis": {
          "enabled": true,
          "items": [
            {
              "name": "lead_intent",
              "description": "What the visitor wanted.",
              "type": "enum",
              "choices": ["pricing", "support", "demo", "other"]
            }
          ]
        },
        "timezone": "UTC",
        "pii_redaction_enabled": false,
        "guardrail_config": {}
      },
      "response_engine": {
        "type": "single_prompt",
        "general_prompt": "You are Riley, a concise support agent for NexiFlow...",
        "begin_message": "Hi {{customer_name}}, how can I help today?",
        "begin_message_mode": "agent_speaks_first",
        "llm_settings": {
          "model_id": "gemini-2.5-flash-lite",
          "temperature": 0.4
        },
        "response_style": {},
        "guardrails": {},
        "voice_settings": {},
        "chat_settings": {
          "language_code": "en-US",
          "timezone": "UTC"
        },
        "tool_calling_config": {},
        "knowledge_config": {},
        "memory_config": {},
        "post_chat_analysis_config": {
          "enabled": true,
          "items": []
        }
      },
      "dynamic_variables": [
        {
          "key": "customer_name",
          "label": "Customer Name",
          "description": "Visitor name if known.",
          "data_type": "string",
          "default_value": "there",
          "is_required": false,
          "source": "agent_default"
        }
      ],
      "webhook_url": "https://example.com/webhooks/nexiflow",
      "webhook_events": ["chat_started", "chat_ended", "chat_failed"],
      "webhook_secret": "f3c0..."
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

## Get Agent

Returns one agent by ID.

```http
GET /agents/{agent_id}
```

Required scope: `agents:read`

### Path Parameters

| Name | Type | Required | Notes |
|---|---|---:|---|
| `agent_id` | UUID | yes | Agent must belong to the tenant and `engine = "single_prompt"`. |

### Success Response

Status: `200 OK`

Response body is the same `AgentResponse` object shown in the list example.

### Common Errors

Agent missing, wrong tenant, archived when not expected, or flow-agent row:

Status: `404 Not Found`

```json
{
  "error": "RESOURCE_NOT_FOUND",
  "message": "Agent not found: 2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "details": {}
}
```

## Update Draft Agent

Updates agent metadata and current draft version config.

```http
PATCH /agents/{agent_id}
```

Required scope: `agents:write`

### Request Body

All fields are optional. Only supplied fields are changed.

| Field | Type | Notes |
|---|---|---|
| `name` | string | New display name. |
| `description` | string | New description. |
| `global_settings` | object | Legacy config. Prefer `response_engine`. |
| `response_engine` | object | Partial canonical config overlay. |
| `dynamic_variables` | array | Replaces the current variable list when supplied. |
| `webhook_url` | string or null | New webhook URL. |
| `webhook_events` | array or null | New webhook event list. |

### Partial Update Example

```json
{
  "name": "Website Support Agent v2",
  "response_engine": {
    "general_prompt": "You are Riley, a concise support agent. For billing requests, collect the visitor email and explain that a specialist will follow up.",
    "llm_settings": {
      "model_id": "gemini-3.1-flash-lite",
      "temperature": 0.3
    },
    "chat_settings": {
      "timezone": "America/New_York"
    }
  }
}
```

### Success Response

Status: `200 OK`

Returns the full updated `AgentResponse`.

### Common Errors

Trying to edit a published version:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot update configuration on a published version. Create a new version before editing configuration.",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "version": 1
  }
}
```

Webhook URL blocked by SSRF validation:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid webhook_url",
  "details": {
    "field": "webhook_url"
  }
}
```

## Draft Status

Checks whether the current draft is ready to test or publish.

```http
GET /agents/{agent_id}/draft-status
```

Required scope: `agents:read`

### Success Response

Status: `200 OK`

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "version": 1,
  "status": "draft",
  "is_ready_to_test": true,
  "is_ready_to_publish": true,
  "checklist": {
    "config_saved": true,
    "has_system_prompt": true,
    "voice_id_configured": true,
    "model_configured": true,
    "language_configured": true
  },
  "validation_errors": [],
  "missing_required_fields": [],
  "invalid_references": [],
  "tool_warnings": [],
  "warnings": []
}
```

Example not-ready response:

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "version": 1,
  "status": "draft",
  "is_ready_to_test": false,
  "is_ready_to_publish": false,
  "checklist": {
    "config_saved": true,
    "has_system_prompt": false,
    "voice_id_configured": true,
    "model_configured": true,
    "language_configured": true
  },
  "validation_errors": ["System prompt is required to publish the agent."],
  "missing_required_fields": ["system_prompt"],
  "invalid_references": [],
  "tool_warnings": [],
  "warnings": []
}
```

## Create New Version

Creates a new draft version by copying the current version config and variables.

```http
POST /agents/{agent_id}/versions
```

Required scope: `agents:write`

### Request Body

No body is required.

### Success Response

Status: `201 Created`

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "version": 2,
  "status": "draft"
}
```

Use this after an agent has been published and you need to edit a new draft.

## List Versions

```http
GET /agents/{agent_id}/versions
```

Required scope: `agents:read`

### Success Response

Status: `200 OK`

```json
[
  {
    "id": "8379ea82-82d6-4bb1-9141-2d0c5c1c22f8",
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "version": 2,
    "is_published": false,
    "published_at": null,
    "created_at": "2026-05-03T09:30:00Z",
    "global_settings": {
      "language_code": "en-US",
      "voice_id": null,
      "begin_message": "Hi {{customer_name}}, how can I help today?",
      "system_prompt": "You are Riley...",
      "llm_config": {
        "model_id": "gemini-3.1-flash-lite",
        "temperature": 0.3
      },
      "expected_dynamic_variables": [],
      "post_call_analysis": {
        "enabled": false,
        "items": []
      },
      "timezone": "UTC",
      "pii_redaction_enabled": false,
      "guardrail_config": {}
    },
    "response_engine": {
      "type": "single_prompt",
      "general_prompt": "You are Riley...",
      "begin_message": "Hi {{customer_name}}, how can I help today?",
      "begin_message_mode": "agent_speaks_first",
      "llm_settings": {
        "model_id": "gemini-3.1-flash-lite",
        "temperature": 0.3
      },
      "response_style": {},
      "guardrails": {},
      "voice_settings": {},
      "chat_settings": {},
      "tool_calling_config": {},
      "knowledge_config": {},
      "memory_config": {},
      "post_chat_analysis_config": {}
    },
    "dynamic_variables": []
  }
]
```

## Publish Current Draft

Publishes the agent's current draft version.

```http
POST /agents/{agent_id}/publish
```

Required scope: `agents:write`

### Request Body

No body is required.

### Success Response

Status: `200 OK`

```json
{
  "status": "published",
  "version": 2
}
```

### Common Errors

Missing system prompt:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent draft is not ready to publish",
  "details": {
    "validation_errors": ["System prompt is required to publish the agent."]
  }
}
```

Already published:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent is already published",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "version": "2"
  }
}
```

## Publish Versioned Endpoint

Publishes the current draft version and requires the path version to match the agent's current version.

```http
POST /agents/{agent_id}/versions/{version}/publish
```

Required scope: `agents:write`

### Common Error

Path version is stale:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Version mismatch: publish endpoint targets the current draft version only.",
  "details": {
    "requested_version": 1,
    "current_version": 2
  }
}
```

## Create Test Session

Creates a session against the current draft version. This is for sandbox/test UI only. Production sessions use `POST /sessions`.

```http
POST https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/agents/{agent_id}/test-session
```

Required scope: `sessions:write`

### Headers

Use a tenant API key:

```http
X-API-Key: nxf_111111111111111111111111111111111111111111111111
Content-Type: application/json
Accept: application/json
```

Or use a member JWT:

```http
Authorization: Bearer <member_jwt>
X-Tenant-ID: 00000000-0000-0000-0000-000000000001
Content-Type: application/json
Accept: application/json
```

If you get this error, authentication is missing or invalid:

```json
{
  "error": "AUTHENTICATION_ERROR",
  "message": "A valid API key or member Bearer token is required",
  "details": {
    "reason": "missing_authenticated_tenant"
  }
}
```

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `initial_variables` | object | no | Values loaded into session state before the first turn. Overrides variable defaults. |

### Example Request

Full request:

```bash
curl -sS -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/agents/ea517389-f9d8-448a-8080-34a6225509fa/test-session" \
  -H "X-API-Key: nxf_111111111111111111111111111111111111111111111111" \
  -H "Content-Type: application/json" \
  -d '{
    "initial_variables": {
      "customer_name": "Ava",
      "plan_name": "Growth"
    }
  }'
```

JSON body only:

```json
{
  "initial_variables": {
    "customer_name": "Ava",
    "plan_name": "Growth"
  }
}
```

Invalid JSON example:

```json
{
  "initial_variables": {
    "customer_name": "Ava",
    "plan_name": "Growth"
  }
},,
```

The trailing `,,` makes the request invalid. Remove it.

### Success Response

Status: `201 Created`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "current_node_id": "prompt_mode",
  "agent_version": 2,
  "begin_message": "Hi Ava, how can I help today?",
  "mode": "test"
}
```

Next call:

```http
POST /sessions/65c86045-32b4-4d9a-a4df-0fd79683bb74/stream
```

### Common Error

Draft is not ready to test:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent draft is not ready to test",
  "details": {
    "validation_errors": ["System prompt is required to test the agent."]
  }
}
```

## Create Tool

Creates a tenant-scoped tool definition. Single-prompt tools are general tools attached to an agent version and available throughout the chat.

```http
POST /tools
```

Required scope: `tools:write`

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `name` | string | yes | Unique per tenant. Used as the LLM tool name. |
| `description` | string | no | Explain when the model should use the tool. |
| `tool_type` | string | no | `webhook` or `builtin`. Defaults to `webhook`. |
| `endpoint_url` | string | required for webhook | HTTPS endpoint. SSRF validation applies. |
| `auth_config` | object or null | no | Auth headers/config used by the executor. |
| `input_schema` | object or null | recommended | JSON Schema for tool input. |
| `output_schema` | object or null | no | JSON Schema for response shape. |

### Example Request

```json
{
  "name": "lookup_order_status",
  "description": "Use when a visitor asks for the status of an order and provides an order ID.",
  "tool_type": "webhook",
  "endpoint_url": "https://api.example.com/nexiflow/tools/order-status",
  "auth_config": {
    "headers": {
      "Authorization": "Bearer ${ORDER_TOOL_TOKEN}"
    },
    "safety": {
      "requires_confirmation": false
    }
  },
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": {
        "type": "string",
        "description": "Customer order ID."
      }
    },
    "required": ["order_id"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "status": {"type": "string"},
      "estimated_delivery": {"type": "string"}
    }
  }
}
```

### Success Response

Status: `201 Created`

```json
{
  "id": "98a41c8e-3f56-49b7-a951-15c4b01440a8",
  "tenant_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
  "name": "lookup_order_status",
  "description": "Use when a visitor asks for the status of an order and provides an order ID.",
  "tool_type": "webhook",
  "endpoint_url": "https://api.example.com/nexiflow/tools/order-status",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": {"type": "string"}
    },
    "required": ["order_id"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "status": {"type": "string"},
      "estimated_delivery": {"type": "string"}
    }
  },
  "is_active": true,
  "created_at": "2026-05-03T09:45:00Z"
}
```

## Attach Tool To Draft Version

```http
POST /agents/{agent_id}/versions/{version}/tools/{tool_id}
```

Required scope: `tools:write`

### Rules

1. The agent and tool must belong to the same tenant.
2. The agent must be a single-prompt agent.
3. The version must exist.
4. The version must not be published.
5. The tool must be active.
6. The operation is idempotent.

### Success Response

Status: `200 OK`

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "version": 2,
  "tool_id": "98a41c8e-3f56-49b7-a951-15c4b01440a8",
  "attached": true,
  "message": "Tool 'lookup_order_status' attached to agent v2."
}
```

### Common Error

Published version:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot attach a tool to a published version. Only draft versions can be modified.",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "version": 2
  }
}
```

## List Tools Attached To Version

```http
GET /agents/{agent_id}/versions/{version}/tools
```

Required scope: `tools:read`

### Success Response

Status: `200 OK`

```json
[
  {
    "id": "98a41c8e-3f56-49b7-a951-15c4b01440a8",
    "tenant_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
    "name": "lookup_order_status",
    "description": "Use when a visitor asks for the status of an order and provides an order ID.",
    "tool_type": "webhook",
    "endpoint_url": "https://api.example.com/nexiflow/tools/order-status",
    "input_schema": {
      "type": "object",
      "properties": {
        "order_id": {"type": "string"}
      },
      "required": ["order_id"]
    },
    "output_schema": {},
    "is_active": true,
    "created_at": "2026-05-03T09:45:00Z"
  }
]
```

## Detach Tool From Draft Version

```http
DELETE /agents/{agent_id}/versions/{version}/tools/{tool_id}
```

Required scope: `tools:write`

### Success Response

Status: `200 OK`

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "version": 2,
  "tool_id": "98a41c8e-3f56-49b7-a951-15c4b01440a8",
  "attached": false,
  "message": "Tool 'lookup_order_status' detached from agent v2."
}
```

## Update Tool

```http
PATCH /tools/{tool_id}
```

Required scope: `tools:write`

Only mutable fields can be updated. Tool `name` is immutable.

Example:

```json
{
  "description": "Use for order status lookups after the visitor provides an order ID.",
  "endpoint_url": "https://api.example.com/nexiflow/tools/order-status-v2"
}
```

Returns the full `ToolResponse`.

## Delete Tool

```http
DELETE /tools/{tool_id}
```

Required scope: `tools:write`

Deleting a tool fails if it is still attached to any agent version.

Success status: `204 No Content`

Common error:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot delete a tool that is attached to one or more agent versions. Detach the tool from all versions first.",
  "details": {
    "tool_id": "98a41c8e-3f56-49b7-a951-15c4b01440a8",
    "attached_version_count": 1
  }
}
```

## Archive Agent

Soft-archives an agent. The row remains in the database and can be restored.

```http
DELETE /agents/{agent_id}
```

Required scope: `agents:write`

Success status: `204 No Content`

Archived agents are hidden from normal `GET /agents` unless `status=archived`.

## Restore Agent

```http
POST /agents/{agent_id}/restore
```

Required scope: `agents:write`

Returns the restored full `AgentResponse`.

Common error when the agent is not archived:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent is not archived",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b"
  }
}
```

## Minimal Curl Flow

Set variables:

```bash
export API_BASE="https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1"
export API_KEY="nxf_111111111111111111111111111111111111111111111111"
export TENANT_ID="00000000-0000-0000-0000-000000000001"
```

Create agent:

```bash
curl -sS -X POST "$API_BASE/agents" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Website Support Agent",
    "channel": "chat",
    "engine": "single_prompt",
    "response_engine": {
      "general_prompt": "You are a concise website support agent.",
      "begin_message": "Hi, how can I help?",
      "begin_message_mode": "agent_speaks_first",
      "llm_settings": {
        "model_id": "gemini-2.5-flash-lite",
        "temperature": 0.4
      },
      "chat_settings": {
        "language_code": "en-US",
        "timezone": "UTC"
      }
    }
  }'
```

Check readiness:

```bash
curl -sS "$API_BASE/agents/$AGENT_ID/draft-status" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID"
```

Publish:

```bash
curl -sS -X POST "$API_BASE/agents/$AGENT_ID/publish" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID"
```
