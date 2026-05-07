# Website Usage API

Use these routes when embedding a single-prompt chat agent into a public website.

Important: browser code must never use tenant API keys. Browser code uses a public key only for session initialization, then uses a short-lived widget session token for messages.

For credential setup and copy-paste test values, see `docs/api_docs/chat/credentials_and_test_values.md`.

## Base URL

Render deployment:

```text
https://nexiflowai-single-prompt-agent-tool.onrender.com
```

Public widget routes are not under `/api/v1`. For example:

```text
POST https://nexiflowai-single-prompt-agent-tool.onrender.com/widget/init
POST https://nexiflowai-single-prompt-agent-tool.onrender.com/widget/sessions/{session_id}/message
```

Management setup routes still use `/api/v1`:

```text
POST https://nexiflowai-single-prompt-agent-tool.onrender.com/api/v1/public-keys
```

## Frontend Integration Checklist

1. Do not call `/api/v1/agents/{agent_id}/test-session` from public website code.
2. Do not expose `X-API-Key` or `Authorization: Bearer <member_jwt>` in browser JavaScript.
3. Store the public key `nxf_pk_...` in frontend configuration.
4. Call `/widget/init` when the chat opens.
5. Store the returned `session_id` and `session_token` for this chat session.
6. Send `Authorization: Bearer <session_token>` to widget message/history/end routes.
7. If `/widget/init` fails with an origin error, update the public key `allowed_domains`.

## Route Groups

Management route for setup:

```text
/api/v1/public-keys
```

Public website routes:

```text
/widget/init
/widget/sessions/{session_id}/message
/widget/sessions/{session_id}/history
/widget/sessions/{session_id}/end
```

## End-To-End Website Flow

1. Backend/admin creates and publishes a `channel = "chat"` single-prompt agent.
2. Backend/admin creates a public key with allowed domains and allowed agent IDs.
3. Browser calls `POST /widget/init` with the public key and agent ID.
4. API validates:
   - public key exists
   - public key is active
   - key has not expired
   - browser `Origin` is allowed
   - `agent_id` is allowed by the key
   - agent exists, belongs to the same tenant, is `channel = "chat"`, and has a published version
5. API creates a session and returns a widget session JWT.
6. Browser calls `POST /widget/sessions/{session_id}/message` with the JWT.
7. API streams assistant tokens as Server-Sent Events.
8. Browser optionally calls history after reconnect.
9. Browser or backend calls end when the chat is complete.

## Security Model

### Public Key

Public keys are safe to embed in frontend JavaScript. They can only initialize widget sessions for configured agent IDs and allowed domains.

Public key format:

```text
nxf_pk_<48 hex chars>
```

### Widget Session Token

The widget token is a JWT returned by `/widget/init`.

Use it as:

```http
Authorization: Bearer <session_token>
```

The token is scoped to one session, one tenant, and one agent. It cannot call management APIs.

### Origin Validation

The browser should send:

```http
Origin: https://www.example.com
```

Allowed domain examples:

| Allowed domain | Matches |
|---|---|
| `example.com` | `https://example.com` |
| `www.example.com` | `https://www.example.com` |
| `*.example.com` | `https://app.example.com`, not `https://example.com` |
| `localhost:*` | `http://localhost:3000`, `http://127.0.0.1:5173` |
| `example.com:3000` | `https://example.com:3000` |

## Create Public Key

This is a management route. Call it from an admin dashboard or backend, never from browser code.

```http
POST /api/v1/public-keys
```

Required scope: `admin`

### Headers

```http
X-API-Key: <tenant_api_key>
X-Tenant-ID: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
Content-Type: application/json
Accept: application/json
```

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `name` | string | yes | Friendly key name. |
| `agent_ids` | UUID array | yes | Chat agents this key can initialize. Must contain at least one ID. |
| `allowed_domains` | string array | yes | Browser origins allowed to use this key. Must contain at least one domain. |
| `rate_limit_requests` | integer | no | Default `120`. Max requests in window. |
| `rate_limit_window` | integer | no | Default `60`. Window in seconds. |
| `expires_at` | datetime or null | no | Optional UTC expiry. |

### Example Request

```json
{
  "name": "Production Website Widget",
  "agent_ids": ["2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b"],
  "allowed_domains": ["example.com", "www.example.com"],
  "rate_limit_requests": 120,
  "rate_limit_window": 60,
  "expires_at": null
}
```

### Success Response

Status: `201 Created`

The full key is returned only once. Store it in your website configuration.

```json
{
  "id": "8e39464a-1d1b-4d9f-bf5b-91a7214d7ed2",
  "key": "nxf_pk_6fc2f7ef0bcb03e59307f278f2f51fb57b44fedc4a1b9ed2",
  "key_prefix": "nxf_pk_6fc2",
  "name": "Production Website Widget",
  "agent_ids": ["2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b"],
  "allowed_domains": ["example.com", "www.example.com"],
  "rate_limit_requests": 120,
  "rate_limit_window": 60,
  "created_at": "2026-05-03T09:50:00Z",
  "expires_at": null
}
```

