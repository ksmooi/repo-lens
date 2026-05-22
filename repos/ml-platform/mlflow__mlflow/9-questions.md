---
repo: mlflow/mlflow
file: 9-questions
studied_at: 2026-05-22
commit_sha: 1c491e7
---

# MLflow · 未解問題

## 還沒搞懂的設計決策

- [ ] **FileStore vs SQLAlchemyStore 的並行限制**
  - 問題: FileStore 使用 file lock 做並行控制，在什麼 threshold 下會開始出現效能問題？MLflow documentation 建議 production 用 SQLAlchemyStore，但沒有明確的量化邊界。
  - 相關程式碼: [`mlflow/store/tracking/file_store.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/store/tracking/file_store.py#L1)
  - 我的推測: [UNVERIFIED] FileLock 在 <50 concurrent writes 應該還好，超過後 contention 會急劇上升。但 MLflow 的實驗追蹤通常是低頻率寫入（一個 training run 可能每分鐘 log 一次），所以實際影響可能不大。

- [ ] **為什麼 tracing 同時走 SQLAlchemyStore 和 InMemoryTraceManager？**
  - 問題: trace 資料同時存在記憶體（InMemoryTraceManager 的 dict）和資料庫（SQLAlchemyStore），兩者的同步策略是什麼？如果 server restart，in-memory trace 應該會丟失。
  - 相關程式碼: [`mlflow/tracing/trace_manager.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/tracing/trace_manager.py)
  - 我的推測: [UNVERIFIED] InMemoryTraceManager 應該是 ephemeral cache——只保留 server 啟動後的 trace，重啟後透過資料庫查詢重建。但 review 程式碼時沒有看到明確的 warm-up 邏輯。

- [ ] **Flask + FastAPI 雙 server 的整合策略**
  - 問題: 為什麼新功能（AI Gateway）選 FastAPI 而非 Flask？兩者之間如何 routing？是共用 port 還是不同 port？
  - 相關程式碼: [`mlflow/server/__init__.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/server/__init__.py)、[`mlflow/gateway/app.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/gateway/app.py)
  - [UNVERIFIED] 從 dependency 來看，FastAPI 被列在 core dependencies（非 optional），可能 Flask 的 handler 和 FastAPI 的 gateway route 在相同 process 中共用 port。需要確認 ASGI 的整合方式。

- [ ] **Protobuf 與 Pydantic 的界線**
  - 問題: 新功能（tracing、gateway）似乎用 Pydantic 而非 protobuf。團隊的決策標準是什麼？何時選 protobuf、何時選 Pydantic？
  - 相關程式碼: [`mlflow/protos/service.proto`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/protos/service.proto) vs [`mlflow/gateway/base_models.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/gateway/base_models.py)
  - [UNVERIFIED] 推測: 需要多語言 SDK 支援的功能用 protobuf（tracking、model registry），純 Python 的新功能用 Pydantic。但 tracing 需要對應的 R/Java SDK 嗎？如果不需要，Pydantic 的開發效率較高。

- [ ] **`mlflow-skinny` 和 `mlflow-tracing` 的切分策略**
  - 問題: MLflow 現在是三個 package：`mlflow`（完整版）、`mlflow-skinny`（最小依賴）、`mlflow-tracing`（tracing only）。這三者的版本如何管理？相依關係如何？
  - 相關程式碼: [`mlflow/version.py`](https://github.com/mlflow/mlflow/blob/1c491e7/mlflow/version.py)——`IS_TRACING_SDK_ONLY` flag 控制 import 路徑
  - [UNVERIFIED] 推測: `mlflow-tracing` 應該是 `mlflow` 的子集，提供僅 tracing 功能。`mlflow-skinny` 是更底層的 subset，不含 pandas/server/ml framework 整合。版本應該共用同一個 base version。

## 想問維護者的問題

- [ ] **你們如何測試 `safe_patch` 的正確性？** Monkey-patch 20+ framework 的關鍵方法，每個 framework 的版本演進都可能 break patch。你們有自動化測試驗證 patch 在最新版 framework 上仍然有效嗎？
- [ ] **AI Gateway 的 provider 數量預計如何管理？** 現在已有 10+ provider，每個 provider 的 API 風格不同（streaming、function calling、tool use）。你們如何保證 provider 之間的行為一致性？
- [ ] **Trace 的儲存成本**：每個 LLM call 都可能產生 kB 級的 trace 資料，長期累積的 storage cost 如何管理？`TraceArchivalService` 的壓縮和保留策略是什麼？

## 下次再看時的待辦

- [ ] 實際跑一次 `mlflow server`，檢視 Flask + FastAPI 的整合起動流程
- [ ] 對照 LangFuse 的 tracing 實作，比較兩者的 span 模型差異
- [ ] 深入 `safe_patch` 的 exception handling 機制——它如何保證原始方法永遠被呼叫？
- [ ] 查閱 `mlflow/protos/service.proto` 了解完整 API surface
- [ ] 閱讀 `mlflow/store/db_migrations/` 了解 alembic migration 策略

## 跨專案對照備忘

- **可插拔 Storage Backend 抽象層**
  - MLflow: `AbstractStore` → FileStore / SQLAlchemyStore / RestStore / DatabricksRestStore（class-based 介面）
  - LightRAG: `BaseKVStorage` → JSON / Redis / Milvus / Neo4j 等（factory-based registry）
  - **評估**：已在 2 個獨立 repo（LightRAG、MLflow）觀察到相同模式。尚未滿足 3 個 repo 的收錄門檻，但值得追蹤下一個 repo 是否也使用類似的 storage backend abstraction。
  - 兩個 repo 的差異：MLflow 使用 class 繼承（`AbstractStore` + 子類別），LightRAG 使用 factory + registry dict。前者在介面穩定時更好維護，後者在動態載入時更靈活。

- **LLM Provider 抽象層**
  - MLflow: `mlflow/gateway/provider_registry.py` 的 provider registry + pluggable provider 實作
  - 已收錄在 `_patterns/llm-provider-abstraction.md`（含 MLflow 在內已有 4+ repo 觀察到此模式）
  - MLflow Gateway 的實作特點：結合 rate limiting、budget control、guardrails 等治理功能，不只是 provider 抽象。

- **`safe_patch` monkey-patch 模式**：類似 OpenTelemetry 的 `instrument()` 模式（`opentelemetry-instrumentation-requests` 等），但 MLflow 的 patch 有更好的 exception safety。→ 目前只在這兩個專案觀察到，尚未足夠。
