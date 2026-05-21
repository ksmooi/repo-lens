# eval/ — 模型評估與 Benchmark

**上層目錄**: [repos/](../README.md)

模型評估框架、benchmark、red-teaming、safety testing。
學習重點在於「如何嚴謹、公平、可重現地衡量模型能力」的評估工程設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **任務定義** | 評估任務怎麼描述?few-shot 怎麼組裝?task 之間怎麼共用 utility? |
| **Metric 實作** | accuracy / F1 / BLEU / ROUGE 怎麼計算?judge model 怎麼整合?human eval 怎麼納入? |
| **Model Adapter** | 怎麼支援多個 LLM provider?API / local model 的 adapter 設計?prompt template 管理? |
| **Dataset 管理** | 資料集版本?HuggingFace datasets 整合?自訂 dataset 怎麼加? |
| **可重現性** | seed 控制?sampling 參數記錄?result caching?environment snapshot? |
| **Leaderboard** | 結果儲存格式?compare / diff?leaderboard 更新機制? |
| **Safety / Red-team** | jailbreak 測試集設計?harmful content 評分?自動 red-teaming 流程? |

套用模板:`_templates/library.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| LMMS-Eval | 多模態評估框架 | 50+ 任務、多模型支援、LMM 專用 |
| OpenCompass | LLM 評估平台 | 100+ 資料集、分散式評估、中文 benchmark |
| DeepEval | LLM 應用評估 | RAG / agent 評估、G-Eval、CI 整合 |
| Inspect AI | UK AISI 評估框架 | solver / scorer 抽象、安全評估、可擴充 |
| lm-evaluation-harness | EleutherAI benchmark | 200+ 任務、基礎模型標準評估 |
| promptbench | adversarial prompt | robustness 測試、prompt attack、分析工具 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 評估框架、benchmark、safety 測試工具 | ✅ `eval` | — |
| 訓練過程的 eval callback(非獨立框架) | — | ✅ [`llm-training`](../llm-training/) |
| Agent 能力的 benchmark(以 agent 設計為主) | — | ✅ [`agentic`](../agentic/) |
| 資料品質驗證工具(非模型評估) | — | ✅ [`data-pipeline`](../data-pipeline/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
