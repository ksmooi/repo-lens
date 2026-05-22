---
repo: mlflow/mlflow
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 1c491e7
---

# MLflow · 值得偷學的設計

## Pattern 1: Store 抽象層（Strategy Pattern for Backend）

**是什麼**: MLflow 用統一的 `AbstractStore` 介面封裝所有 tracking 操作，然後提供 FileStore、SQLAlchemyStore、RestStore、DatabricksRestStore 四種實作。使用者不需知道背後的儲存引擎。

**為什麼有效**: MLflow 的演進路徑（file → SQL → REST → Databricks）是反覆發生的架構需求。AbstractStore 的設計讓每個新 backend 只需要實作固定的方法集合，而不需要動到上層的 handler 或 client 程式碼。

**程式碼位置**:
- AbstractStore 定義: [`mlflow/store/tracking/abstract_store.py:57-1724`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/store/tracking/abstract_store.py#L57)
- FileStore: [`mlflow/store/tracking/file_store.py:1-2913`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/store/tracking/file_store.py#L1)
- SQLAlchemyStore: [`mlflow/store/tracking/sqlalchemy_store.py:1-8720`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/store/tracking/sqlalchemy_store.py#L1)
- RestStore: [`mlflow/store/tracking/rest_store.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/store/tracking/rest_store.py)
- Store 選擇邏輯: [`mlflow/tracking/_tracking_service/utils.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/tracking/_tracking_service/utils.py)

**替代方案**:
- **Plugin hooks（如 PyTorch's dispatch）**: 更靈活，但需要額外的 entry point 註冊機制
- **Configuration-driven factory**: MLflow 也用了這個（`_get_store()` 根據 URI 選擇實作），但 AbstractStore 的介面比 factory 更適合需要多種一致實作的場景
- **Mixin 多層繼承**: `GatewayStoreMixin` 用多重繼承為 AbstractStore 添加 gateway 相關方法，避免修改基底類別

**何時可以借用**:
- 你的系統需要支援多種 backend（local file / SQL / cloud API）
- 你預期未來會新增新的 backend 類型
- 你希望 client 端程式碼與儲存邏輯解耦

**注意事項**:
- 抽象介面會隨時間膨脹——AbstractStore 已經大到 1,724 行，包含太多方法。較新的方法（如 `log_traces`）透過 `GatewayStoreMixin` 而非直接加在基底類別，這是明智的 restraint。
- 介面的 granularity：AbstractStore 的方法以 CRUD 為主，而非以業務流程為主。這讓實作更簡單，但可能導致多個方法總是被一起呼叫（例如 `log_batch` 其實是 log_metrics + log_params + log_tags 的批次版本）。

---

## Pattern 2: Autolog + Safe Patch（攔截式 Instrumentation）

**是什麼**: MLflow 不要求框架主動整合 tracing/logging，而是透過 `safe_patch` 在 runtime 替換目標方法，在呼叫前後注入 logging/tracing 邏輯。

**為什麼有效**: 這是「攔截」（interception）而非「整合」（integration）的策略。攔截的優點：**對 zero-code-change 的使用者完全透明**，不需要每個 framework 都寫 plugin。20+ framework 的支援靠的是同一套 `safe_patch` 機制。

**程式碼位置**:
- safe_patch 核心: [`mlflow/utils/autologging_utils/safety.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/utils/autologging_utils/safety.py)
- autologging 基礎設施: [`mlflow/utils/autologging_utils/__init__.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/utils/autologging_utils/__init__.py)
- 使用範例（OpenAI）: [`mlflow/openai/autolog.py:36-140`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/openai/autolog.py#L36-L140)

**替代方案**:
- **Framework-native hook（如 LangChain callback）**: 更乾淨，不會污染 stack trace，但每個 framework 的 hook 系統不同，維護成本高
- **AST transformation**: 更強大（可修改原始碼結構），但複雜度高、容易出錯
- **Middleware（如 WSGI middleware）**: 只適用於 Web framework，不適用於 library method

**何時可以借用**:
- 需要為多個不相關的 library 添加統一的 logging/tracing/monitoring layer
- 使用者不願意或無法修改原始呼叫程式碼
- 你的攔截邏輯「不影響」原始行為（read-only 或 fire-and-forget）

**注意事項**:
- Monkey-patch 在 multithreading 環境下有 race condition——多個 thread 同時 patch 同一個 class method 時可能互相覆蓋。MLflow 的 `safe_patch` 用 `__wrapped__` 追蹤原始方法來減緩，但無法完全解決。
- 如果目標 framework 在 patch 之後重新載入 module（某些 hot-reload 情境），patch 會丟失。
- Stack trace 多一層 wrapper，debug 時可能混淆。

---

## Pattern 3: OpenTelemetry Wrapper（不重新發明輪子）

**是什麼**: MLflow 的 tracing 系統建立在 OpenTelemetry 之上，而不是從頭實作 span/trace 模型。MLflow 的 span 實際上是 OTel span 的包裝。

**為什麼有效**: 選擇 OTel 作為底層有三個好處：(1) 生態系相容——export 到任何 OTel-compatible backend，(2) 不用自己維護 span 的 propagation、export、sampling 等複雜機制，(3) OTel 的 context propagation 已經是業界標準，跨語言跨服務的 tracing 自然獲得。

**程式碼位置**:
- OTel provider 封裝: [`mlflow/tracing/provider.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/tracing/provider.py)
- Span 包裝（LiveSpan）: [`mlflow/entities/span.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/entities/span.py)
- InMemoryTraceManager: [`mlflow/tracing/trace_manager.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/tracing/trace_manager.py)

**替代方案**:
- **自創 tracing 格式**（如 LangFuse）: 完全掌控，但失去 OTel 生態系整合
- **直接使用 OTel API（不包裝）**: 更輕量，但 MLflow 需要 span 在記憶體中可變（OTel 的 span 是 immutable/one-shot 的），所以包裝是必要的

**MLflow 的關鍵修改**:
- `InMemoryTraceManager` 維護 mutable span dict，支援即時查詢（UI 不用等 export 完成）
- `LiveSpan` 包裝 OTel span，加入 `inputs`、`outputs` 等 MLflow-specific 屬性
- 串流模式下，chunk 事件透過 `SpanEvent` 記錄而非 blocking wait

**何時可以借用**:
- 你的系統需要 tracing，但不想被特定 backend 綁住（vendor lock-in）
- 你需要 OTel 的 context propagation，但需要自訂 span 屬性
- 你的使用者已經有 OTel collector 部署（可以 reuse）

**注意事項**:
- MLflow 的包裝層增加了一層間接——debug 時需要同時理解 OTel 和 MLflow 的 span 狀態
- OTel Python SDK 的 overhead（特別是 `tracer.start_span()`）在極高 throughput 下可能成為瓶頸

---

## Pattern 4: Protobuf-driven API 一致性

**是什麼**: MLflow 的 REST API 合約（contract）由 `.proto` 檔案定義，所有 handler 和 client stub 從 protobuf code gen 產生。

**為什麼有效**: 這保證了 client 和 server 對 API 的理解完全一致——沒有文件與實作不同步的問題。Python、R、Java 三個語言的 SDK 共用同一組 `.proto` 定義，不可能出現某個語言漏接欄位。

**程式碼位置**:
- API contract 定義: [`mlflow/protos/service.proto`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/protos/service.proto)
- Databricks 相關: [`mlflow/protos/databricks.proto`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/protos/databricks.proto)
- Model Registry: [`mlflow/protos/model_registry.proto`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/protos/model_registry.proto)

**替代方案**:
- **OpenAPI/Swagger**: 更廣泛被支援，有更好的工具生態系（code gen、UI explorer）。MLflow 選擇 protobuf 主要是因為 Databricks 的 legacy。
- **手寫 Flask route**: 靈活性最高，但沒有 contract guarantee，容易 client-server 不同步

**MLflow 的 mix**：
有趣的是，MLflow 並沒有 100% protobuf。較新的功能（AI Gateway、tracing endpoints）也接受 JSON body，透過 FastAPI 的 Pydantic schema 而非 protobuf 驗證。這反映了團隊從純 protobuf 轉向混合模式的演進。

**何時可以借用**:
- 你的 API 需要多語言 SDK 支援
- API 變更頻繁，需要防止 client-server 不同步
- 團隊習慣 contract-first 開發流程

**注意事項**:
- protobuf 的 JSON 序列化在某些語言（Python）的 proto3 中有 field presence 問題——optional field 和 zero value 難以區分
- 新增 endpoint 需要修改 proto → code gen → handler → client 共四層，迭代速度較慢
- protobuf 在 Web 生態系較弱（需要 `grpc-web` 或 JSON 轉接）

---

## Pattern 5: Gateway Provider Registry（Plugin 架構）

**是什麼**: AI Gateway 的 provider 透過 entry point 註冊機制，支援 pluggable LLM provider。provider 只需要實作固定的介面就可以被 MLflow Gateway 使用。

**為什麼有效**: LLM provider 的數量快速增長（Anthropic、Bedrock、Gemini、Groq、Mistral…），硬編碼每種 provider 的 API 邏輯在 Gateway 中不可行。Plugin 架構讓 provider 可以獨立開發和部署。

**程式碼位置**:
- Provider registry: [`mlflow/gateway/provider_registry.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/gateway/provider_registry.py)
- Provider 實作（範例）: [`mlflow/gateway/providers/`](https://github.com/mlflow/mlflow/tree/1c491e7/mlflow/gateway/providers)

**替代方案**:
- **單一 provider 抽象 + switch/case**: 簡單但無法擴展——每個新 provider 都需要改核心程式碼
- **Class-based registry（如 Django ORM）**: MLflow 實際上也用了這個模式——每個 provider 是一個 class，registry 負責 lookup

**何時可以借用**:
- 你的系統需要支援多個外部服務，且服務數量會持續增加
- 第三方開發者需要能夠貢獻新 provider 而不改核心
- 每個 provider 的 lifecycle 管理不同（初始化、rate limit、retry）

**注意事項**:
- Plugin 架構需要定義穩定的 provider 介面——如果介面經常變動，每個 provider 都要跟著改
- MLflow 的 provider registry 使用 Python entry points，這意味著 provider 必須安裝為獨立的 Python package——對小規模部署來說有點重

---

## ML 工程品味的觀察

1. **對 abstraction 的態度務實**：MLflow 不是純粹的 abstraction 愛好者。核心層（store、entities）有清晰的抽象，但工具類（utils/）非常扁平和直白。這反映了團隊「在必要的地方抽象，其他地方直接寫」的態度。

2. **Legacy 與新功能共存的智慧**：FileStore（2018）和 tracing（2025）在同一個 codebase 裡和平共存。新功能透過新的 proto 檔案和 handler 加入，不破壞既有 API。`GatewayStoreMixin` 是個好的範例：用 mixin 為基底類別添加新功能，而非修改基底類別。

3. **測試規模的啟示**：1,015 個測試檔案對應 1,109 個 source 檔案——幾乎是 1:1 的比例。這在 OSS ML 專案中相當罕見（許多 ML 專案的測試覆蓋率低很多）。測試按照 `tests/store/`、`tests/tracking/`、`tests/server/` 等結構組織，與 source 結構對應。

4. **串流處理的設計**：OpenAI streaming response 的處理方式值得注意——不是等完整 response 再記錄，而是逐 chunk 記錄為 SpanEvent。這讓使用者可以看到 streaming response 的即時進度，但增加了 trace 存的資料量。
