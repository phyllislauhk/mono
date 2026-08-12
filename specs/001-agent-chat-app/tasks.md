---
description: "Task list for Agent Chat App (v1) implementation"
---

# Tasks: Agent Chat App (v1)

**Input**: Design documents from `/specs/001-agent-chat-app/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: **Required** per constitution Principle V — unit tests for pure logic, integration tests for HTTP/AG-UI boundaries. Each phase checkpoint includes passing tests before proceeding.

**Organization**: Tasks grouped by user story to enable independent implementation and testing.

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
- [ ] T006 [P] Create `frontend/.env.example` with `NEXT_PUBLIC_AGUI_AGENT_URL=http://localhost:7777` and alternate-port comment
- [ ] T007 [P] Create `backend/src/agent_chat/__init__.py` package marker

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core backend AgentOS + AGUI and frontend shared libs — MUST complete before user story phases

**⚠️ CRITICAL**: No user story work can begin until this phase is complete (including T010 unit test)

- [ ] T008 Implement structured JSON event logger in `backend/src/agent_chat/logging.py` (fields: `request_id`, `event`, `timestamp`)
- [ ] T009 Implement deterministic streaming mock model in `backend/src/agent_chat/mock_model.py` (Traditional Chinese demo text, chunked output)
- [ ] T010 Implement `backend/tests/unit/test_mock_model.py` for deterministic streaming chunks (must pass before Phase 3)
- [ ] T011 Implement agent factory in `backend/src/agent_chat/agent.py` (mock when `OPENAI_API_KEY` unset, real `OpenAIResponses` when set; **`add_history_to_context=False`** — context from AG-UI client messages only, FR-003a)
- [ ] T012 Implement AgentOS app in `backend/src/agent_chat/main.py` with `AGUI(agent=...)`, CORS for `http://localhost:3000`, uvicorn serve on port 7777
- [ ] T013 [P] Create `frontend/src/lib/constants.ts` with `MAX_MESSAGE_LENGTH = 4000` and default `AGUI_BASE_URL`
- [ ] T014 [P] Create `frontend/src/lib/agent.ts` with `HttpAgent` factory: `${NEXT_PUBLIC_AGUI_AGENT_URL}/agui`, fallback `http://localhost:7777/agui` (FR-005)
- [ ] T015 [P] Create `frontend/src/app/layout.tsx` with `lang="zh-Hant"` and base metadata
- [ ] T016 Add `install`, `dev-backend` targets to `Makefile` (backend: `uv sync` + `uv run python -m agent_chat.main`)

**Checkpoint**: Backend serves `POST /agui` (mock mode); `test_mock_model` passes; frontend libs ready

---

## Phase 3: User Story 1 - 送出訊息並接收串流回覆 (Priority: P1) 🎯 MVP

**Goal**: Web chat UI sends Traditional Chinese messages and displays streaming agent replies in a single thread

**Independent Test**: Open `http://localhost:3000`, send「你好，請介紹你自己」, observe incremental streaming reply with in-progress indicator; first token within 3s (SC-001); input re-enables after stream completes (quickstart V3)

### Implementation for User Story 1

