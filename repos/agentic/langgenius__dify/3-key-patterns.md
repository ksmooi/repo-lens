---
repo: langgenius/dify
file: 3-key-patterns
---

# Dify · 值得偷學的設計

## Pattern 1: Plugin Daemon 隔離架構

**是什麼**: Dify 不直接用 Python import 載入 plugin，而是將 plugin 執行在獨立的 HTTP 服務（Plugin Daemon）中，API 透過 HTTP 與之通訊。

**為什麼有效**: 第三方 plugin 可能包含不安全的程式碼、無限迴圈、或大量記憶體使用。如果 plugin 直接 import 進 API 行程：
- plugin crash → 整個 API crash
- plugin 記憶體洩漏 → 影響所有請求
- plugin 無限迴圈 → 阻塞 event loop

Plugin Daemon 隔離後，API 行程保持穩定，plugin 的問題被限制在 daemon 中。同時 daemon 可以獨立伸縮——可以給不信任的 plugin 分配獨立的 daemon 實例。

**程式碼位置**:
- `api/core/plugin/impl/base.py:62` — `BasePluginClient`，所有 plugin 客戶端的共同 HTTP 通訊層
- `api/core/plugin/impl/tool.py:17` — `PluginToolManager`，透過 daemon 管理 tool providers
- `api/core/plugin/backwards_invocation/` — 反向調用模式（plugin 反過來呼叫 API 的能力）

**何時可以借用**:
- 當你的系統需要讓使用者撰寫或安裝第三方程式碼
- 當 plugin 的穩定性不可預期（特別是社群貢獻的 plugin）
- 當你需要以 service-level 監控 plugin 行為（可以對 daemon 施加獨立 rate limit）

**注意事項**:
- 多了一次 HTTP round-trip：plugin 調用 latency 會增加（Dify 有 HTTP connection pooling，~50 keepalive connections）
- 需要實作 Backwards Invocation 通道：plugin 需要反過來呼叫 API 的 model / tool / app 能力
- 部署複雜度增加（多一個 daemon 要管、版本要同步）

**替代方案**: LangFlow 和 Flowise 採用直接 import 的方式，簡單但缺乏隔離。
**不適用**: plugin 邏輯非常輕量且不涉及外部系統呼叫時，隔離的成本超過效益。

---

## Pattern 2: Layers（洋蔥模型）Middlewar

**是什麼**: 基於 graphon 引擎的 `GraphEngineLayer` 機制，將 cross-cutting concerns（配額、限制、觀測、持久化）以「洋蔥」方式堆疊在 workflow 執行引擎上。

**為什麼有效**: 傳統 middleware 通常是 request-scoped（一個請求經過 middleware chain），但 Dify 的 layer 是 **workflow-scoped** 且 **stateful**——它們貫穿整個 workflow 的生命週期。`LLMQuotaLayer` 需要跨節點累積 LLM 調用計數，`PersistenceLayer` 需要追蹤每個節點是否已持久化（避免重複寫入）。

程式碼層次：
```
── ExecutionLimitsLayer    最外層：先檢查全域限制
── LLMQuotaLayer           第二層：LLM 配額檢查
── ObservabilityLayer      第三層：OTel 追蹤
── PersistenceLayer         最內層：DB 寫入
```

**程式碼位置**:
- `workflow_entry.py:200-220` — layers 堆疊
- `apps/workflow/layers/llm_quota.py` — LLM 配額 layer（跨節點累積計數）
- `apps/workflow/layers/observability.py` — OTel 追蹤 layer（OpenTelemetry spans）
- `apps/workflow/layers/persistence.py` — Persistence layer（workflow run / node execution 寫入 DB）

**何時可以借用**:
- 當你有一個長時間執行的處理流程（非單純 request），需要跨步驟累積狀態
- 當不同的 cross-cutting 邏輯有嚴格的執行順序要求（配額檢查必須在實際執行前）
- 當你想在不修改核心引擎的前提下加入觀測性

