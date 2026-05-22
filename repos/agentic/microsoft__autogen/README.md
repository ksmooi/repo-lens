---
repo: microsoft/autogen
type: agentic
file: README
studied_at: 2026-05-22
commit_sha: 027ecf0
language: Python
framework: autogen
agent_style: multi-agent
stars: 58.3k
status: active
---

# AutoGen · 概覽

Microsoft 的 agentic AI 程式框架，v0.7 是完全從零重寫的版本，捨棄了 v0.2 的 Legacy API，改以 actor-model 的 agent runtime 為核心，上層建構 ChatAgent → GroupChat 的編排層，並支援 .NET 跨語言互操作。

## 解決什麼問題

它讓開發者可以**用宣告式方式編排 multi-agent 系統**，而不需要自己實作 actor runtime、訊息路由、以及 state 管理。具體來說：

- 提供 single-threaded actor runtime（`SingleThreadedAgentRuntime`），內建模組派發（Topic / Subscription）與序列化
- 把 agent 定義簡化為 `on_messages()` 介面，team 層負責輪詢 / 選擇 / swarm 等編排策略
- 透過 `Component` 機制支援 declarative config（YAML/JSON）載入，方便跟 AutoGen Studio 整合
- 內建 .NET 支援與 gRPC 跨語言通訊，這在 agent 框架中很罕見

## 為什麼值得研究

1. **架構選擇跟市面上大部份 agent 框架都不同** — 底層是 actor model（類似 Erlang/Orleans），不是 graph（如 LangGraph）或 ReAct loop（如 LangChain AgentExecutor）。這個選擇影響了並行、state、可觀測性的整個設計
2. **v0.7 是 clean rewrite** — team 在 2024 年下半年決定重寫，捨棄了 v0.2 的 API 相容包袱。這種「斷尾求生」的決策本身很值得看
3. **Component 系統是 declarative agent 的 reference implementation** — 從 `ComponentModel` 到 `ComponentLoader` 的完整實作，是思考「怎麼讓 agent 系統可以被序列化為 YAML/JSON」的最佳教材
4. **跨語言支援（Python + .NET + gRPC）** — 在 agent 生態系中幾乎沒有其他框架做到這個程度

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | Actor model + message passing |
| 數量 | multi-agent（RoundRobin / Selector / Swarm / MagenticOne） |
| 編排方式 | 由 GroupChatManager 決定（輪詢 / LLM 選擇 / graph flow） |
| Memory | 可插拔（ListMemory 為內建實作），掛載在 AssistantAgent 上 |
| Tool calling | OpenAI 風格 function calling + 自製 Tool 抽象 |

## 技術棧一句話

`Python 3.10+` + `Pydantic 2` + `protobuf` + `OpenTelemetry` + `.NET` + `gRPC`

## 健康度信號

- ⭐ Stars: ~58.3k
- 📅 最後 commit: 2026-04-15
- 👥 主要維護者: Microsoft 團隊（含多核心維護者）
- 🔄 commit 頻率: 活躍，v0.7 系列持續更新

## 我會在後續筆記中回答的問題

- 為什麼選擇 actor model 而不是 graph / ReAct？
- v0.7 的 Component 系統怎麼做到 declarative agent config？
- Team 跟 Agent 之間的訊息路由是怎麼實作的？
- 跟 LangGraph / CrewAI 比，核心取捨是什麼？
