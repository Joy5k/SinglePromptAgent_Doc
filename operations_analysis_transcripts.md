# Chat Operations, Analysis, And Transcripts API

This document covers the runtime APIs used after a chat agent exists: creating sessions, streaming messages, ending sessions, reading transcripts, reading events, retrying analysis, and reading operational analytics.

For agent creation and publishing, see `docs/api_docs/chat/agent_creation_and_maintenance.md`.

For public browser widgets, see `docs/api_docs/chat/website_usage.md`.

## Base URL

Local development:

```text
http://localhost:8001/api/v1
```

All paths in this document are relative to `/api/v1`.

## Authentication

Server-to-server session routes require tenant authentication.

```http
X-API-Key: <tenant_api_key>
X-Tenant-ID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Content-Type: application/json
Accept: application/json
```

or:

```http
Authorization: Bearer <member_jwt>
X-Tenant-ID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
```

## Required API Key Scopes

| Scope | Routes |
|---|---|
| `sessions:write` | create session, stream, step, interrupt, end, retry analysis |
| `sessions:read` | get session, transcripts, events, analysis, agent session history, analytics |

## Runtime Guarantees

Session routes check three things before returning or mutating data:

1. The session exists.
2. The session belongs to the authenticated tenant.
3. The session's agent resolves through the single-prompt agent repository.

Because the database is shared with the flow-agent service, this prevents the chat service from serving flow-engine sessions.

## Standard Error Format

Application errors:

```json
{
  "error": "RESOURCE_NOT_FOUND",
  "message": "Session not found: 65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "details": {}
}
```

FastAPI validation errors:

```json
{
  "detail": [
    {
      "type": "uuid_parsing",
      "loc": ["path", "session_id"],
      "msg": "Input should be a valid UUID"
    }
  ]
}
```

Common status codes:

| Status | Meaning | Common cause |
|---:|---|---|
| `400` | Validation error | terminal session, missing transcript for analysis retry, unpublished agent session creation |
| `401` | Authentication error | missing or invalid API key/JWT |
| `403` | Forbidden | API key lacks required scope |
| `404` | Not found | wrong tenant, missing session, flow-engine session, missing analysis |
| `422` | Schema error | invalid UUID or request body type |
| `429` | Rate or budget limit | rate limiter or LLM budget guard |
| `500` | Server error | queue unavailable, database failure, unexpected exception |

## Create Production Session

Creates a production session against a published agent version.

```http
POST /sessions
```

Required scope: `sessions:write`

### Requirements

1. Agent must exist for this tenant.
2. Agent must be `engine = "single_prompt"`.
3. Version must exist.
4. Version must be published.
5. Agent config must have been valid when published.

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `agent_id` | UUID | yes | Published chat agent ID. |
| `version` | integer | yes | Published version number, min `1`. |
| `initial_variables` | object | no | Values available to prompt templates and tools. |
| `metadata` | object | no | Custom metadata such as CRM IDs, source, page, or user ID. |

### Example Request

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "version": 2,
  "initial_variables": {
    "customer_name": "Ava",
    "plan_name": "Growth"
  },
  "metadata": {
    "source": "backend",
    "ticket_id": "T-1001",
    "user_id": "user_789"
  }
}
```

### Success Response

Status: `201 Created`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "current_node_id": "prompt_mode",
  "begin_message": "Hi Ava, how can I help today?"
}
```

Field meanings:

| Field | Notes |
|---|---|
| `session_id` | Use this in stream, state, transcript, event, analysis, and end routes. |
| `current_node_id` | Always `prompt_mode` for single-prompt runtime. Kept for shared schema compatibility. |
| `begin_message` | First assistant message when configured. Can be `null`. |

### Common Errors

