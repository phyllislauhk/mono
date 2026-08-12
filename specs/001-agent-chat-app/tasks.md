---
description: "Task list for Agent Chat App (v1) implementation"
---

# Tasks: Agent Chat App (v1)

**Input**: Design documents from `/specs/001-agent-chat-app/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Not explicitly requested in spec — validation via quickstart.md scenarios in Polish phase. Constitution-aligned unit/integration tests listed as optional polish tasks.

**Organization**: Tasks grouped by user story for independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: User story label (US1, US2, US3)

## Path Conventions

- **Backend**: `backend/src/agent_chat/`
- **Frontend**: `frontend/src/`
- **Docs/Commands**: root `Makefile`, `README.md`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and dependency scaffolding

- [ ] T001 Create `backend/` and `frontend/` directory structure per `specs/001-agent-chat-app/plan.md`
- [ ] T002 Initialize `backend/pyproject.toml` with `agno[os,agui]`, `openai`, `structlog`, `httpx`, `pytest`, `ruff`
- [ ] T003 [P] Scaffold Next.js App Router project in `frontend/` (TypeScript, ESLint, src dir)
- [ ] T004 [P] Add assistant-ui dependencies to `frontend/package.json`: `@assistant-ui/react`, `@assistant-ui/react-ag-ui`, `@ag-ui/client`
- [ ] T005 [P] Create root `Makefile` with `help` target listing all canonical commands
- [ ] T006 [P] Create `frontend/.env.example` with `NEXT_PUBLIC_AGUI_AGENT_URL=http://localhost:7777`
- [ ] T007 [P] Create `backend/src/agent_chat/__init__.py` package marker

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core backend AgentOS + AGUI and frontend shared libs — MUST complete before user story phases

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T008 Implement structured JSON event logger in `backend/src/agent_chat/logging.py` (fields: `request_id`, `event`, `timestamp`)
- [ ] T009 Implement deterministic streaming mock model in `backend/src/agent_chat/mock_model.py` (Traditional Chinese demo text, chunked output)
- [ ] T010 Implement agent factory in `backend/src/agent_chat/agent.py` (select mock when `OPENAI_API_KEY` unset, real `OpenAIResponses` when set; `add_history_to_context=True`)
- [ ] T011 Implement AgentOS app in `backend/src/agent_chat/main.py` with `AGUI(agent=...)`, CORS for `http://localhost:3000`, uvicorn serve on port 7777
- [ ] T012 [P] Create `frontend/src/lib/constants.ts` with `MAX_MESSAGE_LENGTH = 4000` and default `AGUI_BASE_URL`
- [ ] T013 [P] Create `frontend/src/lib/agent.ts` with `HttpAgent` factory reading `process.env.NEXT_PUBLIC_AGUI_AGENT_URL`
- [ ] T014 [P] Create `frontend/src/app/layout.tsx` with `lang="zh-Hant"` and base metadata
- [ ] T015 Add `install`, `dev-backend` targets to `Makefile` (backend: `uv sync` + `uv run python -m agent_chat.main`)

**Checkpoint**: Backend serves `POST /agui` (mock mode) and frontend libs ready — user story implementation can begin

---

## Phase 3: User Story 1 - 送出訊息並接收串流回覆 (Priority: P1) 🎯 MVP

**Goal**: Web chat UI sends Traditional Chinese messages and displays streaming agent replies in a single thread

**Independent Test**: Open `http://localhost:3000`, send「你好，請介紹你自己」, observe incremental streaming reply; input re-enables after stream completes (quickstart V3)

### Implementation for User Story 1

