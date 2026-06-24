# HTTP API Contract: OpenClaw Gateway ↔ Hermes Agent

## Overview

OpenClaw gateway (TypeScript) connects to hermes-agent (Python) as a custom
model provider via the OpenAI-compatible Chat Completions API. All traffic
flows over the internal Docker network on `http://hermes-agent:8642/v1`.

```
User ──▶ Channel (Telegram/Discord/…)
           │
           ▼
     OpenClaw Gateway (:18789)
           │
           │  POST /v1/chat/completions
           │  Authorization: Bearer <HERMES_API_SERVER_KEY>
           ▼
     Hermes Agent (:8642)
           │
           ▼
     LLM Provider (OpenAI/Anthropic/…)
```

## Authentication

Every request from OpenClaw to hermes-agent carries:

```
Authorization: Bearer <HERMES_API_SERVER_KEY>
```

The key is shared via environment variables in both containers. Hermes-agent
refuses to start without a valid `API_SERVER_KEY` (min 16 chars).

## Endpoints consumed by OpenClaw

### POST /v1/chat/completions

Primary inference endpoint. OpenClaw sends conversation history and receives
either a streaming SSE response or a single JSON response.

**Request:**

```json
{
  "model": "hermes-agent",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Hello" }
  ],
  "stream": true,
  "temperature": 0.7,
  "max_tokens": 4096
}
```

**Headers:**

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | `Bearer <HERMES_API_SERVER_KEY>` |
| `Content-Type` | Yes | `application/json` |
| `X-Hermes-Session-Id` | No | Persist conversation across requests |
| `X-Hermes-Session-Key` | No | Scope long-term memory to a key |

**Response (non-streaming):**

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "hermes-agent",
  "choices": [
    {
      "index": 0,
      "message": { "role": "assistant", "content": "Hello! How can I help?" },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 10,
    "total_tokens": 30
  }
}
```

**Response (streaming):** SSE stream of `data: {...}` chunks followed by
`data: [DONE]`. Each chunk contains a `choices[0].delta` object with
incremental content.

### POST /v1/responses

OpenAI Responses API format. Stateful via `previous_response_id`.

### GET /v1/models

Returns available models. Used by OpenClaw for model discovery.

```json
{
  "object": "list",
  "data": [
    {
      "id": "hermes-agent",
      "object": "model",
      "owned_by": "hermes"
    }
  ]
}
```

### GET /health

Liveness probe used by Docker healthcheck.

```json
{ "status": "ok" }
```

### GET /health/detailed

Rich status for cross-container probing by OpenClaw dashboard.

## Endpoints NOT consumed by OpenClaw

These are hermes-agent endpoints available for direct access but not part
of the OpenClaw integration path:

| Endpoint | Description |
|----------|-------------|
| `GET /api/sessions` | List hermes sessions |
| `POST /api/sessions` | Create a session |
| `GET /api/sessions/{id}/messages` | Read session history |
| `POST /api/sessions/{id}/chat` | Chat with a persisted session |
| `POST /v1/runs` | Start an async agent run |
| `GET /v1/runs/{id}` | Check run status |
| `GET /v1/runs/{id}/events` | SSE stream of run lifecycle events |
| `POST /v1/runs/{id}/approval` | Resolve pending approval |
| `POST /v1/runs/{id}/stop` | Stop a running agent |

## OpenClaw Configuration

Register hermes-agent in `~/.openclaw/openclaw.json`:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "hermes": {
        "baseUrl": "http://hermes-agent:8642/v1",
        "apiKey": "${HERMES_API_SERVER_KEY}",
        "api": "openai-completions",
        "timeoutSeconds": 300,
        "request": { "allowPrivateNetwork": true },
        "models": [{
          "id": "hermes-agent",
          "name": "Hermes Agent (local)",
          "reasoning": false,
          "input": ["text", "image"],
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 128000,
          "maxTokens": 16384
        }]
      }
    }
  }
}
```

Key configuration points:

- `api: "openai-completions"` — tells OpenClaw to use the Chat Completions
  transport, which hermes-agent speaks natively.
- `request.allowPrivateNetwork: true` — required because hermes-agent is on
  the internal Docker network (private IP). Without this, OpenClaw blocks
  requests to private network addresses.
- `timeoutSeconds: 300` — hermes-agent runs full agent loops with tool calls,
  which can take several minutes. The default provider timeout is too short.
- `cost: 0` — hermes-agent proxies to upstream LLMs; cost tracking happens
  inside hermes-agent, not at the OpenClaw provider level.

## Error Handling

| HTTP Status | Meaning | OpenClaw behavior |
|-------------|---------|-------------------|
| 200 | Success | Process response normally |
| 401 | Bad/missing API key | Auth failure; no retry |
| 429 | Rate limited | Retry with backoff |
| 500 | Server error | Fail the request |
| 503 | Agent busy | Retry with backoff |

## Network Topology

Both services run on the `openclaw-net` bridge network. OpenClaw resolves
`hermes-agent` via Docker DNS. Port 8642 is internal-only by default;
port 18789 (OpenClaw gateway) is exposed to the host.

For direct hermes-agent API access (debugging, external clients), uncomment
the `ports` section in `deploy/docker-compose.yml`.