Unpublished version:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot create a session on an unpublished version",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "version": 2
  }
}
```

Wrong tenant or flow-agent ID:

Status: `404 Not Found`

```json
{
  "error": "RESOURCE_NOT_FOUND",
  "message": "Agent not found: 2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "details": {}
}
```

## Stream A Chat Message

Sends one user message and streams the assistant response with Server-Sent Events.

```http
POST /sessions/{session_id}/stream
```

Required scope: `sessions:write`

Use this for interactive chat. This is the recommended runtime endpoint.

### Headers

```http
X-API-Key: <tenant_api_key>
X-Tenant-ID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Content-Type: application/json
Accept: text/event-stream
```

### Path Parameters

| Name | Type | Required | Notes |
|---|---|---:|---|
| `session_id` | UUID | yes | Active session belonging to this tenant and service. |

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `user_input` | string or null | recommended | Message to send. Runtime rejects empty prompt turns. |

### Example Request

```json
{
  "user_input": "Can you explain the Growth plan?"
}
```

### Success Response

Status: `200 OK`

Content type:

```http
text/event-stream
```

Example stream:

```text
event: message.delta
data: {"text":"The Growth plan is designed for teams that need higher usage limits"}

event: message.delta
data: {"text":" and priority support."}

event: message.completed
data: {"session_id":"65c86045-32b4-4d9a-a4df-0fd79683bb74"}

event: done
data: [DONE]
```

### Tool Events

If the agent version has attached tools, the stream may include tool events.

```text
event: tool.started
data: {"tool":"lookup_order_status","tool_call_id":"call_01","tool_mode":"native"}

event: tool.completed
data: {"tool":"lookup_order_status","tool_call_id":"call_01","tool_mode":"native","response":{"status":"processing"}}
```

### SSE Event Contract

| Event | Data | Client behavior |
|---|---|---|
| `message.delta` | `{"text": "..."}` | Append `text` to the active assistant message. |
| `tool.started` | tool metadata | Optional: show "checking..." state. |
| `tool.completed` | tool metadata and response | Optional: hide tool loading state. |
| `message.completed` | `{"session_id": "..."}` | Mark assistant turn complete. |
| `interrupt` | `{}` | Stop rendering current assistant message. |
| `error` | `{"detail": "..."}` | Show recoverable failure UI. |
| `done` | `[DONE]` | Close stream reader. |

### Persistence Behavior

After the final prompt result:

1. User transcript is appended.
2. Assistant transcript is appended.
3. Execution event is written.
4. Tool events are written when tools are used.
5. Extracted variables are saved.
6. Step webhooks are emitted when configured.

If the client disconnects before completion, the route records an SSE disconnect metric and interrupts the active stream.

### Common Errors

Terminal session:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot stream a step for a terminal session",
  "details": {
    "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
    "status": "completed"
  }
}
```

Empty user input may be returned inside stream:

```text
event: error
data: {"detail":"user_input is required for a prompt turn"}
```

LLM budget exceeded:

Status: `429 Too Many Requests`

```json
{
  "error": "BUDGET_EXCEEDED",
  "message": "LLM budget exceeded",
  "details": {}
}
```

## Queue A Non-Streaming Step

Enqueues one message to ARQ and returns immediately.

```http
POST /sessions/{session_id}/step
```

Required scope: `sessions:write`

Use this only for backend workflows where streaming is not needed. The ARQ worker must be running.

### Request Body

```json
{
  "user_input": "Please summarize my options."
}
```

### Success Response

Status: `202 Accepted`

```json
{
  "job_id": "worker_execute_step:1",
  "status": "queued"
}
```

### Common Error

Queue is not initialized:

Status: `500 Internal Server Error`

```json
{
  "error": "INTERNAL_SERVER_ERROR",
  "message": "An unexpected server condition was encountered.",
  "details": "..."
}
```

## Poll Step Job Status

```http
GET /sessions/{session_id}/step-status/{job_id}
```

Required scope: `sessions:read`

### Success Response

Status: `200 OK`

Queued or running:

