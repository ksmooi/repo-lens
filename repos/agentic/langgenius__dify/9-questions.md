---
repo: langgenius/dify
file: 9-questions
---

# Dify · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼選擇 graphon 而非自幹或 Temporal？**
  - 我目前的推測：[UNVERIFIED] graphon 可能由 Dify 團隊內部開發或 fork，但以 `graphon==0.4.0` 的獨立的 PyPI 套件形式發布。好處是核心引擎和平台邏輯分離，壞處是版本升級風險。
  - 相關程式碼:[`api/pyproject.toml:47`](https://github.com/langgenius/dify/blob/72ee50c/api/pyproject.toml#L47) — `graphon==0.4.0`

- [ ] **Agent Node v2 的 Agent Backend 是什麼？**
  - 我目前的推測：[UNVERIFIED] v2 將 agent 執行抽離到獨立的 agent backend 服務，可能是 gRPC-based microservice 或獨立行程。但從 code 中 `binding_resolver.py` 和 `runtime_request_builder.py` 的設計來看，更像是一個 middleware 抽象層而不只是一個服務 client。
  - 相關程式碼:[`nodes/agent_v2/agent_node.py`](https://github.com/langgenius/dify/blob/72ee50c/api/core/workflow/nodes/agent_v2/agent_node.py) — v2 agent node 的完整實作

- [ ] **Dify 的 multitenancy 是 row-level 還是 schema-level 隔離？**
  - 觀察：所有的 query 中都有 `tenant_id` filter，`TenantIsolatedTaskQueue` 使用 `tenant_id` 作為 queue key。
  - 我目前的推測：row-level isolation。如果是 schema-level，selector/data 載入開銷會被攤提。
  - 相關程式碼:[`services/rag_pipeline/rag_pipeline_task_proxy.py:18`](https://github.com/langgenius/dify/blob/72ee50c/api/services/rag_pipeline/rag_pipeline_task_proxy.py#L18) — `TenantIsolatedTaskQueue`

- [ ] **Workflow 暫停/恢復的狀態一致性如何保證？**
  - Worker 在暫停時可能正在執行節點（如 LLM 呼叫已經送出）。這時暫停指令和工作完成事件之間有 race condition。
  - 我目前的推測：`CommandChannel` 的實作可能使用 Redis pub/sub，節點執行完成後檢查暫停標誌而非中斷正在進行的操作。但沒有看到明確的 atomic 操作保證。
  - 相關程式碼:[`workflow_entry.py:178`](https://github.com/langgenius/dify/blob/72ee50c/api/core/workflow/workflow_entry.py#L178) — `InMemoryChannel` 和 `RedisChannel` 的選擇

- [ ] **Pipeline app 的 RAG Pipeline Variable 機制是如何與 Document 生命週期綁定的？**
  - Pipeline 使用 `RAGPipelineVariable` 並包含 `document_id` / `batch` / `dataset_id`，但 `batch` 的生成邏輯（單次 pipeline run 對應的批次）不清楚。
  - 相關程式碼:[`apps/pipeline/pipeline_runner.py:117`](https://github.com/langgenius/dify/blob/72ee50c/api/core/app/apps/pipeline/pipeline_runner.py#L117) — Pipeline 變數引導流程

## 想問維護者的問題

- 為什麼選擇 Flask 而非 FastAPI？Flask-RESTx 的 maintain 狀態曾經長期停滯，2024 年才恢復。
- Enterprise telemetry 在 README.md 中只提「extra enterprise features」，實際只有 OTel telemetry——企業版的主要功能差異到底是什麼？
- Dify 的 RAG pipeline 和 vector DB backend 是 plugin-based（entry points），但 tool 系統的 built-in provider 是硬編碼在程式碼中的。為什麼 tool provider 不也使用 entry-point 註冊？
- `graphon` 套件的維護狀態和 roadmap 是什麼？

## 下次再看時的待辦

- [ ] 深入研究 `agent_v2/` 的 complete flow：從 `AgentBackendClient` 到 `AgentBackendService` 的完整鏈路
- [ ] 看 `api/services/rag_pipeline/rag_pipeline.py`（~1557 lines）的 pipeline template 管理——這是非常大的一個檔案，可能包含了從 legacy dataset 到 pipeline 的遷移邏輯
- [ ] 對照 LangFlow 的 workflow 編輯器實作——Dify 使用 React Flow / custom graph renderer？
- [ ] 看 MCP OAuth flow 的完整實作（`core/mcp/auth/auth_flow.py:738` lines）——特別是 PKCE + Redis state 的部分
- [ ] 跑 local dev setup，實際建立一個包含 agent node 的 workflow 並觀察 graphon 的事件日誌

## 跨專案對照備忘

- **Plugin Daemon 隔離**：跟 `microsoft/autogen` 的 sandbox executor 類似（但 autogen 是 container-based，Dify 是 process-based）。→ **候選 pattern: plugin security isolation**
- **Workflow Layers 架構**：跟 `langchain-ai/langgraph` 的 checkpoint/middleware 概念類似。→ **候選 pattern: workflow execution layer stack**
- **Backwards Invocation**：跟 `modelcontextprotocol/mcp` 的「reversed call」概念類似。→ **候選 pattern: plugin-backward callback for composable plugins**
