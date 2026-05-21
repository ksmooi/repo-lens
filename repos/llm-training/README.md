# llm-training/ — LLM 訓練與微調

**上層目錄**: [repos/](../README.md)

LLM 預訓練、指令微調(SFT)、參數高效微調(LoRA / QLoRA)、對齊訓練(RLHF / DPO / PPO)。
學習重點在於「讓模型從 base 走向 aligned」的訓練工程設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **訓練 Loop** | SFT / RLHF / DPO / PPO 的 loop 差異?reward model 怎麼設計?KL penalty 策略? |
| **參數高效微調** | LoRA rank / alpha 怎麼選?QLoRA 的 NF4 量化細節?Adapter 跟 LoRA 的取捨? |
| **分散式訓練** | FSDP / DeepSpeed ZeRO Stage 選擇?gradient checkpointing 的代價?pipeline parallelism? |
| **資料 Pipeline** | dataset 格式(instruction / chat / preference)?tokenizer 選擇?packing / padding 策略? |
| **Memory 優化** | mixed precision(bf16 / fp16)?activation checkpointing?flash attention 整合? |
| **實驗管理** | hyperparameter 組合?checkpoint 策略?eval 頻率?早停判斷? |
| **對齊評估** | 訓練過程怎麼監控 alignment 品質?MT-Bench / AlpacaEval 整合? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| LLaMA-Factory | 統一微調框架 | 支援 40+ 模型、SFT / RLHF / DPO 一體、WebUI |
| Axolotl | 靈活微調工具 | YAML-driven config、多種 attention 實作、QLoRA |
| TRL | Transformer RL 函式庫 | HuggingFace 官方、PPO / DPO / GRPO trainer |
| OpenRLHF | 大規模 RLHF 框架 | Ray-based 分散式、Hybrid engine、70B+ 訓練 |
| Unsloth | 超快速微調 | 手寫 Triton kernel、2x 速度、低 VRAM |
| Megatron-LM | 大規模預訓練 | tensor / pipeline / sequence parallelism |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 訓練 / 微調 codebase | ✅ `llm-training` | — |
| 模型推論引擎(不涉及訓練) | — | ✅ [`llm-serving`](../llm-serving/) |
| 特定模型架構的復現(無訓練框架) | — | ✅ [`model-arch`](../model-arch/) |
| ML 實驗追蹤平台(非訓練本體) | — | ✅ [`ml-platform`](../ml-platform/) |
| 訓練資料 ETL pipeline | — | ✅ [`data-pipeline`](../data-pipeline/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