- [ ] T017 [P] [US1] Create `frontend/src/lib/messages.ts` with Traditional Chinese error strings (blank input, length exceeded, stream failed, backend unavailable)
- [ ] T018 [P] [US1] Create `frontend/src/lib/validation.ts` with `validateUserMessage(content)` enforcing non-blank and 4,000 char limit (FR-013)
- [ ] T019 [US1] Implement `frontend/tests/message-validation.test.ts` for `validateUserMessage` edge cases (must pass before T022)
- [ ] T020 [US1] Create `frontend/src/components/assistant-ui/thread.tsx` wrapping assistant-ui `Thread` — single thread only (no ThreadList adapter, FR-003); ensure streaming-in-progress indicator visible (US1 scenario 2)
- [ ] T021 [US1] Create `frontend/src/app/page.tsx` with `AssistantRuntimeProvider`, `useAgUiRuntime`, and `Thread` wired to `frontend/src/lib/agent.ts`
- [ ] T022 [US1] Integrate `validateUserMessage` in `frontend/src/components/assistant-ui/thread.tsx` to block invalid sends before AG-UI request
- [ ] T023 [US1] Disable composer input/send while runtime status is running in `frontend/src/app/page.tsx` (FR-012)
- [ ] T024 [US1] Add `onError` handler in `frontend/src/app/page.tsx` displaying Traditional Chinese error from `frontend/src/lib/messages.ts` (FR-010)
- [ ] T025 [US1] Verify multi-turn context: AG-UI `RunAgentInput.messages` includes full thread history on second send (FR-003a); confirm agent uses client messages only (`add_history_to_context=False`)
- [ ] T026 [US1] Implement `backend/tests/integration/test_agui_stream.py` for `POST /agui` SSE events in mock mode; assert first `TEXT_MESSAGE_CONTENT` within 3s (SC-001)
- [ ] T027 [US1] Add `dev-frontend` target to `Makefile` (`cd frontend && npm run dev`)

**Checkpoint**: User Story 1 fully functional; unit + integration tests pass — MVP demo ready (mock mode, no API key)

---

## Phase 4: User Story 2 - 檢查後端健康狀態 (Priority: P2)

**Goal**: `GET /health` returns extended status; `GET /status` remains AG-UI liveness check

**Independent Test**: `curl http://localhost:7777/health` returns JSON with `agent_mode` and `schema_version` 1.0.0 within 1s (SC-003); `curl http://localhost:7777/status` returns `available` (FR-004)

### Implementation for User Story 2

- [ ] T028 [P] [US2] Implement `HealthResponse` builder in `backend/src/agent_chat/health.py` per `specs/001-agent-chat-app/contracts/health.openapi.yaml`
- [ ] T029 [US2] Implement upstream reachability probe in `backend/src/agent_chat/health.py` (real mode only, 2s timeout)
- [ ] T030 [US2] Implement `backend/tests/unit/test_health.py` for HealthResponse status derivation logic (must pass before T032)
- [ ] T031 [US2] Register `GET /health` route on AgentOS FastAPI app in `backend/src/agent_chat/main.py`
- [ ] T032 [US2] Add structured logging to health handler in `backend/src/agent_chat/main.py` with `request_id` per response
- [ ] T033 [US2] Verify `GET /status` (AGUI built-in) remains available and returns basic liveness in `backend/src/agent_chat/main.py` (FR-004)
- [ ] T034 [US2] Implement `backend/tests/integration/test_health_endpoint.py` for `GET /health` and `GET /status`; assert `/health` responds within 1s (SC-003)
- [ ] T035 [US2] Add `health` target to `Makefile` (`curl -s http://localhost:7777/health | jq`)

**Checkpoint**: `make health` passes in mock mode; health integration tests pass; degraded status verifiable when real mode + unreachable upstream

---

## Phase 5: User Story 3 - 透過環境變數設定後端位址 (Priority: P3)

**Goal**: Frontend backend URL configurable via `NEXT_PUBLIC_AGUI_AGENT_URL` without code changes

**Independent Test**: Start backend on port 8888, set env, restart frontend, send message — request hits new port (quickstart V5, SC-004)

### Implementation for User Story 3

- [ ] T036 [US3] Add alternate-port example to `frontend/.env.example` documenting restart requirement after env change
- [ ] T037 [US3] Verify `frontend/next.config.ts` does not block `NEXT_PUBLIC_*` env passthrough to client bundle
- [ ] T038 [US3] Document env-switch procedure in root `README.md` under canonical commands section (FR-005, SC-004)
- [ ] T039 [US3] Validate env switch end-to-end per quickstart V5 (record in quickstart checklist)

**Checkpoint**: Changing `NEXT_PUBLIC_AGUI_AGENT_URL` and restarting frontend redirects chat traffic without source edits

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Edge cases, backend validation, command completeness, full validation

