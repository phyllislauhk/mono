# Feature Specification: Agent Chat App (v1)

**Feature Directory**: `specs/001-agent-chat-app`

**Created**: 2026-08-12

**Status**: Draft

**Input**: User description: "建立一個簡單的 agent chat app。使用者可在 web 介面輸入繁體中文訊息，並收到由 backend agent 串流回傳的回覆。v1 只需要單一聊天 thread；不包含登入、資料庫、RAG、tools、上傳附件或 production deployment。"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 送出訊息並接收串流回覆 (Priority: P1)

使用者開啟 web 聊天介面，在輸入框以繁體中文輸入問題或指令，送出後在畫面上即時看到 agent 以串流方式逐段顯示回覆內容，無需等待完整回覆結束才顯示。

**Why this priority**: 這是產品核心價值——讓使用者與 agent 進行即時對話。沒有此功能，其餘能力皆無意義。

**Independent Test**: 僅實作聊天介面與串流回覆即可獨立驗證；使用者能完成一輪「輸入 → 送出 → 看到串流回覆」即代表 MVP 成立。

**Acceptance Scenarios**:

1. **Given** 使用者已開啟聊天頁面且後端可用，**When** 使用者輸入繁體中文訊息「你好，請介紹你自己」並送出，**Then** 使用者訊息出現在對話區，且 agent 回覆以串流方式逐步顯示於同一對話區。
2. **Given** agent 正在串流回覆，**When** 回覆尚未結束，**Then** 使用者已能看到目前已收到的部分內容，且介面標示回覆進行中（例如載入指示或游標閃爍）。
3. **Given** agent 已完成串流回覆，**When** 使用者檢視對話區，**Then** 完整回覆內容保留顯示，且使用者可繼續輸入下一則訊息。

---

### User Story 2 - 檢查後端健康狀態 (Priority: P2)

開發者或維運人員需要在不進行聊天互動的情況下，確認後端 agent 服務是否正常運作。

**Why this priority**: 可觀測的 health/status 端點是除錯與本地開發的基本需求，對應驗收條件第 2 項。

**Independent Test**: 僅實作 health/status 端點即可獨立驗證；對該端點發出請求並收到明確狀態回應即通過。

**Acceptance Scenarios**:

1. **Given** 後端服務已啟動且運作正常，**When** 對 health/status 端點發出檢查請求，**Then** 回應表示服務健康（例如狀態為 ok 或 healthy）。
2. **Given** 後端服務未啟動或無法處理請求，**When** 對 health/status 端點發出檢查請求，**Then** 回應明確表示不可用，或請求失敗且可被偵測（例如連線錯誤或非成功狀態碼）。

---

### User Story 3 - 透過環境變數設定後端位址 (Priority: P3)

開發者需要將前端指向不同環境的後端（例如本機、同事機器、測試實例），且不需修改前端程式碼。

**Why this priority**: 對應驗收條件第 3 項；支援本地開發與不同後端實例切換，無需重新建置前端。

**Independent Test**: 僅設定環境變數並重啟或重新載入前端即可驗證；變更變數後聊天請求應送往新位址。

**Acceptance Scenarios**:

1. **Given** 環境變數已設定為後端 A 的位址，**When** 使用者送出聊天訊息，**Then** 請求送往後端 A。
2. **Given** 環境變數已變更為後端 B 的位址且前端已重新載入，**When** 使用者送出聊天訊息，**Then** 請求送往後端 B，且無需修改前端原始碼。

---

### Edge Cases

- 使用者送出空白或僅含空白字元的訊息時，系統應阻止送出或提示輸入有效內容，不應向後端發送無效請求。
- 後端在串流中途斷線或逾時時，介面應顯示已收到部分內容，並以明確訊息告知回覆中斷，允許使用者重試。
- 使用者在 agent 尚未回覆完成時再次送出訊息時，系統應排隊或暫停新送出，避免同一 thread 內回覆交錯混亂。
- 使用者輸入極長訊息（例如超過 4,000 字）時，系統應拒絕或截斷並提示長度限制。
- 重新整理頁面後，對話紀錄清空（v1 無持久化）；使用者應能立即開始新對話。

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: 系統 MUST 提供 web 聊天介面，允許使用者輸入並送出繁體中文文字訊息。
- **FR-002**: 系統 MUST 將使用者訊息傳送至後端 agent，並以串流方式接收回覆，逐段顯示於介面上。
- **FR-003**: 系統 MUST 維持單一聊天 thread（單一連續對話區）；v1 不支援多 thread 切換或建立。
- **FR-004**: 系統 MUST 提供可獨立存取的 health/status 端點，回傳後端服務是否可用的明確狀態。
- **FR-005**: 前端 MUST 透過環境變數設定後端 API 位址；變更環境變數後，前端行為應指向新位址，無需修改程式碼。
- **FR-006**: 系統 MUST NOT 在 v1 實作使用者登入、身分驗證或權限管理。
- **FR-007**: 系統 MUST NOT 在 v1 使用資料庫或任何跨工作階段的訊息持久化。
- **FR-008**: 系統 MUST NOT 在 v1 實作 RAG、外部 tools、檔案上傳或附件功能。
- **FR-009**: 系統 MUST NOT 在 v1 包含 production deployment 設定或流程（僅限本地/開發環境使用）。
- **FR-010**: 當後端不可用或串流失敗時，系統 MUST 向使用者顯示可理解的錯誤訊息（繁體中文），並允許重試。

### Key Entities

- **Message**: 單則對話內容，包含角色（使用者或 agent）、文字內容、顯示順序。v1 僅存在於目前工作階段的記憶體中。
- **Chat Thread**: 單一連續對話序列，由依序排列的 Message 組成。v1 僅允許一個 thread，無名稱或分頁。

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 使用者在送出訊息後，3 秒內開始看到 agent 回覆的第一段串流內容（後端正常運作時）。
- **SC-002**: 100% 的驗收測試案例（三項 acceptance criteria）可通過手動或自動化驗證。
- **SC-003**: health/status 端點可在 1 秒內回應，且回應內容足以判斷服務是否可用，無需進行聊天互動。
- **SC-004**: 變更後端位址環境變數後，開發者可在 5 分鐘內（含重啟/重載）完成切換並成功送出訊息，無需修改前端程式碼。
- **SC-005**: 串流回覆過程中，使用者可見內容隨接收逐步增加；完整回覆結束後，最終文字與串流過程一致，無遺漏或重複段落。

## Assumptions

- v1 目標為本地開發與示範用途，不面向 production 流量或高可用需求。
- 使用者介面語言與錯誤訊息以繁體中文為主；agent 回覆語言以繁體中文為預期，但不強制限制其他語言輸出。
- 單一使用者、單一瀏覽器分頁；不支援多裝置同步或跨分頁共享對話。
- 重新整理頁面後對話紀錄清空，符合 v1 無持久化假設。
- Agent 的智慧與回覆品質取決於後端所連接的模型或服務；v1 規格不定義模型選擇或 prompt 策略。
- 前端與後端為兩個可獨立啟動的元件，但 v1 仍維持最小架構（不引入額外中介服務或佇列）。
- 網路連線為穩定本地或區域網路；不處理離線模式。

## Out of Scope (v1)

- 使用者登入與身分驗證
- 資料庫與訊息持久化
- RAG（檢索增強生成）
- Agent tools 與 function calling
- 檔案上傳與附件
- 多聊天 thread 管理
- Production deployment、CI/CD、監控告警
- 行動裝置專用介面或離線支援