**注意事項**:
- 每個 layer 都增加不可忽略的 overhead（特別是 persistence layer 每次節點執行都寫 DB）
- Layers 的執行順序必須精確——如果 `ObservabilityLayer` 在 `LLMQuotaLayer` 之外，failure span 會先於配額錯誤被記錄，產生誤導性的 traces
- 當 layer 數量增加到 5+ 時，除錯難度顯著增加（trace 裡看到的是 layer chain 而不是直接的執行流）

**替代方案**: 傳統 decorator 模式（裝飾每個 node 類別）、event hooks（node 執行前/後觸發 callback）。

---

## Pattern 3: Fork Runtime —— Tool 執行期狀態隔離

**是什麼**: 每次 tool 調用前，透過 `Tool.fork_tool_runtime()` 克隆 tool 實例並注入新的 `ToolRuntime`，確保工具之間的執行期狀態（credentials、runtime_parameters）完全隔離。

**為什麼有效**: 一個 tool provider 可能管理多個 tool 實例（例如 `webscraper` provider 提供 `webscraper` tool），這些工具共享同一個 provider 設定但有不同的執行期參數。如果修改 runtime 會影響原始 tool 實例，平行調用時就會出現 race conditions。

```python
# __base/tool.py:29
def fork_tool_runtime(self, runtime: ToolRuntime) -> Tool:
    # 使用 deep-copy 或新實例來隔離狀態
    new_tool = copy(self)
    new_tool.runtime = runtime
    return new_tool
```

**程式碼位置**:
- `api/core/tools/__base/tool.py:29` — `fork_tool_runtime()` 定義
- `api/core/tools/__base/tool_runtime.py:10` — `ToolRuntime` Pydantic model
- `api/core/tools/tool_engine.py:42` — `ToolEngine` 調用路徑的 fork 使用

**何時可以借用**:
- 當一個 factory/mananger 物件需要為每次呼叫提供隔離的執行期上下文
- 當工具/處理函式的 credentials 或 parameters 在每次呼叫時會改變
- 特別適合 parallel execution 場景

**注意事項**:
- Fork 的實現要是 shallow copy（deep copy 太貴），但 mutable fields 要特別處理
- 要確保 fork 後的 tool 可以正確 garbage collected——如果持有大物件（HTTP connection pool 等），fork 可能產生多份連線

**不適用**: stateless tool（相同的 credentials、相同的參數）、執行期無副作用的工具。

---

## Pattern 4: Agent 雙態自動切換（FC → CoT）

