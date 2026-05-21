---
repo: hkuds/nanobot
file: 00-overview
studied_at: 2026-05-21
commit_sha: ccbc0bb
type: agentic
language: Python
framework: 自製 MessageBus + TurnState state machine
agent_style: single-agent (支援 subagent 協作)
stars: 42.9k
status: active
---

# Nanobot · 概覽

Nanobot 是一個極輕量級的開源 AI agent 框架，由香港大學資料科學團隊（HKUDS）開發。它跟 Hermes Agent、Claude Code、Codex CLI 同屬一個品類：讓 LLM 有能力執行工具、管理對話記憶、跨聊天平台運作的個人助手框架。核心 agent loop 只有約 1,600 行，但支援了 15+ 聊天頻道、7+ LLM provider、14+ 內建工具、MCP 整合、cron 排程、WebUI 等生產就緒功能。

## 解決什麼問題

AI agent 框架常面臨「要嘛太重（LangGraph）、要嘛太學術（Autogen）、要嘛綁特定平台（Claude Code）」的困境。Nanobot 的定位是：

- **極輕量但可長期執行** — 核心 loop 夠小能讀完，但仍支援 persistent session、memory consolidation、cron 等需長時間運行的元件
- **研究友善** — codebase intentionally simple，適合學習、修改、延伸，不像 LangGraph 有 graph abstraction 的學習曲線
- **跨平台即用** — 一次設定就能同時接 Telegram、Discord、Feishu、Slack、WhatsApp、Email、WebSocket 等頻道
- **LLM provider 中立** — 抽象層在 `providers/base.py`，切換 provider 不需改 agent 邏輯

## 為什麼值得研究

- **MessageBus 解耦模式** — 頻道和 agent core 之間透過純 asyncio.Queue 解耦，比 callback 或 event emitter 更簡單、更容易測試
- **State Machine 驅動的 Agent Loop** — `TurnState` enum 定義了 RESTORE→COMPACT→COMMAND→BUILD→RUN→SAVE→RESPOND→DONE 的顯式狀態轉換，每個 stage 有明確的 entry/exit
- **Context Governance 的分層設計** — 每次 LLM call 前經過 `_microcompact`、`_snip_history`、`_apply_tool_result_budget`、`_backfill_missing_tool_results` 等多層修復，處理各種 edge case（orphan tool results、長度超標、角色交替違規）
- **Dream 兩階段記憶系統** — 非同步的 background LLM 記憶處理，第一階段提取事實，第二階段交叉比對，避免 blocking agent loop
- **輕量 plugin 架構** — tools、channels、providers 都透過 `pkgutil` 自動發現，第三方可透過 entry_point 註冊 plugin，不需修改核心程式碼

## Agent 系統定位

| 面向 | 選擇 |
|------|------|
| Agent 風格 | ReAct loop（觀察→思考→行動→觀察）搭配 TurnState state machine |
| 數量 | Single-agent |
| 編排方式 | LLM-decided（LLM 決定何時呼叫 tool、何時給最終答案） |
| Memory | Session history + JSONL 持久化 + Dream 兩階段長期記憶 |
| Tool calling | Native function calling（OpenAI/Anthropic 原生格式） |
| 並行度 | 預設 3 個 concurrent request，可透過 `NANOBOT_MAX_CONCURRENT_REQUESTS` 調整 |

## 技術棧一句話

`Python 3.11+` + `asyncio` + `MessageBus` + multi-provider (Anthropic, OpenAI, Azure, Bedrock, GitHub Copilot, OpenAI Codex, openai-compat) + `MCP` + `Jinja2` prompts + WebUI (React/TypeScript)

## 健康度信號

- ⭐ Stars: ~42.9k
- 📅 最後 commit: 2026-05-21
- 👥 主要維護者: Xubin Ren + HKUDS team
- 🔄 commit 頻率: 每日活躍（v0.1.4 → v0.2.0 一個半月內有數百次 commit）

## 跟主要競品的簡要對照

| 面向 | Nanobot | Hermes Agent | Claude Code |
|------|---------|-------------|-------------|
| 核心抽象 | MessageBus + TurnState FSM | plugin-based agent loop | CLI-based coding agent |
| 頻道支援 | 15+（Telegram/Discord/Feishu/等） | 多平台 | 無（純 CLI） |
| Provider 彈性 | 7+ backend + fallback | 多 provider | 僅 Anthropic |
| 程式碼規模 | 核心 ~1,600 行 | 較大 | 閉源 |
| 適用場景 | 個人長期 agent + 跨平台 | CLI agent + 工作流 | 程式碼開發 |

## 我會在後續筆記中回答的問題

- Nanobot 的 TurnState 狀態機如何讓 agent loop 保持可預測？
- MessageBus 的 asyncio.Queue 解耦模式比起 event bus 有什麼取捨？
- Dream 兩階段記憶系統如何在 background 處理記憶而不影響主 loop？
- 14+ 工具如何透過 pkgutil 自動發現並註冊？
