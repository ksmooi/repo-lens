---
repo: hiyouga/LLaMA-Factory
type: llm-training
studied_at: 2026-05-24
commit_sha: 16ff5a23
language: Python
framework: PyTorch
task: LLM Fine-Tuning (SFT / RLHF / DPO / PPO / KTO)
paper: ACL 2024 (arXiv:2410.19128)
stars: 71,538
status: active
---

# LLaMA-Factory · 統一微調框架的工程取捨

## 解決什麼問題

LLaMA-Factory 解決的是 LLM 微調領域的「組合爆炸」問題：模型架構（100+ 種）× 微調方法（full / LoRA / QLoRA / OFT / freeze）× 訓練階段（SFT / DPO / PPO / KTO / RM / PT）× 分散式策略（DDP / FSDP / DeepSpeed / Ray）的笛卡兒積。每個組合都可能需要不同的 dataloader、loss function、optimizer 配置。LLaMA-Factory 的做法是**用一個統一的 config 介面 + 單一入口 `llamafactory-cli train`，把所有組合的設定參數化**。

本質上它是一個 **LLM 微調的 compiler**：你把 "what to train"（模型、資料、方法）用 YAML 描述，它幫你編譯成底層的 HuggingFace Trainer + PEFT + TRL 組合。

## 為什麼值得研究

1. **規模驗證** — 71k stars、1000+ citations、被 Amazon / NVIDIA 使用，代表其工程設計經受了大規模用戶考驗
2. **架構整潔度** — 所有訓練階段共用同一套 `_training_function()`，差異僅在 Trainer 類別與 loss function（template method pattern 的極致運用）
3. **擴充性設計** — 121 個 conversation template、6 種 dataset source、33+ 種 VLM 的 mm_plugin，全部透過 registration 而非 if/else
4. **optimizer plugin 系統** — 7 種 optimizer 變體（GaLore、APOLLO、LoRA+、BAdam、Adam-mini、Muon）全都可插拔，不影響主訓練迴圈
5. **production-grade 細節** — checkpoint resume、分布式 crash recovery、FP8、混合精度、gradient checkpointing、profiling 一應俱全

## 任務與資料

| 面向 | 值 |
|---|---|
| 任務類型 | Supervised fine-tuning / RLHF / Preference optimization |
| 輸入 | Text / Multi-turn dialogue / Image-Text / Video-Text / Audio-Text |
| 輸出 | Text tokens / Reward scores / Policy logits |
| 主要資料集 | HuggingFace Hub、ModelScope、OpenMind、本地檔案（JSON / JSONL / Arrow / Parquet） |
| 評估指標 | Perplexity / BLEU / ROUGE / Reward accuracy |

## 一句話架構

> 一個基於 HuggingFace Transformers 的微調框架，以 **`_training_function()`** 為核心調度器，將 6 種訓練階段、5 種微調方法、7 種 optimizer、3 種分散式策略參數化為 YAML 配置。

## 對應論文

- **論文**: LLaMA-Factory: A Unified Framework for Fine-Tuning 100+ Large Language Models ([arXiv:2410.19128](https://arxiv.org/abs/2410.19128))
- **ACL 2024 接收**
- **implementation 與論文偏離**: 論文發表時支援 40+ 模型，目前 codebase 已擴展到 100+ 模型；分散式訓練支援（Ray / Elastic / MCA）遠超出論文範圍

## 健康度信號

| 指標 | 值 |
|---|---|
| ⭐ Stars | 71,538 |
| 📅 最後 commit | 2026-05-21 |
| 👥 主要維護者 | hiyouga（buaa.edu.cn，單一主要維護者 + 數十位貢獻者） |
| 🔬 對應論文 | ACL 2024，持續活躍開發中 |
| 🐳 Docker | 提供 CUDA / NPU / ROCm 三種映像 |
| 📦 PyPI | `llamafactory` 套件可 pip install |

## 與競品的比較

| 面向 | LLaMA-Factory | Axolotl | TRL | Unsloth |
|---|---|---|---|---|
| 核心抽象 | YAML config → 一體化入口 | YAML config → 一體化入口 | HuggingFace Trainer 擴展 | 手寫 Triton kernel 加速 |
| 支援模型數 | 100+ | ~40+ | ~10（HuggingFace 生態） | 10+ |
| 訓練階段 | PT / SFT / RM / PPO / DPO / KTO | SFT / DPO | SFT / PPO / DPO / GRPO | SFT / DPO |
| 分散式 | FSDP / DeepSpeed / Ray / MCA | FSDP / DeepSpeed | FSDP / DeepSpeed | FSDP |
| Web UI | ✅ LLaMA Board（Gradio） | ❌ | ❌ | ❌ |
| OpenAI API | ✅ 內建 API server | ❌ | ❌ | ❌ |
| 量化支援 | BNB / HQQ / EETQ / GPTQ / AWQ / AQLM / MXFP4 / FP8 | BNB | BNB / GPTQ / AWQ | BNB / GPTQ |
| 主要取捨 | 深度依賴 HF ecosystem，無法脫離 transformers | 靈活性更高但維護負擔重 | HF 官方但功能較少 | 速度優先但支援模型有限 |

## 我會在後續筆記中回答的問題

- LLaMA-Factory 如何用單一 `_training_function()` 處理 6 種訓練階段的差異？
- 121 個 conversation template 是怎麼註冊與解析的？為什麼需要這麼多？
- 7 種 optimizer 變體如何無縫嵌入 HuggingFace Trainer？
- 多模態支援的 `mm_plugin` 架構如何擴展？
- YAML config 到 HuggingFace HfArgumentParser 的轉換鏈是怎麼做的？