**是什麼**: Dify 不讓使用者手動選擇 agent 策略，而是由 [`app_runner.py:185-186`](https://github.com/langgenius/dify/blob/72ee50c/api/core/app/apps/agent_chat/app_runner.py#L185) 自動判斷模型是否支援 tool calling，動態切換 Function Call Agent 或 Chain-of-Thought Agent。

**為什麼有效**: LLM 生態持續演進（2023 年 CoT 是主流，2024 年 FC 變成標準），但使用者不應該每年重寫他們的 agent 配置。將策略選擇與 LLM model schema 綁定，讓使用者切換模型時自動獲得最適合的 agent 行為。

```python
# app_runner.py:185-186
features = model_schema.features or []
if {ModelFeature.MULTI_TOOL_CALL, ModelFeature.TOOL_CALL}.intersection(features):
    agent_entity.strategy = AgentEntity.Strategy.FUNCTION_CALLING
```

**程式碼位置**:
- `api/core/app/apps/agent_chat/app_runner.py:185` — 策略自動切換點
- `api/core/agent/fc_agent_runner.py:77` — Function Calling agent loop（LLM native tool calling）
- `api/core/agent/cot_agent_runner.py:103` — Chain-of-Thought agent loop（ReAct JSON）

**何時可以借用**:
- 你的平台需要支援多種 LLM providers 且不要求使用者了解模型能力差異
- 你的 agent 實作基於 LLM 的 tool calling，但需要優雅的 fallback

**注意事項**:
- FC 和 CoT 的 prompt template 完全不同——這意味著使用者在 system prompt 中的 tool 描述需要同時服務兩種模式
- 自動切換的臨界點由 model schema 的 `features` 欄位決定——如果某個 provider 的 schema 不準確，切換行為會出錯
- 使用者無法強制使用 CoT 模式（除非寫自訂 plugin strategy）

---

## Pattern 5: Backwards Invocation —— Plugin 回調 API 能力

**是什麼**: Plugin 執行時需要反過來使用 Dify API 的能力（如呼叫 LLM、執行 workflow node、存取 tool）。Dify 透過 `backwards_invocation/` 模組提供這個通道，讓 Plugin Daemon 可以在執行 plugin 時發起 HTTP 請求回 Dify API。

**為什麼有效**: Plugin 不僅僅是「提供外部功能」，它需要：
- 呼叫模型（plugin 的 tool 可能需要 LLM 做 intermediate reasoning）
- 執行 workflow node（plugin 可能需要 parameter extraction）
- 調用其他工具（plugin 可能 composable）

直接在 daemon 側提供這些能力會讓 daemon 變得臃腫且不安全。讓 plugin **回撥** API 是更清晰的責任分工。

**程式碼位置**:
- `api/core/plugin/backwards_invocation/base.py:21` — `BaseBackwardsInvocationResponse`（泛型）
- `api/core/plugin/backwards_invocation/tool.py:12` — `PluginToolBackwardsInvocation.invoke()`: 反向調用 `ToolEngine.generic_invoke()`
- `api/core/plugin/backwards_invocation/model.py:38` — `PluginModelBackwardsInvocation`: 反向調用 LLM / Embedding / Rerank / TTS
- `api/core/plugin/backwards_invocation/app.py:23` — `PluginAppBackwardsInvocation`: 反向調用 Dify App
- `api/core/plugin/backwards_invocation/node.py:15` — `PluginNodeBackwardsInvocation`: 反向調用 ParameterExtractor / QuestionClassifier

**何時可以借用**:
- 當 plugin 不僅是純粹的外部函數，還需要回調宿主系統的能力
- 當你希望將 plugin 的安全邊界設在 daemon 層級，並確保所有敏感操作（LLM 調用、檔案存取）仍由主 API 控制

**注意事項**:
- 所有 backwards invocation 的回應都透過 `convert_to_event_stream()` 統一卷成 JSON event stream——確保統一的序列化格式
- 安全性：backwards invocation 請求必須在可信通道中傳輸（Dify 透過 daemon 間的內部網路），不能暴露給外部
- Latency：每次 plugin 需要 LLM 時，要走「API → daemon → plugin → daemon → API → LLM → API → daemon → plugin」這麼長的路徑

---

## Agent 設計的哲學觀察

Dify 的 agent 設計反映了「platform 思維」：

1. **讓使用者不需要知道 agent 內部** — 自動切換 FC/CoT、wrapper 讓 tool calling 在兩種模式下一致、DB-centric 狀態管理讓使用者可以暫停/恢復 dialogue
2. **production 優先於靈活** — Plugin Daemon 隔離、backwards invocation、多層 layers——這些都是 production 環境才會在意的問題
3. **外部引擎 > 自幹** — graphon 做 workflow、外部 LLM provider 做推論——Dify 不試圖 reinvent，而是做整合與 UI 層的價值
4. **MCP 是下個 stage** — 完整的 MCP 協議支援（SSE + StreamableHTTP + OAuth）顯示 Dify 對 Agent-to-Agent 通訊的重視，而不是停留在單一 agent 框架的競爭

## 跟其他 agent 框架比較

| 面向 | Dify | LangGraph | AutoGen |
|---|---|---|---|
| Agent 抽象層級 | Platform（UI + API + workflow） | Framework（低階 graph primitives） | Framework（multi-agent 編排） |
| 執行引擎 | graphon DAG（外部） | LangGraph（內部圖引擎） | Autogen（內部 agent runtime） |
| Plugin 安全 | Plugin Daemon 隔離 | 不支援 plugin | 不支援 plugin |
| 預設 agent 風格 | Workflow-based single agent | Graph state machine | Conversational multi-agent |
| 主要 trade-off | 靈活性低（被 UI 框架限制）vs 易用性高 | 靈活性高 vs 入門門檻高 | Multi-agent 是第一等 vs 單 agent 場景過度設計 |