- [ ] T016 [P] [US1] Create `frontend/src/lib/messages.ts` with Traditional Chinese error strings (blank input, length exceeded, stream failed, backend unavailable)
- [ ] T017 [P] [US1] Create `frontend/src/lib/validation.ts` with `validateUserMessage(content)` enforcing non-blank and 4,000 char limit (FR-013)
- [ ] T018 [US1] Create `frontend/src/components/assistant-ui/thread.tsx` wrapping assistant-ui `Thread` component with styled layout
- [ ] T019 [US1] Create `frontend/src/app/page.tsx` with `AssistantRuntimeProvider`, `useAgUiRuntime`, and `Thread` wired to `frontend/src/lib/agent.ts`
- [ ] T020 [US1] Integrate `validateUserMessage` in `frontend/src/components/assistant-ui/thread.tsx` to block invalid sends before AG-UI request
- [ ] T021 [US1] Disable composer input/send while runtime status is running in `frontend/src/app/page.tsx` (FR-012)
- [ ] T022 [US1] Add `onError` handler in `frontend/src/app/page.tsx` displaying Traditional Chinese error from `frontend/src/lib/messages.ts` (FR-010)
- [ ] T023 [US1] Verify multi-turn context: AG-UI `RunAgentInput.messages` includes full thread history on second send (FR-003a) — adjust `frontend/src/app/page.tsx` or runtime config if needed
- [ ] T024 [US1] Add `dev-frontend` target to `Makefile` (`cd frontend && npm run dev`)

**Checkpoint**: User Story 1 fully functional — MVP demo ready (mock mode, no API key)

---

## Phase 4: User Story 2 - 檢查後端健康狀態 (Priority: P2)

**Goal**: Extended `GET /health` returns service status, agent mode (mock/real), and upstream reachability

**Independent Test**: `curl http://localhost:7777/health` returns JSON with `agent_mode` and `schema_version` 1.0.0 within 1s (quickstart V1)

### Implementation for User Story 2

- [ ] T025 [P] [US2] Implement `HealthResponse` builder in `backend/src/agent_chat/health.py` per `specs/001-agent-chat-app/contracts/health.openapi.yaml`
- [ ] T026 [US2] Implement upstream reachability probe in `backend/src/agent_chat/health.py` (real mode only, 2s timeout)
- [ ] T027 [US2] Register `GET /health` route on AgentOS FastAPI app in `backend/src/agent_chat/main.py`
- [ ] T028 [US2] Add structured logging to health handler in `backend/src/agent_chat/main.py` with `request_id` per response
- [ ] T029 [US2] Add `health` target to `Makefile` (`curl -s http://localhost:7777/health | jq`)

**Checkpoint**: `make health` passes in mock mode; degraded status verifiable when real mode + unreachable upstream

---

## Phase 5: User Story 3 - 透過環境變數設定後端位址 (Priority: P3)

**Goal**: Frontend backend URL configurable via `NEXT_PUBLIC_AGUI_AGENT_URL` without code changes

**Independent Test**: Start backend on port 8888, set env, restart frontend, send message — request hits new port (quickstart V5)

### Implementation for User Story 3

- [ ] T030 [US3] Ensure `frontend/src/lib/agent.ts` constructs `HttpAgent` URL as `${baseUrl}/agui` with fallback `http://localhost:7777`
- [ ] T031 [US3] Add alternate-port example to `frontend/.env.example` documenting restart requirement after env change
- [ ] T032 [US3] Verify `frontend/next.config.ts` does not block `NEXT_PUBLIC_*` env passthrough to client bundle
- [ ] T033 [US3] Document env-switch procedure in root `README.md` under canonical commands section (FR-005, SC-004)

**Checkpoint**: Changing `NEXT_PUBLIC_AGUI_AGENT_URL` and restarting frontend redirects chat traffic without source edits

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Edge cases, command completeness, validation, documentation

