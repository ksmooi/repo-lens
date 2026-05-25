---
repo: langgenius/dify
type: agentic
studied_at: 2026-05-24
commit_sha: 72ee50c
language: TypeScript / Python
framework: graphon (自研 DAG 引擎)
agent_style: single-agent + workflow-based orchestration
stars: 142k
status: active
---

# Dify · 概覽

Production-ready 的 LLM 應用開發平台，把 agentic workflow 的開發從「寫程式」變成「拖拖拉拉配置」。

## 解決什麼問題

LLM 應用開發有一塊「中部地帶」——說簡單不簡單（要串 LLM、RAG、tools、workflow），但說困難也不是在寫 framework 核心。Dify 想佔的就是這塊地：讓開發者用**視覺化 workflow 編輯器**組合 LLM、RAG pipeline、tools 和 agent 行為，同時保留 API 層（Backend-as-a-Service）給前端整合。

## 為什麼值得研究

- **graphon DAG 引擎**：Dify 沒有重新發明 workflow 輪子，而是採用 `graphon==0.4.0` 這個外部 DAG 引擎，自己只寫整合層（`WorkflowEntry`）。這個取捨在 142k stars 的專案中很少見，多數同級專案會選擇自幹。
- **生產級 MCP 整合**：完整實作了 MCP 2025-06-18 協議（SSE + StreamableHTTP + OAuth PKCE），是所有同類平台中最早且最完整支援 MCP 的之一。
- **Plugin 隔離架構**：透過獨立的 Plugin Daemon 行程隔離第三方 plugin 程式碼，不走常見的 import-based plugin loading——這是 production 環境才會有意識去處理的問題。
- **Agent 雙態策略**：Function Calling 優先、Chain-of-Thought 備援，由 model schema 的 `MULTI_TOOL_CALL` feature flag 自動切換，無需使用者手動決定。

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | Workflow-based（agent 是 workflow 中的一個節點類型） |
| 數量 | single-agent per workflow node |
| 編排方式 | declarative（graphon DAG） |
| Memory | session-based（TokenBufferMemory） |
| Tool calling | 雙軌：LLM native function calling（FC） / ReAct + JSON（CoT） |

## 技術棧一句話

`TypeScript (Next.js)` + `Python (Flask)` + `graphon (DAG)` + `Celery` + `30+ vector DB backends` + `Plugin Daemon`

## 健康度信號

- ⭐ Stars: ~142k
- 📅 最後 commit: 2026-05-23（學習當天仍有活動）
- 👥 主要維護者: langgenius 團隊（~30-50 活躍貢獻者）
- 🔄 commit 頻率: 每天 10-30 個 commits，極度活躍

## 我會在後續筆記中回答的問題

- Dify 的 workflow 引擎 (graphon) 跟 LangGraph / Temporal 的比較？
- Agent 節點 v2 相比 v1 的設計改進是什麼？
- 為什麼選擇 Plugin Daemon 隔離而非 sandboxed import？
- Pipeline app 類型跟傳統 workflow 在設計上的區別？
- 28+ VDB 後端的抽象層做得好嗎？trade-off 是什麼？

## 競品比較

| 面向 | Dify | LangFlow | Flowise | Coze |
|---|---|---|---|---|
| 主要抽象 | Workflow DAG (graphon) | LangGraph-based graph | LangChain-based flow | Closed ecosystem |
| Agent 類型 | FC / CoT 自動切換 | LangGraph agent | ReAct agent | 黑盒 |
| Plugin 機制 | Plugin Daemon（隔離行程） | 無 | 無 | 封閉 |
| MCP 支援 | 完整（SSE + StreamableHTTP + OAuth） | 無 | 部分 | 封閉 |
| 部署彈性 | Cloud + Self-hosted + Enterprise | Self-hosted | Self-hosted | Cloud only |
| 社群規模 | 142k ⭐ | 45k ⭐ | 45k ⭐ | N/A |
