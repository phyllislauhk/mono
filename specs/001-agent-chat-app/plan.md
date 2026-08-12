# Implementation Plan: Agent Chat App (v1)

**Branch**: `001-agent-chat-app` | **Date**: 2026-08-12 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-agent-chat-app/spec.md`

**User directive**: Use **assistant-ui** for frontend and **Agno SDK** for backend with **AgentOS** enabled (AGUI interface).

## Summary

Build a local-development agent chat application where users send Traditional Chinese messages in a web UI and receive streaming agent replies. The frontend uses **assistant-ui** with the **AG-UI runtime** (`@assistant-ui/react-ag-ui`). The backend uses **Agno AgentOS** with the **AGUI** interface exposing `POST /agui` (streaming) and a custom `GET /health` endpoint for extended status (mock/real mode, upstream reachability). v1 scope: single in-memory thread, no auth, no database, no RAG/tools/attachments, no production deployment.

## Technical Context

**Language/Version**: Python 3.11+ (backend), TypeScript 5.x (frontend)

**Primary Dependencies**:
- Backend: `agno[os,agui]`, `openai` (real mode), `uvicorn`, `structlog`
- Frontend: `next@15`, `react@19`, `@assistant-ui/react`, `@assistant-ui/react-ag-ui`, `@ag-ui/client`

**Storage**: N/A (in-memory only; FR-007)

**Testing**: `pytest` + `httpx` (backend integration), `vitest` + `@testing-library/react` (frontend unit), manual E2E via quickstart.md

**Target Platform**: Local development (macOS/Linux/Windows); browser (Chrome/Firefox/Safari latest)

**Project Type**: Web application (frontend + backend)

**Performance Goals**: First stream token visible within 3s (SC-001); health response within 1s (SC-003)

**Constraints**:
- Single chat thread per browser session
- 4,000 character user message limit (planning default)
- Mock mode must work without API keys
- Traditional Chinese UI strings for errors

**Scale/Scope**: Single developer/demo user; no concurrent multi-tenant requirements

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Pre-Design | Post-Design | Notes |
|-----------|--------------|-------------|-------|
| I. Do Not Distribute by Default | ⚠️ Exception | ⚠️ Exception | Two dev processes (Next.js + AgentOS) — not microservices; single product, no queue/DB. Justified in Complexity Tracking. |
| II. Optimize for Deletion | ✅ Pass | ✅ Pass | Small modules: `agent.py`, `mock_model.py`, `health.py`, single `Chat` page |
| III. Explicit Dependencies | ✅ Pass | ✅ Pass | Model factory injects dependencies; env vars read at startup only |
| IV. Contract at Boundary | ✅ Pass | ✅ Pass | AG-UI protocol + OpenAPI `health.openapi.yaml` v1.0.0 |
| V. Test Transformations | ✅ Pass | ✅ Pass | Unit tests for validation/health builder; integration for `/agui` SSE |
| VI. Structured Events | ✅ Pass | ✅ Pass | `request_id` on all backend logs and health responses |
| VII. Recovery Over Prevention | ✅ Pass | ✅ Pass | Mock mode default; env toggle for real mode; no migrations |
| VIII. Attention Is Finite | ✅ Pass (v1) | ✅ Pass (v1) | No paging in v1 local scope; health endpoint is diagnostic only |
| IX. Value at User | ✅ Pass (v1) | ✅ Pass (v1) | quickstart.md defines deploy-to-local validation |
| X. Commands Discoverable | ✅ Pass | ✅ Pass | Root `Makefile` with `help` target |

**Post-design re-check**: No new violations introduced. AG-UI + AgentOS is the minimal integration path for assistant-ui + Agno per user directive.

## Project Structure

### Documentation (this feature)

```text
specs/001-agent-chat-app/
├── plan.md              # This file
├── research.md          # Phase 0
├── data-model.md        # Phase 1
├── quickstart.md        # Phase 1
├── contracts/           # Phase 1
│   ├── README.md
│   └── health.openapi.yaml
└── tasks.md             # Phase 2 (/speckit-tasks — not yet created)
```

### Source Code (repository root)

```text
backend/
├── pyproject.toml
├── src/agent_chat/
│   ├── __init__.py
│   ├── main.py           # AgentOS + AGUI mount + CORS + custom /health
│   ├── agent.py          # Agent factory (mock/real mode selection)
│   ├── mock_model.py     # Deterministic streaming mock model
│   ├── health.py         # HealthResponse builder + upstream probe
│   └── logging.py        # Structured JSON event logger
└── tests/
    ├── unit/
    │   ├── test_health.py
    │   └── test_mock_model.py
    └── integration/
        ├── test_agui_stream.py
        └── test_health_endpoint.py

