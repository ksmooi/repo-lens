---
repo: infiniflow/ragflow
file: 3-key-patterns
studied_at: 2026-05-24
commit_sha: e6dd3975
---

# RAGFlow · 值得偷學的設計

## Pattern 1: DAG-based Component System 取代傳統 Pipeline

**是什麼**: RAGFlow 將 RAG 流程表達為一個由元件（Component）構成的 DAG，而不是傳統的順序 pipeline。每個元件是獨立的功能單元（Begin、LLM、Retrieval、Generate、Categorize、Switch、Loop 等），透過 JSON DSL 定義 `upstream` / `downstream` 連接關係。執行引擎（`Graph` 類別）做 topological sort 後依序執行。

**為什麼有效**: 傳統 RAG pipeline 的每一步都是固定的 sequential chain（retrieve → rerank → generate）。但真實世界的 RAG 場景需要的不是線性流程——可能需要先 categorize 再決定 query expansion 策略、可能需要 loop 來處理多文件 chunk、可能需要 branch 來處理不同類型的查詢。DAG 表達力比 pipeline 強得多，且不需要改程式碼就能重組流程。

**程式碼位置**:
- Graph 基礎類別: [`agent/canvas.py:43-80`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/canvas.py#L43-L80)
- Canvas（Graph 的子類，實際執行者）: [`agent/canvas.py:285-847`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/canvas.py#L285-L847)
- 元件列表: [`agent/component/`](https://github.com/infiniflow/ragflow/tree/e6dd3975/agent/component)
- DSL 範例結構: [`agent/canvas.py:43-80`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/canvas.py#L43-L80)

**何時可以借用**: 當你的系統需要讓使用者（不一定是開發者）自訂多步驟流程，且不同場景需要不同流程時。例如：
- RAG pipeline 的自訂配置（不同查詢用不同 retrieval 策略）
- 資料處理 pipeline
- 表單 / 審批流程

**注意事項**:
- DSL 是 JSON，缺少靜態類型檢查。拼錯元件名、參數名錯誤只能在執行時發現。
- DAG 的執行路徑不是預先決定的——`Categorize` 和 `Switch` 元件可以在運行時決定走哪條分支，這讓除錯更困難（同一個 DSL 可能因輸入不同而走完全不同的路徑）。
- `Graph` 實現了類似 dataflow 的執行模型（input 就緒才執行），這種模型對 side-effect 敏感的元件（如檔案寫入）需要謹慎處理。

**替代方案**:
- **程式碼層級的 Pipeline（LangChain Chain / LlamaIndex Query Pipeline）**: 更靈活（可以寫任意邏輯），但不可視覺化、不可持久化為配置。
- **狀態機（State Machine）**: 比 DAG 更適合處理「事件驅動」的流程，但不適合「資料流轉」的場景。
- **DAG + 低程式碼編輯器**: RAGFlow 的路線。UI 拖放 → JSON → Graph 執行。適合非開發者，但複雜條件邏輯仍然在 Switch / Categorize 元件中受限。

---

## Pattern 2: Component Registry 透過 `component_class` 動態載入

**是什麼**: RAGFlow 的元件不是透過類別繼承或裝飾器註冊的，而是透過 `agent/component/__init__.py` 中的 `component_class` 字典將字串名稱映射到類別。DSL 中的 `component_name` 欄位（如 `"Begin"`, `"LLM"`, `"Retrieval"`）在執行時透過這個字典動態解析對應的類別。

**為什麼有效**: 這種「字串 → 類別」的 registry 模式讓 DSL 可以完全脫離 Python 實作被序列化、儲存、版本控制。要新增一個元件只需要 import 並註冊到字典即可，不需要修改 Graph 執行引擎。

**程式碼位置**: [`agent/component/__init__.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/component/__init__.py)

**何時可以借用**: 任何需要 pluggable component 的系統：
- ETL pipeline component registry
- CI/CD step registry
- 自訂指令處理器

**注意事項**:
- 所有元件必須在同一個 `__init__` 中被 import，否則不會被註冊。這表示大型系統的 import overhead 會集中在此。
- 如果元件需要 outside-in 的依賴注入（例如需要外部 service 實例），registry 模式需要額外處理。RAGFlow 的元件透過 `Canvas` 物件作為 bridge 存取外部服務（如 DB、LLM），元件本身不直接依賴外部 service。

**替代方案**:
- **裝飾器註冊（`@register_component("name")`）**: 更顯式，但需要額外的裝飾器掃描機制（如 `pkgutil.walk_packages`）。
- **Python entry point plugin system**: 適合第三方插件，但對內部元件的管理過於重量級。
- **Subclass detection（`__subclasses__()`）**: 全自動但不可控制註冊順序，且無法為同一類別註冊多個名稱。

---

## Pattern 3: Go + Python 雙語言搜尋引擎抽象層

**是什麼**: RAGFlow 的 Go 子系統在 `internal/engine/` 中對 Elasticsearch 與 Infinity 兩種 document store 做了統一抽象，透過 `engine/elasticsearch/` 和 `engine/infinity/` 實作。Python 端也有對應的搜尋服務（`api/db/services/search_service.py`）向 Go 發送 HTTP 請求。

**為什麼有效**: RAGFlow 的創建者同樣維護 Infinity（一個專用向量資料庫），因此有強烈的動機讓 RAGFlow 可以無縫切換到底層引擎。抽象層讓兩種 engine 對上層提供一致的搜尋介面，使用者在配置檔中切換 `DOC_ENGINE=infinity` 即可更換引擎，不需要修改應用程式碼。

**程式碼位置**:
- Engine 抽象目錄: [`internal/engine/`](https://github.com/infiniflow/ragflow/tree/e6dd3975/internal/engine)
- ES 實作: [`internal/engine/elasticsearch/`](https://github.com/infiniflow/ragflow/tree/e6dd3975/internal/engine/elasticsearch)
- Infinity 實作: [`internal/engine/infinity/`](https://github.com/infiniflow/ragflow/tree/e6dd3975/internal/engine/infinity)
- Engine types: [`internal/engine/types/`](https://github.com/infiniflow/ragflow/tree/e6dd3975/internal/engine/types)
- Python 端搜尋服務: [`api/db/services/search_service.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/api/db/services/search_service.py)

**何時可以借用**: 你的專案需要支援多種後端（特別是當其中一個後端是自己維護的），且希望使用者在配置時選擇，而不是在程式碼層級決定。

**注意事項**:
- 抽象層的介面越通用，特定 engine 的獨特功能就越難暴露。ES 的 advanced aggregation 和 Infinity 的某種混合查詢語法可能無法被統一介面完全涵蓋。
- 維護兩套 engine 實作的成本——每次新增功能（如新的 filter type）需要同時修改兩個 engine 的實作。
- RAGFlow 的雙語言架構讓這個抽象不僅跨 engine，還跨程式語言（Python 端透過 HTTP 與 Go 端的 engine 抽象層對話）。

**替代方案**:
- **Single engine + plugin hooks**: 只支援一個主要 engine，但對其他 engine 提供 plugin 整合。維護成本更低但靈活性也較低（如 LightRAG 的做法）。
- **抽象儲存層在 Python 端實現**: 全部用 Python，不需要 Go 層。但 Python 的 sync IO 對高吞吐搜尋場景可能成為瓶頸。

---

## Pattern 4: 全域變數系統 + 變數插值 (Variable Interpolation)

**是什麼**: RAGFlow 的 Graph 維護一個 `globals` dict，包含系統級變數（以 `sys.` 為前綴，如 `sys.query`、`sys.files`、`sys.user_id`）和元件輸出（透過 `{component_id}@{variable_name}` 引用）。執行時，元件可以從 `globals` 讀取其他元件的輸出或系統狀態。

**為什麼有效**: 這實現了 component 之間的鬆耦合通訊——元件不需要知道彼此的存在，只需要知道「我需要一個叫 `sys.query` 的值」。這讓 DSL 在編輯時可以任意重排 component，只要變數名稱一致，資料就會正確傳遞。

**程式碼位置**:
- 變數解析邏輯: [`agent/canvas.py:168-210`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/canvas.py#L168-L210)
- 系統變數設定: [`agent/canvas.py:375-410`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/canvas.py#L375-L410)

**何時可以借用**: 任何需要動態組合資料流的 DSL 系統：
- CI/CD pipeline 的變數傳遞
- 資料轉換 pipeline
- 低程式碼自動化平台

**注意事項**:
- 變數名稱依賴字串匹配 (`{sys.query}`、`{retrieval_0@chunks}`)，沒有 IDE 自動補全或編譯時檢查。錯字會導致 `None` 或空字串，且只會在執行時發現。
- `globals` 是可變 dict——任何元件都可以對它做任何操作（讀取、寫入、刪除）。這表示元件之間可以透過 globals 作隱式的副作用，導致非預期的行為。

**替代方案**:
- **Explicit input/output ports**: 每個元件宣告 input/output schema，由引擎在執行前做 type checking。更安全但在 DSL 編輯器中 UI 更複雜（如 Node-RED 的做法）。
- **Function composition（函數組合）**: 元件直接以程式碼方式 pipe。更靈活但無法序列化與視覺化。
- Reactive streams: 如 RxPY，元件訂閱資料流。更適合非同步事件流。

---

## Pattern 5: LiteLLM Provider 抽象 + Factory Defaults

**是什麼**: RAGFlow 透過 LiteLLM 封裝了 40+ 種 LLM provider，定義了 `SupportedLiteLLMProvider` enum 和 `FACTORY_DEFAULT_BASE_URL` 字典。每個 provider 的模型名稱加上前綴（`dashscope/`、`gemini/`、`deepseek/`）以區分不同 provider 的同名模型。

**為什麼有效**: RAGFlow 不需要自己維護 40+ provider 的 API 差異——LiteLLM 已經處理了認證、錯誤格式統一、streaming 協議轉換等。RAGFlow 在此基礎上增加了兩層價值：
1. `FACTORY_DEFAULT_BASE_URL` 讓使用者不需要記住每個 provider 的 endpoint URL
2. 模型類型分離（chat / embedding / rerank / cv / ocr / tts）讓 pipeline 可以為不同階段配置不同模型

**程式碼位置**:
- Provider enum 與預設 URL: [`rag/llm/__init__.py:25-96`](https://github.com/infiniflow/ragflow/blob/e6dd3975/rag/llm/__init__.py#L25-L96)
- ChatModel 實作: [`rag/llm/chat_model.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/rag/llm/chat_model.py)

**何時可以借用**: 你的系統需要支援多個 LLM provider，且不希望在維護 provider 差異上花太多精力。

**注意事項**:
- LiteLLM 是活躍開發的套件，版本更新可能引入 breaking changes。RAGFlow 對 `litellm` 的版本 pinned（`litellm~=1.82.0,!=1.82.7,!=1.82.8`）反映了這種顧慮。
- 部分 provider 的 edge case（如非標準的 streaming 行為）可能需要 RAGFlow 層級的 workaround。
- 不是所有 provider 都支援所有功能（如 function calling），這需要在上層做 fallback 邏輯。

**替代方案**:
- **自製 Provider Adapter（LangChain 的做法）**: 更可控但維護成本高。
- **Direct OpenAI SDK + proxy（如 OneAPI）**: 如果主要使用 OpenAI-compatible API，可以在 proxy 層處理 provider 切換。
- **同 provider 綁定（vLLM / Ollama only）**: 適合單一部署環境，不需要 multi-provider 支援。

---

## Pattern 6: Redis-backed Pipeline 進度追蹤 + SSE 串流

**是什麼**: RAGFlow 的 pipeline 執行狀態透過 Redis 追蹤，每個 component 執行時調用 `callback()` 方法，將進度寫入 Redis（key: `{flow_id}-{task_id}-logs`）。前端透過 SSE（Server-Sent Events）即時接收進度更新。支援取消操作——被取消的 task 會在所有 component 的 `invoke()` 入口檢查 `has_canceled()`，拋出 `TaskCanceledException`。

**為什麼有效**: 使用者可以即時看到 pipeline 執行了哪個 component、每個 component 花了多少時間、當前進度百分比。對於複雜的 agent 調用（可能包含多輪 LLM 呼叫），這種透明度對開發者除錯和終端使用者體驗都很重要。

**程式碼位置**:
- Callback 實作: [`rag/flow/pipeline.py:43-98`](https://github.com/infiniflow/ragflow/blob/e6dd3975/rag/flow/pipeline.py#L43-L98)
- 取消檢查: [`agent/canvas.py:429-432`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/canvas.py#L429-L432)
- SSE 串流: [`api/db/services/canvas_service.py:360-368`](https://github.com/infiniflow/ragflow/blob/e6dd3975/api/db/services/canvas_service.py#L360-L368)

**何時可以借用**: 任何長時間執行的 pipeline（超過 5 秒）都應該提供進度可視化。特別是：
- RAG pipeline（涉及多輪 LLM 呼叫）
- 資料處理 pipeline
- 模型訓練 / 推論任務

**注意事項**:
- Redis 的 TTL（30 分鐘）表示長時間執行的 pipeline 可能在執行中途失去進度追蹤。
- 進度計算方式是簡單的 `1.0 / component_count`，對執行時間不均的 pipeline（例如一個 LLM 呼叫佔 80% 時間、其他元件佔 20%）會顯示不準確的進度。
- 取消機制需要每個 component 的 `invoke()` 函數有取消檢查點。如果某個 component 長時間阻塞在外部 API 呼叫中（如 LLM timeout），取消不會立即生效。

**替代方案**:
- **WebSocket 取代 SSE**: 雙向通訊，支援更豐富的事件類型。但 SSE 對單向串流已經夠用，實作更簡單。
- **DB 寫入進度**: MySQL/PG 的進度查詢更靈活，但寫入頻率受限（2x / 秒對 DB 已經是高頻寫入）。

---

## 設計哲學觀察

RAGFlow 的作者對 RAG 系統的設計哲學可以歸納為：

1. **「RAG 不只是 retrieval + generation」**: DeepDoc 模組的存在說明了作者認為文件解析品質是 RAG 效能的瓶頸，比 retrieval 策略更重要。這與多數 RAG 框架專注於 retrieval 策略形成對比。

2. **「宣告式比命令式好」**: 選擇 JSON DSL 而非程式碼 API，說明作者優先考慮可視化編輯與非開發者使用體驗，而非開發者的靈活性。

3. **「高品質的工具比豐富的生態更重要」**: 內建 tool 清單相對精簡（對比 LangChain 的數百種 tool），但每個內建工具的品質（如 web search、code executor）都經過 sandbox 安全考量。

4. **「企業部署是 first-class citizen」**: 支援多種 database backend、文件超過 20 種格式、OAuth / SAML 等認證方式，以及 Docker Compose 一鍵部署，說明主要目標使用者是企業而非個人開發者。

## 跟其他 RAG 框架比較

| 面向 | RAGFlow | LightRAG | LlamaIndex | LangChain |
|------|---------|----------|------------|-----------|
| 核心抽象 | 元件 DAG | Graph + Vector | Query Engine | Chain / Agent |
| 文件解析 | DeepDoc（自有） | 無 | Data Connectors | 無（靠第三方） |
| 部署 | Docker Compose | pip | pip | pip |
| LLM 抽象 | LiteLLM | 自製 wrapper | 自製 | 自製 |
| UI | React 後台 | 無 | 無 | 無（LangSmith 另計） |
| Agent | DSL Canvas | 無 | Agent runner | AgentExecutor |
| MCP | ✅ 內建 | ❌ | ❌ | ✅ 插件 |
| Memory | Redis + Plugin | 無 | ChatMemoryBuffer | ConversationBufferMemory |
