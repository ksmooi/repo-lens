# agentic/ — LLM Agent 系統

**上層目錄**: [repos/](../README.md)

LLM agent 框架、multi-agent 系統、orchestration 引擎、agent runtime。
學習重點在於「agent 怎麼思考、怎麼行動、怎麼記憶」的設計抉擇。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **Agent 控制流** | 是 ReAct、Plan-Execute、StateGraph 還是自製?終止條件怎麼判斷?retry / fallback 策略? |
| **Prompt 系統** | prompt 存在哪?如何版本管理?token budget 怎麼控制?system / user 切割慣例? |
| **Tool / Function calling** | tool 註冊機制?schema 定義方式?tool failure 後 LLM 怎麼知道? |
| **Memory 架構** | short-term 結構?long-term 用什麼後端?寫入時機?讀取策略? |
| **LLM Provider 抽象** | 抽象層厚還是薄?支援哪些 providers?是否有 fallback / cost routing? |
| **Multi-agent 協作** | agents 之間怎麼通訊?編排者是誰?衝突怎麼解決? |
| **可觀測性** | tracing 用什麼?token / cost 怎麼追蹤?有沒有內建 eval? |

套用模板:`_templates/agentic.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| LangGraph | graph-based agent 框架 | 顯式 StateGraph、可中斷續跑、human-in-the-loop |
| CrewAI | multi-agent 協作框架 | Role-based agent、task 抽象、process 編排 |
| AutoGen | 對話式 multi-agent | ConversableAgent、group chat、code execution |
| Hermes Agent | 自我演進 CLI agent | skills 系統、memory、MCP 整合、cron 排程 |
| Swarm | 輕量 agent handoff | 極簡 API、educational、OpenAI 官方實驗專案 |
| Burr | state machine agent | Python-native、可視化、production 導向 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 以 agent 框架為核心,RAG 是工具之一 | ✅ `agentic` | — |
| 以檢索增強為核心,agent 是選配 | — | ✅ [`rag`](../rag/) |
| LLM 推論引擎(不涉及 agent 邏輯) | — | ✅ [`llm-serving`](../llm-serving/) |
| 評估 agent 能力的 benchmark 框架 | — | ✅ [`eval`](../eval/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
