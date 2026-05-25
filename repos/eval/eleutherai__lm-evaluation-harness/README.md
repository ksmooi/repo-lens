---
repo: EleutherAI/lm-evaluation-harness
type: eval
studied_at: 2026-05-24
commit_sha: 95d5806
language: Python
category: llm-evaluation-framework
api_style: imperative + YAML-declarative (task config)
stars: 12.7k
status: active
---

# LM Evaluation Harness · 概覽

## 解決什麼問題

LM Evaluation Harness 是 EleutherAI 維護的統一 LLM 評估框架。它解決的是：

- **評估標準化**— 同一個模型在不同 paper 裡用不同 prompt、不同 metric，無法橫向比較。Harness 提供了統一的 prompt 格式、metric 計算、結果格式
- **模型後端多樣化**— HF transformers、vLLM、OpenAI API、SGLang、TensorRT-LLM 等 20+ 後端，共用同一套 evaluation pipeline
- **task 定義可分享**— 用 YAML 描述一個 benchmark 的所有參數（資料集、prompt template、metric），不必寫 Python 程式碼

## 為什麼值得研究

1. **YAML-as-config 的 task 定義系統** — 13,556 個 task 定義檔全部用 YAML + `!function` tag 實現「宣告式 task，必要時程式碼擴充」，是 declarative config 在評估領域的大規模實戰
2. **LM 抽象層的設計取捨** — 只用 4 個 abstract method (`loglikelihood`、`loglikelihood_rolling`、`generate_until`、`apply_chat_template`) 就涵蓋了所有評估類型，是 interface 設計的範例
3. **分散式評估的 rank 管理** — 如何讓 data parallel / tensor parallel 的每個 rank 只處理部分資料，最後在 rank 0 聚合結果
4. **TaskManager + Registry 雙層發現機制** — 可擴充的註冊系統 + 目錄掃描自動發現

## API 風格一句話

`lm-eval run --model hf --model_args pretrained=gpt2 --tasks hellaswag` — 評估是「CLI 指令 + YAML config」，不是寫 Python 腳本

## 技術棧一句話

`Python >= 3.8` + `HuggingFace transformers (主要後端)` + `PyYAML (!function tag)` + `SQLite (request caching)` + `sacrebleu (翻譯 metric)`

## 健康度信號

- ⭐ Stars: ~12,700
- 📅 最後 commit: 2026-05-11
- 📦 最新版本: 0.4.13.dev0（持續開發中）
- 📈 Release 頻率: 約每月一次
- 🔄 SemVer 遵循狀況: 寬鬆，pre-release 頻繁
- 🏢 贊助: EleutherAI（非營利研究團體）
- 🔧 活躍貢獻者: ~50 人

## 競品比較

| 面向 | LM Evaluation Harness | DeepEval | Inspect AI | lmms-eval |
|---|---|---|---|---|
| 主要抽象 | YAML task config + Task class | pytest plugin + LLM-as-judge | Task + Solver + Epochs | Fork of Harness, 多模態 |
| 模型後端 | 20+（HF/vLLM/OpenAI 等） | LLM-as-judge（OpenAI）為主 | 支援 HF/OpenAI | 同 Harness + 多模態 |
| 評估範圍 | 學術 benchmark（MMLU/GSM8K 等 60+） | 應用層評估（hallucination/bias/toxicity） | 學術 + safety 評估 | 多模態 benchmark |
| 可擴充方式 | YAML + !function + Python Task subclass | Metric 自訂 + plugin | Solver chain | 同 Harness |
| 主要 trade-off | 強在標準化，弱在應用層評估（無 RAG/Agent 評估） | 強在應用層，弱在標準化 benchmark | 最靈活但學習曲線陡 | 多模態專用 |

## 我會在後續筆記中回答的問題

- YAML only 的 task 定義能做到什麼程度？什麼時候需要 fallback 到 Python？
- `Instance` + 4 種 output type 的設計是如何涵蓋所有評估類型的？
- 分散式評估中 rank 之間的資料分配與結果聚合是怎麼做的？
- 為什麼 Harness 選擇 SQLite 做 request caching 而不是其他方案？
