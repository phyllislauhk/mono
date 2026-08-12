# API Contracts: Agent Chat App (v1)

**Schema version**: 1.0.0  
**Date**: 2026-08-12

## Overview

| Contract | Method | Path | Protocol | Owner |
|----------|--------|------|----------|-------|
| AG-UI Chat | POST | `/agui` | AG-UI SSE | Agno `AGUI` interface |
| AG-UI Status | GET | `/status` | JSON | Agno `AGUI` interface |
| Extended Health | GET | `/health` | JSON (OpenAPI) | Custom route on AgentOS app |

Frontend connects via `HttpAgent` from `@ag-ui/client` to `{NEXT_PUBLIC_AGUI_AGENT_URL}/agui`.

## AG-UI Chat (`POST /agui`)

**Specification**: [AG-UI Protocol](https://github.com/ag-ui-protocol/ag-ui) — `RunAgentInput` request, SSE event stream response.

**Request** (`Content-Type: application/json`):

```json
{
  "threadId": "string",
  "runId": "string",
  "messages": [
    {
      "role": "user",
      "content": "你好，請介紹你自己"
    }
  ],
  "state": {},
  "tools": [],
  "context": [],
  "forwardedProps": {}
}
```

**Response** (`Content-Type: text/event-stream`):

Stream of AG-UI events. Minimum required for v1:

| Event | Purpose |
|-------|---------|
| `RUN_STARTED` | Run begins |
| `TEXT_MESSAGE_START` | Assistant message begins |
| `TEXT_MESSAGE_CONTENT` | Streaming text delta |
| `TEXT_MESSAGE_END` | Assistant message complete |
| `RUN_FINISHED` | Run complete |

**v1 exclusions**: No tool call events (FR-008).

**Error handling**:
- Malformed input → HTTP 422
- Agent failure mid-stream → `RUN_ERROR` event + HTTP 200 (stream) or HTTP 500 (pre-stream)
- Frontend MUST display Traditional Chinese error and allow retry (FR-010)

## AG-UI Status (`GET /status`)

**Provided by**: Agno `AGUI` interface (built-in).

**Response** (example):

```json
{
  "status": "available"
}
```

Used for basic liveness. Extended diagnostics use `/health`.

## Extended Health (`GET /health`)

See [health.openapi.yaml](./health.openapi.yaml) for full OpenAPI 3.1 schema.

**Response 200** (healthy mock mode):

```json
{
  "schema_version": "1.0.0",
  "status": "healthy",
  "agent_mode": "mock",
  "upstream_reachable": null,
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-08-12T07:33:00Z"
}
```

**Response 200** (degraded real mode, upstream down):

```json
{
  "schema_version": "1.0.0",
  "status": "degraded",
  "agent_mode": "real",
  "upstream_reachable": false,
  "request_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2026-08-12T07:33:00Z"
}
```

**Response 503** (unhealthy):

```json
{
  "schema_version": "1.0.0",
  "status": "unhealthy",
  "agent_mode": "mock",
  "upstream_reachable": null,
  "request_id": "550e8400-e29b-41d4-a716-446655440002",
  "timestamp": "2026-08-12T07:33:00Z"
}
```

## CORS

AgentOS MUST include frontend origin in `cors_allowed_origins`:

- `http://localhost:3000` (Next.js dev default)

## Versioning Policy

- `schema_version` on `/health` follows semver; breaking changes increment MAJOR.
- AG-UI protocol version pinned by `ag-ui-protocol` package version in backend and `@ag-ui/client` in frontend.