```json
{
  "job_id": "worker_execute_step:1",
  "status": "queued",
  "result": null,
  "error": null
}
```

Complete:

```json
{
  "job_id": "worker_execute_step:1",
  "status": "complete",
  "result": {
    "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
    "assistant_response": "Here are your options...",
    "status": "active"
  },
  "error": null
}
```

Failed:

```json
{
  "job_id": "worker_execute_step:1",
  "status": "failed",
  "result": null,
  "error": "user_input is required for a prompt turn"
}
```

## Interrupt Active Stream

Cancels an active streaming generation for the session.

```http
POST /sessions/{session_id}/interrupt
```

Required scope: `sessions:write`

### Request Body

No body is required.

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "interrupted": true
}
```

If no stream was active:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "interrupted": false
}
```

## End Session

Marks a session terminal and runs terminal workflow.

```http
POST /sessions/{session_id}/end
```

Required scope: `sessions:write`

### Request Body

| Field | Type | Required | Default | Notes |
|---|---|---:|---|---|
| `reason` | string | no | `user_ended` | Max 255 chars. |
| `ended_by` | string | no | `api` | Max 64 chars. Examples: `api`, `widget`, `system`. |

### Example Request

```json
{
  "reason": "resolved",
  "ended_by": "api"
}
```

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "completed",
  "reason": "resolved",
  "ended_by": "api",
  "already_terminal": false,
  "flush_job_id": "worker_persist_session_to_db:2",
  "analysis_job_id": "worker_run_pca:3"
}
```

If ARQ is unavailable, job IDs may be `null`, and cached data is cleared after direct persistence.

If the session is already terminal:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "completed",
  "reason": "user_ended",
  "ended_by": "api",
  "already_terminal": true,
  "flush_job_id": null,
  "analysis_job_id": null
}
```

Terminal workflow:

1. Merge cached transcripts, variables, and events.
2. Mark session `completed`.
3. Close call metadata.
4. Emit terminal webhook such as `chat_ended`.
5. Queue durable cache flush.
6. Queue post-chat analysis when enabled and not a test session.

## Get Session State

```http
GET /sessions/{session_id}
```

Required scope: `sessions:read`

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "agent_version": 2,
  "current_node_id": "prompt_mode",
  "state": {
    "customer_name": "Ava",
    "plan_name": "Growth",
    "_analysis_status": "pending",
    "_analysis_job_id": "worker_run_pca:3"
  },
  "status": "active"
}
```

Status values:

| Status | Meaning |
|---|---|
| `active` | Session can receive messages. |
| `completed` | Session ended normally. |
| `failed` | Session ended due to failure. |
| `timed_out` | Session ended due to inactivity timeout. |

## Get Transcripts

Returns user and assistant messages for a session. Tool metadata can appear on transcript rows when present.

```http
GET /sessions/{session_id}/transcripts
```

Required scope: `sessions:read`

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "message_count": 3,
  "transcripts": [
    {
      "role": "assistant",
      "content": "Hi Ava, how can I help today?",
      "timestamp": "2026-05-03T10:00:00Z",
      "tool_name": null,
      "tool_input": null,
      "tool_output": null
    },
    {
      "role": "user",
      "content": "Can you explain the Growth plan?",
      "timestamp": "2026-05-03T10:00:12Z",
      "tool_name": null,
      "tool_input": null,
      "tool_output": null
    },
    {
      "role": "assistant",
      "content": "The Growth plan is designed for teams that need higher usage limits.",
      "timestamp": "2026-05-03T10:00:14Z",
      "tool_name": null,
      "tool_input": null,
      "tool_output": null
    }
  ]
}
```

Implementation note: the route reads Redis cache first, then falls back to the session row transcript snapshot.

## Get Execution Events

Returns low-level events for debugging, analytics, and audit-style replay.

```http
GET /sessions/{session_id}/events
```

Required scope: `sessions:read`