### Common Errors

Missing admin scope:

Status: `403 Forbidden`

```json
{
  "detail": "API key does not grant required scope: admin"
}
```

Bad request body:

Status: `422 Unprocessable Entity`

```json
{
  "detail": [
    {
      "type": "too_short",
      "loc": ["body", "allowed_domains"],
      "msg": "List should have at least 1 item"
    }
  ]
}
```

## Initialize Widget Session

This route is called by the website when the visitor opens the chat widget.

```http
POST /widget/init
```

Authentication: public key in JSON body.

### Headers

```http
Origin: https://www.example.com
Content-Type: application/json
Accept: application/json
```

`Origin` matters. If it is missing, the service falls back to `Referer`, but a real browser integration should rely on `Origin`.

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `public_key` | string | yes | The `nxf_pk_...` key created by management API. |
| `agent_id` | UUID | yes | Must be in the public key `agent_ids`. |
| `visitor_id` | string or null | no | Your anonymous visitor/user ID. Max 255 chars. |
| `metadata` | object | no | Any JSON metadata you want attached to the session. |

### Example Request

Full browser-style request:

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

JSON body only:

```json
{
  "public_key": "nxf_pk_6fc2f7ef0bcb03e59307f278f2f51fb57b44fedc4a1b9ed2",
  "agent_id": "ea517389-f9d8-448a-8080-34a6225509fa",
  "visitor_id": "visitor-123",
  "metadata": {
    "page": "/pricing",
    "utm_source": "google",
    "locale": "en-US"
  }
}
```

### Success Response

Status: `201 Created`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "session_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "agent_name": "Website Support Agent",
  "greeting": "Hi there, how can I help today?"
}
```

Field meanings:

| Field | Notes |
|---|---|
| `session_id` | Store in widget state. Used in later widget route paths. |
| `session_token` | Store in memory or session storage. Send as Bearer token on later widget calls. |
| `agent_name` | Display name for the widget header. |
| `greeting` | First assistant message. Render immediately when non-null. |

If `greeting` is `null`, do not render an assistant opener. Wait for the visitor's first message.

### Common Errors

Invalid public key:

Status: `401 Unauthorized`

```json
{
  "error": "AUTHENTICATION_ERROR",
  "message": "Invalid public key",
  "details": {}
}
```

Origin not allowed:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Origin domain is not allowed for this public key",
  "details": {
    "origin": "https://not-allowed.example"
  }
}
```

Agent not allowed for this key:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent is not authorized for this public key",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b"
  }
}
```

Agent is not a chat agent:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Widget sessions require a published chat agent",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b",
    "channel": "voice"
  }
}
```

Agent has no published version:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Agent has no published version",
  "details": {
    "agent_id": "2c6a4b53-3c1f-4ef7-8f98-7e3192e29c0b"
  }
}
```

## Send Message And Stream Response

This route sends one visitor message and streams the assistant response.

```http
POST /widget/sessions/{session_id}/message
```

Authentication: widget session token.

### Headers

```http
Authorization: Bearer <session_token>
Content-Type: application/json
Accept: text/event-stream
```

### Path Parameters

| Name | Type | Required | Notes |
|---|---|---:|---|
| `session_id` | UUID | yes | Must match the `session_id` claim inside the widget token. |

### Request Body

| Field | Type | Required | Notes |
|---|---|---:|---|
| `message` | string | yes | Visitor message. Min 1 char, max 10000 chars. |

### Example Request

Full request:

```bash
curl -N -X POST \
  "https://nexiflowai-single-prompt-agent-tool.onrender.com/widget/sessions/65c86045-32b4-4d9a-a4df-0fd79683bb74/message" \
  -H "Authorization: Bearer <session_token_from_widget_init>" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"message":"Can you explain the Growth plan?"}'