- [ ] T040 [P] Implement backend message length validation in `backend/src/agent_chat/validation.py` rejecting user messages >4,000 chars at AG-UI ingress (FR-013 backend enforcement)
- [ ] T041 [P] Handle stream disconnect mid-response in `frontend/src/app/page.tsx` (preserve partial content + retry prompt per edge case)
- [ ] T042 [P] Add `dev` target to `Makefile` (prints instructions to run backend + frontend in two terminals)
- [ ] T043 [P] Add `test` and `lint` targets to `Makefile` (`pytest` in backend, `eslint` + `vitest` in frontend)
- [ ] T044 [P] Add frontend logging rule: no unstructured `console.log` in `frontend/src/` except behind `DEBUG` flag; use structured objects when logging (constitution Principle VI)
- [ ] T045 Document real-mode streaming validation as manual gate in `specs/001-agent-chat-app/quickstart.md` V6 (FR-011 real mode; requires `OPENAI_API_KEY`)
- [ ] T046 Execute all scenarios in `specs/001-agent-chat-app/quickstart.md` (V1–V8) including page-refresh history clear and SC-002 pass/fail record
- [ ] T047 Update root `README.md` with full command index, prerequisites, and link to quickstart.md

---

## Dependencies & Execution Order

### Phase Dependencies

```text
Phase 1 (Setup)
    └──► Phase 2 (Foundational + T010 test)
              ├──► Phase 3 (US1 + T019, T026 tests)
              ├──► Phase 4 (US2 + T030, T034 tests)  ── parallel after Phase 2
              └──► Phase 5 (US3)  ── after T014 exists
                        └──► Phase 6 (Polish)
```

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — **BLOCKS all user stories**; T010 must pass before Phase 3
- **US1 (Phase 3)**: Depends on Phase 2; T019 + T026 required at checkpoint
- **US2 (Phase 4)**: Depends on Phase 2 — parallel with US1; T030 + T034 required at checkpoint
- **US3 (Phase 5)**: Depends on Phase 2 + T014; can run parallel with US1/US2 after T014
- **Polish (Phase 6)**: Depends on US1–US3 completion

### User Story Dependencies

| Story | Depends On | Independently Testable Via |
|-------|------------|---------------------------|
| US1 (P1) | Foundational + T010 | Browser chat; `pytest test_agui_stream` |
| US2 (P2) | Foundational + T010 | `curl /health`, `curl /status`; `pytest test_health_endpoint` |
| US3 (P3) | Foundational + T014 | Env change + quickstart V5 |

### Parallel Opportunities

**Phase 1** — after T001:
```text
T003 frontend ║ T002 backend ║ T005 Makefile ║ T006 .env.example
```

**Phase 2** — after T008:
```text
T013 constants.ts ║ T014 agent.ts ║ T015 layout.tsx
T009 mock_model → T010 test → T011 agent → T012 main
```

**Phase 3+4** — after Phase 2 checkpoint:
```text
Developer A: US1 (T017–T027)     Developer B: US2 (T028–T035)     parallel
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T007)
2. Complete Phase 2: Foundational (T008–T016) — **including T010 unit test**
3. Complete Phase 3: User Story 1 (T017–T027) — **including T019, T026 tests**
4. **STOP and VALIDATE**: quickstart V3; `make test` passes US1 tests
5. Demo MVP

### Incremental Delivery

1. Setup + Foundational → backend `POST /agui` works; mock model tested
2. **US1** → streaming chat MVP with tests
3. **US2** → `make health` + `/status` liveness; latency tests
4. **US3** → env-based backend switching
5. **Polish** → backend validation, edge cases, full quickstart V1–V8

### Suggested MVP Scope

**Phases 1–3** (T001–T027): delivers acceptance criteria #1 (streaming chat) in mock mode with constitution-compliant tests.

---

## Notes

- Total tasks: **47** (T001–T047)
- US1: **11** tasks | US2: **8** tasks | US3: **4** tasks
- Tests are **required** at phase checkpoints (T010, T019, T026, T030, T034), not optional polish
- Mock mode requires no `OPENAI_API_KEY`; real mode validated manually via quickstart V6 (T045)
- No database, auth, RAG, tools, or production deploy per spec out-of-scope
- Commit after each phase checkpoint
