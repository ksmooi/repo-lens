---
repo: nousresearch/hermes-agent
file: README
type: agentic
studied_at: 2026-05-21
commit_sha: 48be2e0
language: Python
framework: 自製（無第三方 agent framework 依賴）
agent_style: single-agent（支援 multi-agent via kanban plugin）
stars: ~160k
status: active
---

# Hermes Agent · 概覽

> The agent that grows with you — 一個會從經驗中學習、自我改進的 CLI-first AI agent。

## 解決什麼問題

Hermes Agent 不是「又一個 agent framework」，它是一個**可以直接用的 agent 產品**。它解決的是「如何讓一個 LLM agent 在日常開發工作中可靠地運作」這個實務問題，涵蓋：

- **持久化對話** — SQLite 儲存 session，支援 FTS5 全文搜尋、resume、branch
- **工具系統** — 1789 個 Python 檔案、~142K LOC 的程式碼庫，內建 terminal、file I/O、web search、browser、vision 等 50+ 工具
- **技能系統** — 讓 agent 從經驗中自動建立可重複使用的技能（SKILL.md），形成自我改進循環
- **多平台 gateway** — 同一 agent core 同時服務 CLI、Telegram、Discord、Slack、WhatsApp 等 40+ 平台
- **plugin 生態** — memory provider、model provider、observability、kanban 等可插拔模組

## 為什麼值得研究

1. **規模與成熟度** — 160k stars、1789 個 Python 檔案、~142K LOC，是少數真正 production-deployed 的 agent 專案（非 research prototype）
2. **自製 agent loop** — 沒有倚賴 LangGraph / CrewAI 等第三方框架，conversation loop 完全自建，可以看到 agent 系統所有真實的取捨
3. **自我改進設計** — Skill system 讓 agent 可以將完成過的任務抽象為可重用的 skill，形成「經驗→技能」循環，這是目前少見的設計
4. **Plugin 生態系** — 不只是 agent，同時是一個 plugin 平台（memory providers、model providers、gateway 平台、observability 等）

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | ReAct loop（synchronous while + tool calling） |
| 數量 | Single-agent（內建） / Multi-agent（kanban plugin） |
| 編排方式 | Hardcoded loop（非 declarative / graph-based） |
| Memory | Session（SQLite FTS5）+ Cross-session（memory provider plugin） |
| Tool calling | OpenAI-compatible function calling + native streaming |
| 模型通訊協定 | chat_completions / codex_responses / anthropic_messages |

## 技術棧一句話

`Python 3.11+` + `OpenAI SDK` + `SQLite (WAL+FTS5)` + `Rich/prompt_toolkit` + `Jinja2`

## 健康度信號

- ⭐ Stars: ~160k
- 📅 最後 commit: 2026-05-21（今天）
- 👥 主要維護者: Nous Research（全職團隊）
- 🔄 commit 頻率: 每日活躍

## 我會在後續筆記中回答的問題

- 為什麼選擇 synchronous conversation loop 而非 async？
- Tool 的 AST-based auto-discovery 是怎麼運作的？
- System prompt 的三層結構（stable / context / volatile）為什麼這樣切？
- Skill 從「經驗」到「技能」的自我改進循環如何實作？
- Gateway 的 40+ 平台 adapter 模式有什麼可取之處？

## 競品對照

| 面向 | Hermes Agent | LangGraph | Claude Code | Codex CLI |
|---|---|---|---|---|
| 主要抽象 | Agent class + tool registry | StateGraph + node/edge | Agent loop（閉源） | Agent loop（閉源） |
| 部署方式 | CLI + Gateway + ACP server | Python library | CLI only | CLI only |
| 工具系統 | Self-registering module | Node-implemented | Built-in | Built-in |
| 持久化 | SQLite FTS5 | User-managed | JSONL | JSONL |
| Self-improvement | Skill system（核心功能） | 無 | 無 | 無 |
| 平台 | CLI + 40+ messaging platforms | Python library only | CLI only | CLI only |
| 主要 trade-off | 重量級（~142K LOC）但功能完整 | 輕量但需自行拼裝 | 封閉生態 | 封閉生態 |
