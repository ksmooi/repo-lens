---
repo: confident-ai/deepeval
type: eval
studied_at: 2026-05-23
commit_sha: 17e676f
language: Python
category: llm-evaluation-framework
api_style: imperative + decorator-driven (pytest plugin)
stars: 15.6k
status: active
---

# DeepEval · 概覽

## 解決什麼問題

DeepEval 是專為 LLM 應用設計的評估框架——不是傳統的 accuracy / F1 benchmark，而是「LLM-as-a-judge」為核心的評估系統。它解決的是：

- **LLM 輸出的主觀評估**無法用傳統 metric（BLEU、ROUGE）衡量——需要另一個 LLM 來評分
- **RAG / Agent / Multi-turn** 等複雜場景各自需要不同的評估維度
- **評估結果需要與 CI/CD 整合**，而不是跑完就算
- **評估資料需要版本管理、團隊協作**（透過 Confident AI 平台）

## 為什麼值得研究

1. **G-Eval 的實作**（arXiv 2303.16634）——如何在生產級框架中實作 LLM-as-a-judge metric，包括 rubric、evaluation steps、chain-of-thought scoring
2. **DAG 式 metric 組裝**——用有向無環圖組裝多個 sub-metric 的設計
3. **pytest 插件深度整合**——把 LLM 評估無縫嵌入 Python 測試生態系
4. **多種評估模式**——end-to-end（black-box）、component-level（white-box）、trace-scope（agentic loop）
5. **框架整合策略**——LangChain callback handler、OpenAI Agents wrapper、Pydantic AI 整合等

## API 風格一句話

`from deepeval import assert_test` + pytest decorator — 評估是「斷言」，不是「執行腳本」

## 技術棧一句話

`Python >= 3.9` + `OpenAI (LLM-as-judge)` + `typer (CLI)` + `pytest (plugin)` + `Pydantic v2 (schema)` + `OpenTelemetry (tracing)`

## 健康度信號

- ⭐ Stars: ~15.6k
- 📅 最後 commit: 2026-05-21（活躍開發中）
- 📦 最新版本: v4.0.3
- 📈 Release 頻率: 每月 1-2 次
- 🔄 SemVer 遵循狀況: 嚴格
- 👤 主要維護者: Jeffrey Ip（Confident AI 創辦人）
- 💬 Discord 社群活躍，文件完善

## 競品比較

| 面向 | DeepEval | lm-evaluation-harness (EleutherAI) | Inspect AI (UK AISI) |
|------|----------|-----------------------------------|---------------------|
| **核心定位** | LLM 應用（RAG/Agent）評估 | 基礎模型 benchmark | 安全評估框架 |
| **主要抽象** | Metric + TestCase | Task + Dataset | Solver + Scorer |
| **評估方式** | LLM-as-judge（G-Eval）為主 | 標準化 benchmark 任務 | solver 執行 + scorer 評分 |
| **CI 整合** | pytest plugin（原生） | CLI 驅動 | CLI + SDK |
| **Agent 評估** | ✅ 專屬 agentic metrics | ❌ 無 | ❌ 以安全為主 |
| **多輪對話** | ✅ Conversational metrics | ❌ 無 | ✅ Task state |
| **Synthetic data** | ✅ 內建 synthesizer | ❌ 無 | ❌ 僅外部 |
| **自訂 metric** | `BaseMetric` subclass | 需寫 task adapter | Scorer 抽象 |
| **平台整合** | Confident AI（自家） | 無 | 無 |
| **研究支撐** | G-Eval, LLM-as-judge | MMLU, HellaSwag, BIG-Bench | 安全 evals |

DeepEval 的核心差異在於「LLM-as-judge 的 practicality」——它不評比模型知識，而是評比應用輸出的品質，且深度融入 pytest 生態系。

## 我會在後續筆記中回答的問題

- G-Eval 的 rubric / evaluation steps / chain-of-thought 怎麼設計？
- DAG metric 的評估圖是怎麼建構的？
- `assert_test` 的執行路徑從呼叫到結果回傳走了哪些層？
- 如何同時支援 sync 和 async 的 metric 執行？
- framework integration（LangChain / OpenAI Agents）的接入策略是什麼？