### Success Response

Status: `200 OK`

```json
[
  {
    "id": "34e31144-e01c-40da-b02d-97e2a8795801",
    "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
    "node_id": "prompt_mode",
    "input_data": {
      "user_message": "Can you explain the Growth plan?",
      "stream": true,
      "model_id": "gemini-2.5-flash-lite"
    },
    "output_data": {
      "assistant_response": "The Growth plan is designed for teams that need higher usage limits.",
      "tool_mode": "none"
    },
    "transition_selected": null,
    "latency_ms": 842,
    "tool_called": null,
    "latency_llm_ms": 842,
    "tokens_input": 520,
    "tokens_output": 42,
    "created_at": "2026-05-03T10:00:14Z"
  }
]
```

Tool event example:

```json
{
  "id": "e0221cbb-7494-4d7d-817e-4903eeebac2f",
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "node_id": "prompt_mode",
  "input_data": {
    "tool_call_id": "call_01",
    "tool_arguments": {
      "order_id": "ORD-1001"
    },
    "tool_mode": "native",
    "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "tool_id": "98a41c8e-3f56-49b7-a951-15c4b01440a8",
    "attempt": 1
  },
  "output_data": {
    "tool_call_id": "call_01",
    "tool_response": {
      "status": "processing"
    },
    "tool_error": null,
    "tool_mode": "native",
    "status": "success",
    "error_type": null
  },
  "transition_selected": null,
  "latency_ms": null,
  "tool_called": "lookup_order_status",
  "latency_llm_ms": null,
  "tokens_input": null,
  "tokens_output": null,
  "created_at": "2026-05-03T10:01:05Z"
}
```

## Get Post-Chat Analysis

Returns extracted analysis for a completed session.

```http
GET /sessions/{session_id}/analysis
```

Required scope: `sessions:read`

### Success Response

Status: `200 OK`

Successful analysis:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "success",
  "extracted_data_json": {
    "lead_intent": "pricing",
    "needs_follow_up": true
  },
  "error": null,
  "created_at": "2026-05-03T10:05:00Z"
}
```

Pending analysis:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "pending",
  "extracted_data_json": null,
  "error": null,
  "created_at": "2026-05-03T10:04:30Z"
}
```

Disabled analysis:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "disabled",
  "extracted_data_json": null,
  "error": null,
  "created_at": "2026-05-03T10:04:30Z"
}
```

### Common Error

No analysis record or status:

Status: `404 Not Found`

```json
{
  "error": "RESOURCE_NOT_FOUND",
  "message": "SessionAnalysis not found: 65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "details": {}
}
```

## Retry Post-Chat Analysis

Queues a new analysis job using the existing transcript.

```http
POST /sessions/{session_id}/analysis/retry
```

Required scope: `sessions:write`

### Request Body

No body is required.

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "queued",
  "job_id": "worker_retry_pca:4"
}
```

### Common Errors

