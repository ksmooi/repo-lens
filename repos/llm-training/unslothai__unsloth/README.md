---
repo: unslothai/unsloth
file: README
type: llm-training
studied_at: 2026-05-25
commit_sha: eeb49d5
language: Python (CUDA, Triton)
framework: PyTorch
task: LLM fine-tuning / RL training
stars: 65.1k
status: active
---

# unsloth · 概覽

## 解決什麼問題

LLM 微調的兩大硬傷：**VRAM 不足以跑全參數微調**、**LoRA 雖省顯存卻比全參數訓練慢**（因為每個 LoRA adapter 都有額外的矩陣乘法開銷）。Unsloth 的核心洞察是「不把 LoRA 當成獨立 adapter 來計算，而是將 LoRA 的 forward/backward **融合進 base model 的線性層內**」，用自訂 Triton kernel 一次性算出 `X @ (W + A@B)@scale` 來消除冗餘 I/O 與 kernel launch。

結果是訓練 2x 快、VRAM 省 70%，且 accuracy 不降。

## 為什麼值得研究

Unsloth 的技術路線跟 LLaMA-Factory / Axolotl 等微調框架完全不同——它不是「幫你管理 config 和 script」，而是從 Triton kernel 層級直接重寫了訓練計算圖。這使得它：

1. **API 極簡** — 只要 `FastLanguageModel.from_pretrained()` 一行，所有優化在 import time 自動 patch
2. **monkey-patch 深度** — 直接修改 transformers/peft/trl 內部的 class `__init__` 與 `forward` 方法
3. **橫跨 SFT / DPO / GRPO** — 同一套 kernel 服務所有訓練範式
4. **65k stars** — 2023 起活躍至今，支援 500+ 模型架構

## 比較表格

| 面向 | Unsloth | LLaMA-Factory | TRL |
|------|---------|---------------|-----|
| 主要抽象 | import-time kernel patching | YAML config → training | HF Trainer subclass |
| 加速手段 | 手寫 Triton kernel (LoRA fusion) | 無（被動依賴 HF Trainer） | 無（同上） |
| RL 支援 | GRPO (patch TRL + vLLM 權重同步) | PPO / DPO / GRPO | PPO / DPO / GRPO / KTO |
| 模型支援數 | 500+ (含 MoE、VLM) | 40+ | 原生 Transformer 系列 |
| 自訂 config | 少（靠 kernel 自動優化） | 多（完全 YAML-driven） | 中等 (TrainingArguments) |
| 2x 加速 | ✅ 核心賣點 | ❌ | ❌ |
| 社群規模 | 65k stars | 34k stars | 32k stars |

## 健康度信號

- ⭐ Stars: ~65.1k
- 📅 最後 commit: 2026-05-25（每日活躍）
- 👥 核心團隊: Daniel Han-Chen + Unsloth team
- 📝 License: Apache-2.0
- 🔬 已從單純微調函式庫擴展為 Unsloth Studio（完整 Web UI + Tauri 桌面端）

## 我會在後續筆記中回答的問題

- 為什麼 monkey-patch 是比 subclass 更高效的選擇？
- Unsloth 的 LoRA fusion 跟標準 PEFT LoRA 在數學上差在哪？
- padding-free training 跟 sample packing 如何讓 throughput 翻倍？
- 為什麼 GRPO 的實作要走 code injection（模板替換）而不是繼承？