frontend/
├── package.json
├── next.config.ts
├── .env.example
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   └── page.tsx          # Chat page
│   ├── components/
│   │   └── assistant-ui/
│   │       └── thread.tsx    # assistant-ui Thread wrapper
│   └── lib/
│       ├── agent.ts          # HttpAgent factory (env URL)
│       └── constants.ts      # MAX_MESSAGE_LENGTH = 4000
└── tests/
    └── message-validation.test.ts

Makefile                    # Canonical command index (Principle X)
```

**Structure Decision**: Web application layout (`frontend/` + `backend/`) chosen because React browser runtime and Python AgentOS are genuinely independent compute environments (Constitution I exception). No shared code packages — all contracts at HTTP/AG-UI boundary.

## Architecture

```text
┌─────────────────────────────────────────────────────────┐
│  Browser (localhost:3000)                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │  assistant-ui Thread + useAgUiRuntime             │  │
│  │  HttpAgent → NEXT_PUBLIC_AGUI_AGENT_URL/agui      │  │
│  └───────────────────────┬───────────────────────────┘  │
└──────────────────────────┼──────────────────────────────┘
                           │ AG-UI SSE (POST /agui)
                           │ JSON (GET /health)
┌──────────────────────────▼──────────────────────────────┐
│  Agno AgentOS (localhost:7777)                        │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ AGUI iface  │  │ Custom /health│  │ Agent factory │  │
│  │ POST /agui  │  │ GET /health   │  │ mock | real   │  │
│  └──────┬──────┘  └──────────────┘  └───────┬───────┘  │
│         │                                    │          │
│         └──────────── Agno Agent ────────────┘          │
│                    (streaming runs)                     │
└─────────────────────────────────────────────────────────┘
                           │ (real mode only)
                           ▼
                    OpenAI API
```

## Implementation Phases (high-level)

### Phase A: Backend scaffold
1. `pyproject.toml` with `agno[os,agui]`
2. `main.py`: AgentOS + `AGUI(agent=...)` + CORS + `/health` route
3. `agent.py`: factory selecting mock vs real model by `OPENAI_API_KEY`
4. `mock_model.py`: deterministic Traditional Chinese streaming response
5. `health.py`: schema v1.0.0 response + upstream probe

### Phase B: Frontend scaffold
1. `npx create-next-app` + assistant-ui packages
2. `page.tsx`: `AssistantRuntimeProvider` + `useAgUiRuntime` + `Thread`
3. `agent.ts`: `HttpAgent` with `NEXT_PUBLIC_AGUI_AGENT_URL`
4. Input validation (blank, 4000 char limit, disable while streaming)
5. Traditional Chinese error messages

### Phase C: Integration & validation
1. Wire CORS for `localhost:3000`
2. Verify multi-turn via AG-UI messages array
3. Execute quickstart.md scenarios V1–V8
4. `Makefile` commands + `make test`

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Two dev processes (frontend + backend) | Browser JS runtime cannot host Python Agno AgentOS; user mandated both assistant-ui and Agno SDK | Single-process — impossible across Python/JS runtimes |
| Custom `/health` beyond Agno `GET /status` | Clarified FR-004 requires `agent_mode` + `upstream_reachable` | Built-in `/status` only returns `available` |

## Phase 0 & Phase 1 Outputs

| Artifact | Path | Status |
|----------|------|--------|
| Research | [research.md](./research.md) | ✅ Complete |
| Data Model | [data-model.md](./data-model.md) | ✅ Complete |
| Contracts | [contracts/](./contracts/) | ✅ Complete |
| Quickstart | [quickstart.md](./quickstart.md) | ✅ Complete |

## Next Step

Run `/speckit-tasks` to generate actionable implementation tasks from this plan.
