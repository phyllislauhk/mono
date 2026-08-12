# Data Model: Agent Chat App (v1)

**Date**: 2026-08-12  
**Storage**: In-memory only (browser session + request-scoped backend). No database (FR-007).

## Entity: Message

Represents a single utterance in the chat thread.

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `id` | string (UUID) | yes | Unique within thread |
| `role` | enum | yes | `user` \| `assistant` |
| `content` | string | yes | Non-empty after trim; max 4,000 chars when `role=user` |
| `created_at` | ISO 8601 datetime | yes | Set at creation |
| `status` | enum | no | `complete` \| `streaming` \| `error` (assistant only) |

**Validation rules**:
- User messages MUST NOT be blank or whitespace-only (edge case).
- User messages exceeding 4,000 characters MUST be rejected with Traditional Chinese error before send.
- Assistant messages in `streaming` state MAY have partial `content` until stream completes.

**Lifecycle**:
```
[user types] → [validate] → [append to thread] → [send to backend]
[assistant] → streaming (partial content updates) → complete | error
[page refresh] → all messages discarded
```

## Entity: ChatThread

Single continuous conversation (FR-003). v1 allows exactly one thread per browser session.

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `id` | string | yes | Generated client-side; stable for session |
| `messages` | Message[] | yes | Ordered chronologically |
| `status` | enum | yes | `idle` \| `streaming` \| `error` |

**State transitions**:

```text
idle ──(user sends)──► streaming ──(stream done)──► idle
streaming ──(error)──► error ──(user retries)──► streaming
idle/streaming ──(page refresh)──► [thread destroyed]
```

**Rules**:
- While `status=streaming`, composer input and send MUST be disabled (FR-012).
- On each send, full `messages` array serialized into AG-UI `RunAgentInput.messages` (FR-003a).

## Entity: HealthStatus

Response from `GET /health` (custom endpoint, schema version `1.0.0`).

| Field | Type | Required | Values |
|-------|------|----------|--------|
| `schema_version` | string | yes | `1.0.0` |
| `status` | enum | yes | `healthy` \| `degraded` \| `unhealthy` |
| `agent_mode` | enum | yes | `mock` \| `real` |
| `upstream_reachable` | boolean \| null | yes | `true`/`false` in real mode; `null` in mock mode |
| `request_id` | string | yes | UUID for correlation |
| `timestamp` | ISO 8601 | yes | Response time |

**Status derivation**:

| agent_mode | upstream_reachable | status |
|------------|-------------------|--------|
| mock | null | healthy |
| real | true | healthy |
| real | false | degraded |
| (process down) | — | unhealthy (HTTP 503) |

## Entity: AgentConfiguration

Runtime configuration (environment variables, not persisted).

| Variable | Required | Default | Purpose |
|----------|----------|---------|---------|
| `OPENAI_API_KEY` | no | — | When set, enables `real` mode |
| `AGENT_OS_PORT` | no | `7777` | Backend listen port |
| `AGENT_OS_HOST` | no | `localhost` | Backend bind host |
| `NEXT_PUBLIC_AGUI_AGENT_URL` | no | `http://localhost:7777` | Frontend → backend base URL |

## Relationships

```text
ChatThread 1──* Message
HealthStatus ── reflects ── AgentConfiguration (runtime)
```

## AG-UI Boundary Mapping

Frontend `Message` ↔ AG-UI protocol message objects at `POST /agui` boundary only. No shared types across frontend/backend codebases (Constitution Principle IV).