- [ ] T034 [P] Handle stream disconnect mid-response in `frontend/src/app/page.tsx` (preserve partial content + retry prompt per edge case)
- [ ] T035 [P] Add `dev` target to `Makefile` (prints instructions to run backend + frontend in two terminals)
- [ ] T036 [P] Add `test` and `lint` targets to `Makefile` (`pytest` in backend, `eslint` in frontend)
- [ ] T037 [P] Implement `backend/tests/unit/test_health.py` for HealthResponse status derivation logic
- [ ] T038 [P] Implement `backend/tests/unit/test_mock_model.py` for deterministic streaming chunks
- [ ] T039 [P] Implement `backend/tests/integration/test_health_endpoint.py` against running app (mock mode)
- [ ] T040 [P] Implement `backend/tests/integration/test_agui_stream.py` for `POST /agui` SSE events in mock mode
- [ ] T041 [P] Implement `frontend/tests/message-validation.test.ts` for `validateUserMessage` edge cases
- [ ] T042 Execute all scenarios in `specs/001-agent-chat-app/quickstart.md` (V1–V8) and record pass/fail
- [ ] T043 Update root `README.md` with full command index, prerequisites, and link to quickstart.md

---

## Dependencies & Execution Order

### Phase Dependencies

```text
Phase 1 (Setup)
    └──► Phase 2 (Foundational) ──► Phase 3 (US1) ──► Phase 6 (Polish)
                              ├──► Phase 4 (US2)  ──┘
                              └──► Phase 5 (US3)  ──┘
```

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — **BLOCKS all user stories**
- **US1 (Phase 3)**: Depends on Phase 2 — MVP, no dependency on US2/US3
- **US2 (Phase 4)**: Depends on Phase 2 — independent of US1 frontend (testable via curl)
- **US3 (Phase 5)**: Depends on Phase 2 + US1 `agent.ts` existing — mostly config/docs
- **Polish (Phase 6)**: Depends on US1–US3 completion

### User Story Dependencies

| Story | Depends On | Independently Testable Via |
|-------|------------|---------------------------|
| US1 (P1) | Foundational | Browser chat at localhost:3000 |
| US2 (P2) | Foundational | `curl localhost:7777/health` |
| US3 (P3) | Foundational + T013 | Env change + chat request to alternate port |

### Parallel Opportunities

**Phase 1** — after T001:
```text
T003 frontend scaffold ║ T002 backend pyproject ║ T005 Makefile ║ T006 .env.example
```

**Phase 2** — after T008:
```text
T012 constants.ts ║ T013 agent.ts ║ T014 layout.tsx    (frontend, parallel)
T009 mock_model ║ T010 agent.py     (backend, sequential: mock before agent)
```

**Phase 3+4** — after Foundational checkpoint:
```text
Developer A: US1 (T016–T024)     Developer B: US2 (T025–T029)     parallel
```

**Phase 6** — all [P] tasks (T034–T041) can run in parallel

---

## Parallel Example: User Story 1

```bash
# Launch validation libs in parallel:
# T016 frontend/src/lib/messages.ts
# T017 frontend/src/lib/validation.ts

# Then sequentially wire UI:
# T018 thread.tsx → T019 page.tsx → T020 validation integrate → T021 disable on stream
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T007)
2. Complete Phase 2: Foundational (T008–T015)
3. Complete Phase 3: User Story 1 (T016–T024)
4. **STOP and VALIDATE**: quickstart V3 (streaming chat, mock mode)
5. Demo MVP

### Incremental Delivery

1. Setup + Foundational → backend `POST /agui` works
2. **US1** → streaming chat MVP
3. **US2** → `make health` diagnostics
4. **US3** → env-based backend switching
5. **Polish** → edge cases, tests, full quickstart validation

### Suggested MVP Scope

**Phases 1–3 only** (T001–T024): delivers acceptance criteria #1 (streaming chat) in mock mode.

---

## Notes

- Total tasks: **43** (T001–T043)
- US1: **9** tasks | US2: **5** tasks | US3: **4** tasks
- Mock mode requires no `OPENAI_API_KEY`; real mode optional for polish validation
- No database, auth, RAG, tools, or production deploy per spec out-of-scope
- Commit after each phase checkpoint
