# Research: Agent Chat App (v1)

**Date**: 2026-08-12  
**Feature**: `specs/001-agent-chat-app`

## R1: Frontend framework and chat UI library

**Decision**: React (Next.js App Router) + [assistant-ui](https://www.assistant-ui.com/) with `@assistant-ui/react-ag-ui` runtime.

**Rationale**:
- User explicitly requested assistant-ui.
- assistant-ui provides production-ready chat primitives (`Thread`, streaming state, input blocking).
- `@assistant-ui/react-ag-ui` + `HttpAgent` from `@ag-ui/client` natively consume AG-UI SSE events (`TEXT_MESSAGE_CONTENT`, etc.).

**Alternatives considered**:
- Raw React + custom SSE parser — rejected; duplicates assistant-ui streaming logic.
- CopilotKit UI — rejected; user specified assistant-ui.
- Agno Dojo frontend — rejected; user specified assistant-ui.

## R2: Backend framework and agent runtime

**Decision**: Python [Agno SDK](https://docs.agno.com/) with **AgentOS** enabled and **AGUI** interface mounted.

**Rationale**:
- User explicitly requested Agno SDK with AgentOS.
- Agno provides first-class `AGUI` interface (`POST /agui`, `GET /status`) compatible with assistant-ui's `HttpAgent`.
- AgentOS supplies FastAPI app, CORS, streaming runs, and OpenAPI docs without extra services.

**Alternatives considered**:
- Custom FastAPI + manual AG-UI encoder — rejected; Agno ships maintained `agui` interface.
- AgentOS native `/agents/{id}/runs` only — rejected; assistant-ui expects AG-UI protocol, not raw AgentOS run SSE format.

## R3: Frontend ↔ backend protocol

**Decision**: AG-UI protocol over HTTP Server-Sent Events (SSE).

**Rationale**:
- assistant-ui `useAgUiRuntime` requires AG-UI-compliant backend.
- Agno `AGUI(agent=...)` mounts `POST /agui` streaming `RunAgentInput` → AG-UI events.
- Single protocol boundary with versioned event types from `ag-ui-protocol`.

**Wire contract**:
- Chat: `POST {AGUI_BASE_URL}/agui` with `Accept: text/event-stream`
- Health (AG-UI built-in): `GET {AGUI_BASE_URL}/status`
- Health (extended): `GET {AGUI_BASE_URL}/health` (custom route on same FastAPI app)

## R4: Mock vs real AI mode (FR-011)

**Decision**: Environment-gated model factory at AgentOS startup.

| Condition | Mode | Behavior |
|-----------|------|----------|
| `OPENAI_API_KEY` unset | `mock` | Custom Agno-compatible model streams deterministic Traditional Chinese demo text in chunks |
| `OPENAI_API_KEY` set | `real` | `OpenAIResponses` (or `OpenAIChat`) model via Agno `Agent` |

**Rationale**:
- Matches clarified spec: pluggable mock-by-default, real when configured.
- No database or feature-flag service needed for v1 local dev.
- Mock streams tokens to satisfy FR-002 and SC-005 without external dependency.

**Alternatives considered**:
- Separate mock HTTP server — rejected; violates single-backend simplicity.
- `stream=false` JSON only for mock — rejected; spec requires streaming in both modes.

## R5: Extended health/status (FR-004)

**Decision**: Custom `GET /health` route (schema version `1.0.0`) alongside Agno's `GET /status`.

**Response fields**:
- `status`: `healthy` | `degraded` | `unhealthy`
- `agent_mode`: `mock` | `real`
- `upstream_reachable`: `true` | `false` | `null` (`null` when `agent_mode=mock`)
- `schema_version`: `1.0.0`
- `request_id`: correlation id (constitution Principle VI)

**Upstream check** (real mode only): lightweight probe (e.g., models list or minimal completion) with 2s timeout; failure → `degraded` or `unhealthy`.

**Rationale**:
- Agno `GET /status` returns only `{"status": "available"}` — insufficient for clarified acceptance criteria.
- Custom route keeps Agno interface intact while satisfying FR-004 without forking Agno.

## R6: Multi-turn conversation (FR-003a)

**Decision**: Rely on AG-UI `RunAgentInput.messages` populated by assistant-ui runtime; backend Agno agent receives full thread context per request.

**Rationale**:
- Clarified answer: multi-turn with full history per request.
- assistant-ui maintains in-memory thread; AG-UI protocol serializes messages array.
- Agno agent uses `add_history_to_context=True` only for server-side session when `session_id` present; v1 primary path is client-sent history (no DB per FR-007).

## R7: Input blocking during stream (FR-012)

**Decision**: assistant-ui `Thread` composer disabled while `status === "running"` (built-in runtime behavior + explicit `disabled` on send when streaming).

**Rationale**: Matches clarified Option A (block, no queue, no cancel).

## R8: Frontend backend URL configuration (FR-005)

**Decision**: `NEXT_PUBLIC_AGUI_AGENT_URL` environment variable (default `http://localhost:7777`).

**Rationale**:
- Next.js convention for browser-accessible env vars.
- Passed to `HttpAgent({ url: \`${base}/agui\` })`.
- Changing env + restart satisfies SC-004.

## R9: Message length limit

**Decision**: 4,000 characters per user message (planning default; clarify session Q5 was not answered — using edge-case recommendation).

**Rationale**: Documented in spec edge cases; sufficient for Traditional Chinese paragraphs without oversized payloads.

## R10: Structured observability (Constitution Principle VI)

**Decision**: JSON structured log events on backend (Python `structlog` or stdlib `logging` JSON formatter); frontend emits structured `console` events only in dev behind `DEBUG` flag — production console logging avoided.

**Required fields**: `request_id`, `event`, `timestamp`; optional `session_id`, `agent_mode`.

**Rationale**: Constitution compliance without full metrics stack in v1 local scope.

## R11: Testing strategy (Constitution Principle V)

| Layer | Scope |
|-------|--------|
| Unit | Message validation, health response builder, mock model token chunking |
| Integration | `GET /health`, `POST /agui` SSE stream (mock mode), CORS headers |
| E2E (manual) | quickstart.md scenarios via browser |

**Rule**: Do not mock owned backend modules in integration tests; mock external OpenAI API only in real-mode tests.

## R12: Canonical commands (Constitution Principle X)

**Decision**: Root `Makefile` with `help`, `dev`, `dev-backend`, `dev-frontend`, `test`, `lint`, `health` targets. CI invokes same targets.