No transcript content:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot retry analysis without transcript content",
  "details": {
    "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74"
  }
}
```

ARQ queue unavailable:

Status: `500 Internal Server Error`

```json
{
  "error": "INTERNAL_SERVER_ERROR",
  "message": "An unexpected server condition was encountered.",
  "details": "..."
}
```

## List Sessions For Agent

Returns paginated session history for one agent.

```http
GET /agents/{agent_id}/sessions?limit=50&offset=0
```

Required scope: `sessions:read`

### Query Parameters

| Name | Type | Required | Default | Notes |
|---|---|---:|---:|---|
| `limit` | integer | no | `50` | Min `1`, max `200`. |
| `offset` | integer | no | `0` | Pagination offset. |

### Success Response

Status: `200 OK`

```json
{
  "items": [
    {
      "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
      "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
      "agent_version": 2,
      "current_node_id": "prompt_mode",
      "status": "completed",
      "message_count": 3,
      "last_message_preview": "The Growth plan is designed for teams that need higher usage limits.",
      "created_at": "2026-05-03T10:00:00Z",
      "updated_at": "2026-05-03T10:05:00Z"
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

## Agent Analytics

Returns operational metrics for one agent.

```http
GET /agents/{agent_id}/analytics
```

Required scope: `sessions:read`

### Success Response

Status: `200 OK`

```json
{
  "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
  "session_count": 120,
  "completed_sessions": 100,
  "failed_sessions": 5,
  "timed_out_sessions": 10,
  "active_sessions": 5,
  "average_latency_ms": 860.5,
  "tool_call_count": 45,
  "tool_failure_rate": 0.04,
  "analysis_success_count": 80,
  "analysis_failure_count": 3,
  "analysis_pending_count": 2,
  "total_tokens_input": 250000,
  "total_tokens_output": 64000,
  "total_tokens": 314000,
  "total_cost_usd": 3.42
}
```

Use this for dashboards. For billing details, use the spend API.

## System Registries

These read-only endpoints help your UI build valid agent configuration forms.

### List LLM Models

```http
GET /system/llm-models
```

Authentication: none in the current FastAPI router. These are read-only configuration endpoints. If your deployment sits behind an API gateway, protect them according to your environment policy.

Returns LiteLLM aliases configured for this single-prompt service. These are valid values for `response_engine.llm_settings.model_id`.

Example response:

```json
[
  {
    "id": "72d12569-8e08-5b63-a9d6-981ad79c9631",
    "name": "Gemini 2.5 Flash Lite",
    "api_code": "gemini-2.5-flash-lite",
    "provider": "google",
    "litellm_model_id": "gemini/gemini-2.5-flash-lite",
    "cost_per_input_token": 0.0,
    "cost_per_output_token": 0.0,
    "is_active": true,
    "description": "LiteLLM route: gemini/gemini-2.5-flash-lite",
    "supports_streaming": true,
    "supports_tools": true,
    "recommended_max_output_tokens": 2048
  }
]
```

### List Languages

```http
GET /system/languages
```

Example response:

```json
[
  {
    "id": "bb6bdf0a-b4c2-4a32-909f-fb336d62c777",
    "code": "en-US",
    "name": "English (US)",
    "is_active": true
  }
]
```

### List Voices

```http
GET /system/voices?provider=11labs&language_code=en-US
```

Voice data remains available for future voice work. Chat agents do not need `voice_id`.

Example response:

```json
[
  {
    "id": "a6dbd1f2-6e03-48bb-9025-f5159f5700f9",
    "provider": "11labs",
    "voice_id": "pFZP5JQG7iQjI",
    "name": "Lily",
    "language_code": "en-US",
    "preview_audio_url": "https://example.com/voices/lily.mp3",
    "is_active": true
  }
]
```

## Minimal Curl Flow

Set variables:

```bash
export API_BASE="http://localhost:8001/api/v1"
export API_KEY="nxf_sk_example"
export TENANT_ID="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
export AGENT_ID="2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b"
```

Create session:

```bash
curl -sS -X POST "$API_BASE/sessions" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d "{
    \"agent_id\": \"$AGENT_ID\",
    \"version\": 2,
    \"initial_variables\": {\"customer_name\": \"Ava\"},
    \"metadata\": {\"source\": \"backend\"}
  }"
```

Stream message:

```bash
curl -N -X POST "$API_BASE/sessions/$SESSION_ID/stream" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"user_input":"Can you explain the Growth plan?"}'
```

End session:

```bash
curl -sS -X POST "$API_BASE/sessions/$SESSION_ID/end" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{"reason":"resolved","ended_by":"api"}'
```

Read transcript:

```bash
curl -sS "$API_BASE/sessions/$SESSION_ID/transcripts" \
  -H "X-API-Key: $API_KEY" \
  -H "X-Tenant-ID: $TENANT_ID"
```
