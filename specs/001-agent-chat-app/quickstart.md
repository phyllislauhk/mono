# Quickstart: Agent Chat App (v1)

**Date**: 2026-08-12  
**Purpose**: End-to-end validation scenarios for local development.

## Prerequisites

- Python 3.11+
- Node.js 20+
- [uv](https://docs.astral.sh/uv/) installed
- (Optional) `OPENAI_API_KEY` for real AI mode

## Setup

```bash
# From repository root
make install    # install backend (uv) + frontend (npm) dependencies
```

## Start services

**Terminal 1 — Backend (mock mode, no API key required):**

```bash
make dev-backend
# Expected: AgentOS listening on http://localhost:7777
# OpenAPI docs: http://localhost:7777/docs
```

**Terminal 2 — Frontend:**

```bash
make dev-frontend
# Expected: Next.js on http://localhost:3000
```

## Validation scenarios

### V1: Health endpoint (Acceptance criteria #2)

```bash
make health
```

**Expected output** (mock mode):

```json
{
  "schema_version": "1.0.0",
  "status": "healthy",
  "agent_mode": "mock",
  "upstream_reachable": null,
  "request_id": "<uuid>",
  "timestamp": "<iso8601>"
}
```

**Pass criteria**: HTTP 200, `agent_mode` present, response within 1s (SC-003).

### V2: AG-UI status (built-in)

```bash
curl -s http://localhost:7777/status
```

**Expected**: `{"status":"available"}`

### V3: Streaming chat — mock mode (Acceptance criteria #1)

1. Open http://localhost:3000
2. Type `你好，請介紹你自己` and send
3. Observe:
   - User message appears immediately
   - Agent reply streams incrementally (within 3s for first token — SC-001)
   - Input disabled while streaming (FR-012)
   - Input re-enabled after stream completes

**Pass criteria**: SC-001, SC-005, FR-012 satisfied.

### V4: Multi-turn context (FR-003a)

1. Send: `我的名字是小明`
2. After reply completes, send: `我剛剛說我叫什麼名字？`
3. **Expected**: Agent reply references「小明」 (context from prior turn)

### V5: Backend URL via environment variable (Acceptance criteria #3)

1. Stop frontend
2. Start backend on alternate port:
   ```bash
   AGENT_OS_PORT=8888 make dev-backend
   ```
3. Start frontend with custom URL:
   ```bash
   NEXT_PUBLIC_AGUI_AGENT_URL=http://localhost:8888 make dev-frontend
   ```
4. Send a chat message

**Expected**: Message reaches backend on port 8888 (verify backend logs). No frontend code changes.

### V6: Real AI mode (optional)

```bash
export OPENAI_API_KEY="sk-..."
make dev-backend
make health
```

**Expected**: `agent_mode: "real"`, `upstream_reachable: true` (when OpenAI reachable).

Repeat V3 with real model responses.

### V7: Error handling

1. Stop backend while frontend running
2. Send a message

**Expected**: Traditional Chinese error message in UI; retry possible after restarting backend (FR-010).

### V8: Edge cases

| Case | Action | Expected |
|------|--------|----------|
| Blank message | Send empty input | Blocked with prompt; no backend request |
| Long message | Paste >4,000 chars | Rejected with length error |
| Refresh | Reload page mid-conversation | History cleared; new thread |
| Double send | Click send during stream | Input disabled; no second request |

## Run tests

```bash
make test     # unit + integration (mock mode)
make lint     # ruff + eslint
```

## Stop services

`Ctrl+C` in each terminal.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| CORS error in browser | Backend `cors_allowed_origins` includes `http://localhost:3000` |
| Stream not appearing | `curl -N -X POST http://localhost:7777/agui` with sample `RunAgentInput` |
| `agent_mode: real` without key | Ensure `OPENAI_API_KEY` unset and restart backend |
| Frontend wrong backend | Verify `NEXT_PUBLIC_AGUI_AGENT_URL` and restart `make dev-frontend` |

## References

- [spec.md](./spec.md) — functional requirements
- [data-model.md](./data-model.md) — entities and validation
- [contracts/README.md](./contracts/README.md) — API contracts
- [research.md](./research.md) — technology decisions