```

JSON body only:

```json
{
  "message": "Can you explain the Growth plan?"
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
data: {"text":"The Growth plan is built for teams that need higher usage limits"}

event: message.delta
data: {"text":" and production support. "}

event: message.delta
data: {"text":"Do you want pricing or setup details?"}

event: message.completed
data: {"session_id":"65c86045-32b4-4d9a-a4df-0fd79683bb74"}

event: done
data: [DONE]
```

### SSE Events

| Event | Data shape | Meaning |
|---|---|---|
| `message.delta` | `{"text": "..."}` | A chunk of assistant text. Append it to the current assistant message. |
| `tool.started` | `{"tool": "...", "tool_call_id": "...", "tool_mode": "..."}` | Agent started a tool call. Optional UI event. |
| `tool.completed` | `{"tool": "...", "response": {...}}` | Tool completed. Optional UI event. |
| `message.completed` | `{"session_id": "..."}` | Assistant turn is complete. |
| `interrupt` | `{}` | Stream was interrupted. |
| `error` | `{"detail": "..."}` | The stream failed. |
| `done` | `[DONE]` | Terminal stream marker. Close the stream. |

### Frontend Parsing Example

```ts
async function sendWidgetMessage(apiBase: string, sessionId: string, token: string, message: string) {
  const response = await fetch(`${apiBase}/widget/sessions/${sessionId}/message`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${token}`,
      "Content-Type": "application/json",
      "Accept": "text/event-stream"
    },
    body: JSON.stringify({ message })
  });

  if (!response.ok || !response.body) {
    throw new Error(`Widget message failed: ${response.status}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";
  let assistantText = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const events = buffer.split("\n\n");
    buffer = events.pop() ?? "";

    for (const rawEvent of events) {
      const eventLine = rawEvent.split("\n").find((line) => line.startsWith("event: "));
      const dataLine = rawEvent.split("\n").find((line) => line.startsWith("data: "));
      const eventName = eventLine?.slice("event: ".length);
      const dataText = dataLine?.slice("data: ".length);

      if (eventName === "message.delta" && dataText) {
        const payload = JSON.parse(dataText) as { text: string };
        assistantText += payload.text;
      }

      if (eventName === "error" && dataText) {
        throw new Error(JSON.parse(dataText).detail);
      }

      if (eventName === "done") {
        return assistantText;
      }
    }
  }

  return assistantText;
}
```

### Common Errors

Wrong or expired widget token:

Status: `401 Unauthorized`

```json
{
  "error": "AUTHENTICATION_ERROR",
  "message": "Invalid token",
  "details": {}
}
```

Session ID does not match token:

Status: `404 Not Found`

```json
{
  "error": "RESOURCE_NOT_FOUND",
  "message": "Session not found: 65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "details": {}
}
```

Terminal session:

Status: `400 Bad Request`

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Cannot send a message to a terminal session",
  "details": {
    "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
    "status": "completed"
  }
}
```

LLM/tool failure inside stream:

Status may still be `200 OK`, then stream emits:

```text
event: error
data: {"detail":"The tool endpoint could not be reached."}
```

## Load Widget History

Use this after page refresh or reconnection.

```http
GET /widget/sessions/{session_id}/history
```

Authentication: widget session token.

### Headers

```http
Authorization: Bearer <session_token>
Accept: application/json
```

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "messages": [
    {
      "role": "assistant",
      "content": "Hi there, how can I help today?",
      "timestamp": "2026-05-03T09:55:00Z"
    },
    {
      "role": "user",
      "content": "Can you explain the Growth plan?",
      "timestamp": "2026-05-03T09:55:10Z"
    },
    {
      "role": "assistant",
      "content": "The Growth plan is built for teams that need higher usage limits.",
      "timestamp": "2026-05-03T09:55:12Z"
    }
  ],
  "is_active": true
}
```

Only `user` and `assistant` transcript rows are returned.

## End Widget Session

Call this when the user closes the chat or when your frontend knows the conversation is done.

```http
POST /widget/sessions/{session_id}/end
```

Authentication: widget session token.

### Headers

```http
Authorization: Bearer <session_token>
Accept: application/json
```

### Request Body

No body is required.

### Success Response

Status: `200 OK`

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "completed",
  "already_terminal": false
}
```

If the session was already terminal:

```json
{
  "session_id": "65c86045-32b4-4d9a-a4df-0fd79683bb74",
  "status": "completed",
  "already_terminal": true
}
```

Ending a widget session can trigger:

1. final transcript/event persistence
2. terminal webhooks such as `chat_ended`
3. post-chat analysis when configured

## Minimal Browser Flow

```ts
const WIDGET_BASE = "https://nexiflowai-single-prompt-agent-tool.onrender.com/widget";
const PUBLIC_KEY = "nxf_pk_6fc2f7ef0bcb03e59307f278f2f51fb57b44fedc4a1b9ed2";
const AGENT_ID = "ea517389-f9d8-448a-8080-34a6225509fa";

async function initChat() {
  const response = await fetch(`${WIDGET_BASE}/init`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Accept": "application/json"
    },
    body: JSON.stringify({
      public_key: PUBLIC_KEY,
      agent_id: AGENT_ID,
      visitor_id: crypto.randomUUID(),
      metadata: {
        page: window.location.pathname
      }
    })
  });

  if (!response.ok) {
    throw new Error(`Widget init failed: ${response.status}`);
  }

  return response.json() as Promise<{
    session_id: string;
    session_token: string;
    agent_name: string;
    greeting: string | null;
  }>;
}
```

## Server-To-Server Alternative

If another backend service uses the chat agent, do not use `/widget/*`. Use tenant-authenticated session routes:

```text
POST /api/v1/sessions
POST /api/v1/sessions/{session_id}/stream
POST /api/v1/sessions/{session_id}/end
```

See `docs/api_docs/chat/operations_analysis_transcripts.md`.

## Operational Notes

1. Keep the public key in frontend environment configuration, not in a private backend secret store. It is intentionally public.
2. Keep tenant API keys on the backend only.
3. A widget token should be treated like a session credential. Store it only for the life of the chat session.
4. Handle both HTTP errors and SSE `error` events.
5. Always close the SSE reader after `done`.
6. Use `/history` after reload to rebuild UI state.
7. Call `/end` when the visitor closes the chat if you want analysis and terminal webhooks promptly.
