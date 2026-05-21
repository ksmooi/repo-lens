## 學習對象
- Repo: [nousresearch/hermes-agent](https://github.com/nousresearch/hermes-agent)
- Commit: `48be2e0` (2026-05-21)
- 語言 / 主要技術: Python, OpenAI SDK, SQLite
- 專案類型 / 類別: agentic (套用模板: agentic.md)
- 學習深度: standard
- 筆記路徑: `repos/agentic/nousresearch__hermes-agent/`

## 關鍵入口檔案
| 檔案 | 選擇理由 | 深度精讀指標 |
|---|---|---|
| `run_agent.py` | AIAgent 類別，agent 的 public API 與初始化 (~4153 lines) | 60+ parameters, provider auto-detection, iteration budget |
| `agent/conversation_loop.py` | Agent 主迴圈的完整實作 (~3900 lines) | 同步 while loop, 7-stage empty recovery, streaming always-on |
| `tools/registry.py` | 工具註冊與 AST-based discovery 核心 | Self-registering, `__slots__`, generation counter |
| `model_tools.py` | Tool orchestration layer (~923 lines) | get_tool_definitions, handle_function_call, async bridging |
| `hermes_state.py` | SQLite 狀態儲存 (~3273 lines) | WAL mode, FTS5, schema versioning, session splitting |

## 三個最重要的發現
1. **AST-based Tool Discovery** — 不用中央註冊表，用 AST 掃描 `tools/*.py` 找出有 `registry.register()` 的 module。每次在 `tools/` 下新增檔案 = 註冊一個新工具，零 boilerplate。
2. **三層 System Prompt 快取** — stable/context/volatile 分層，stable 層在 session 內完全不變，維持 Anthropic prefix cache 高命中率。Date-only timestamp (YYYY-MM-DD) 確保同一天的 prompt byte 完全一致。
3. **Implicit tool execution fallback** — 從 credential pool 輪換到 provider fallback chain，每個層級都有精細的 error classification 和 recovery 策略（不是單純 retry 或 abort）。

## 候選 Pattern
- **Registry generation counter**（自動 cache 失效）：`tools/registry.py` 的 `_generation` counter 讓所有 cache 消費者在 tool 註冊變更時自動失效。已在 `paper_lens` 看到類似版本，但需要 3+ repo 才能收錄到 `_patterns/`。
- **Imperative while loop vs State machine**：Hermes Agent 的同步 while loop 與 `hkuds/nanobot` 的顯式狀態機形成對照，可用於補充 `_patterns/agent-state-machine.md`。

## 品質檢查摘要
- 各檔大小：README=3.8KB / 1-arch=17.5KB / 2-walk=11.7KB / 3-pat=18.6KB / 9-q=6.7KB
- Mermaid 圖數量：1-arch=3 張、2-walk=1 張、3-pat=0 張
- path:line 引用總數：38（commit-pinned 至 48be2e0）
- 比較表格次數：2（README 競品對照 + 3-key-patterns 框架比較）

## 執行決策日誌
- REPO_SLUG 決定: `nousresearch__hermes-agent`（無衝突）
- 類別確認: CATEGORY=agentic / TEMPLATE=agentic.md（AI agent 框架，非 RAG）
- 類別決策說明: Hermes Agent 是純 agent 產品（非 RAG pipeline），歸 agentic 類別
- 新增類別時的附帶動作: 無，使用既有類別
- 關鍵入口檔案選擇與排除理由: 排除 gateway/run.py（18K+ lines）因其主要為 platform adapter 而非常見 agent loop；排除 cli.py（676KB）因其為 CLI UI 層而非 agent 核心
- 競品識別: Claude Code (closed source), Codex CLI (closed source), LangGraph (open source, graph-based)
- subagent 使用情況: 使用 3 個平行 subagent 分析 (a) agent loop/control flow (b) tool/skills system (c) memory/state/gateway
- 工具替換: 無
- [UNVERIFIED] 標註: 9-questions.md 中有 3 處 [UNVERIFIED] 標記

## Git 身份驗證
- Commit author: ksmooi <kaishianmooi@gmail.com>

## 驗證版本說明
此 PR 為 Hermes Agent repo 學習功能，學習對象: nousresearch/hermes-agent。
