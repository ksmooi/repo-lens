## 學習對象
- Repo: [langgenius/dify](https://github.com/langgenius/dify)
- Commit: `72ee50c` (2026-05-23)
- 語言 / 主要技術: TypeScript + Python (Flask + Next.js)
- 專案類型 / 類別: agentic (套用模板: _templates/agentic.md)
- 學習深度: standard
- 筆記路徑: `repos/agentic/langgenius__dify/`

## 關鍵入口檔案

| 檔案 | 選擇理由 | 深度精讀指標 |
|---|---|---|
| `api/core/workflow/workflow_entry.py` | 核心入口，graphon DAG 引擎整合層 | 608 行，完整閱讀 |
| `api/core/agent/base_agent_runner.py` | Agent 執行核心，FC/CoT 雙態 agent | 549 行，完整閱讀 |
| `api/core/tools/tool_manager.py` | Tool 系統核心，6 種 provider 類型註冊 | 完整閱讀 get_tool_runtime() |
| `api/core/tools/__base/tool.py` | Tool 四層抽象設計 | 完整閱讀 fork_runtime 模式 |
| `api/core/mcp/mcp_client.py` | MCP 協定整合（SSE + StreamableHTTP + OAuth） | 完整閱讀 |

## 三個最重要的發現

1. **Workflow Layers (洋蔥模型)**: Dify 透過 `GraphEngineLayer` 將配額控制、執行限制、OTel 追蹤、DB 持久化以 middleware 方式堆疊在 graphon DAG 引擎上——這是比單純的 decorator 或 event hooks 更系統化的 cross-cutting concern 處理方式。
2. **Plugin Daemon 隔離架構**: 不直接 import plugin 程式碼，而是透過獨立 HTTP 服務執行，搭配 Backwards Invocation 模式讓 plugin 回調 API 能力。這是 production 等級的安全隔離選擇。
3. **Fork Runtime 執行期隔離**: 每次 tool 調用前 clone 工具實例並注入新 runtime，確保平行調用無 race conditions——簡單但有效的設計模式。

## 候選 Pattern

- **Plugin Backward Invocation**: Plugin 需要反過來呼叫宿主系統的模型/tool/app 能力，形成雙向通訊模式。已在 microsoft/autogen 的 executor 模式觀察到類似設計。
- **Workflow Layer Stack**: 用 middleware layers 處理長期執行工作流的 cross-cutting concerns。值得追蹤是否在其他 workflow-based agent framework 重現。

## 品質檢查摘要
- 各檔字數: README=3,303 / 1-arch=11,512 / 2-walk=11,808 / 3-pat=11,005 / 9-q=4,506
- Mermaid 圖數量: 1-arch=3 張、2-walk=1 張、3-pat=0 張
- path:line 引用總數: 28
- 量化資訊出現次數: 20+（版本號、stars、檔案數、layers 數量、VDB 後端數等）
- 比較表格出現次數: 3（README 競品對照、3-key 跨框架對照、App 類型對照）

## 執行決策日誌
- REPO_SLUG 決定與衝突檢查結果: `langgenius__dify`，無衝突
- 類別確認: CATEGORY=agentic / TEMPLATE=_templates/agentic.md
- 類別決策說明: Dify 的核心是 agentic workflow development platform，agent 編排是其核心抽象，RAG 是功能性而非核心定位
- 新增類別時的附帶動作: 無，使用既有 `agentic` 類別
- 關鍵入口檔案選擇與排除理由: 選擇 workflow_entry.py（graphon 整合層）、agent runner（FC/CoT 雙態）、tool system（4 層抽象）、MCP client（MCP 整合）；排除 RAG pipeline（非 agentic 核心）、frontend web（TypeScript 非分析主力）
- 競品識別: LangFlow / Flowise / Coze（詳見 README 比較表格）
- subagent 使用情況: 使用 4 個 subagent 分別分析 agent 系統、tool/MCP 系統、workflow/app 系統、plugin/RAG 系統
- 工具替換: pygount 超時，改用 find + sed + wc 做語言分布統計
- 模板章節未明確規則的處理: 無
- 任務 prompt 覆寫 AGENTS.md 的項目: STUDY_WORKSPACE_PATH 使用 .tmp/runs/ 而非 .tmp/studies/（任務 prompt 指定路徑優先）
- [UNVERIFIED] 標註的部分清單:
  - graphon 套件的社群規模（無法查到明確的 stars/contributor）
  - Agent Node v2 的 Agent Backend 是否為 gRPC-based microservice
  - Workflow 暫停/恢復的 race condition 處理方式

## Git 身份驗證
- Commit author: ksmooi <kaishianmooi@gmail.com>
- Git config 確認指令輸出:
  `ksmooi <kaishianmooi@gmail.com>`

## 驗證版本說明
此 PR 為 Hermes Agent repo 學習功能，學習對象: langgenius/dify。
