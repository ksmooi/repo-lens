---
repo: karpathy/llm.c
file: README
studied_at: 2025-06-26
commit_sha: f1e2ace
type: model-arch
language: Cuda (C/CUDA)
framework: 自製 CUDA 訓練框架
stars: 30.0k
status: active
---

# llm.c · 概覽

## 解決什麼問題

llm.c 解決一個核心矛盾：**想深入理解 LLM 訓練，但 PyTorch/HuggingFace 的抽象層太厚了**。Andrej Karpathy 用 ~1,000 行純 C 實作了 GPT-2 的完整訓練 pipeline — 從 token embedding 到 cross-entropy loss、從 forward 到 backward、從 AdamW 到 checkpoint — 全部在一個檔案內，沒有 autograd，沒有 nn.Module，只有 plain C arrays 和直接的數學運算。

這是 nanoGPT 的「進一步簡化」版本，同時也是「進一步加速」的版本：GPU CUDA 版本進一步擴展到 ~1,900 行主檔 + 14 個 CUDA kernel 模組，支援 BF16/FP16/FP32 混合精度、cuBLASLt、cuDNN flash attention、多 GPU（ZeRO Stage 1）、activation recomputation 等生產級功能，而在超參數與資料 pipeline 對齊下，**效能比 PyTorch 快 ~7%**。

## 為什麼值得研究

| 面向 | 說明 |
|---|---|
| **教育價值** | 全部在一個 C 檔案的 CPU 版（1182 行），是理解 Transformer 訓練最乾淨的教學材料 |
| **硬底子 CUDA** | 14 個手寫 kernel 模組，128-bit 向量化、streaming cache hints、online softmax、fused residual+layernorm — 是學習高效 CUDA 的絕佳參考 |
| **效能對照** | 同一演算法在 C/CUDA vs PyTorch 的實作對照，可直接比較效能開銷與 abstraction tax |
| **production 級功能** | 看似教學 repo 卻支援 cuDNN、cuBLASLt、ZeRO、gradient checkpointing、HellaSwag eval |
| **自建訓練框架的起點** | 想 clone 一個最小的 LLM 訓練框架來改，這裡比 nanoGPT 更底層、比 pytorch 更可控 |

## 跟競品的比較

| 面向 | llm.c | nanoGPT | PyTorch + HF Transformers |
|---|---|---|---|
| **依賴** | 無（只需 C/CUDA + cuBLAS） | PyTorch + tiktoken | PyTorch + transformers + 數百 MB |
| **抽象層級** | 直接陣列運算 | Python class + nn.Module | 高度封裝的 Trainer API |
| **可讀性** | 單一檔案，所有 ops 可視 | 多檔案，分散式設計 | 工廠模式 + 大量繼承 |
| **生產力** | 低（每層都要手寫 grad） | 中（autograd 自動） | 高（Trainer 一行啟動） |
| **訓練速度** | 比 PyTorch 快 ~7% | 等同 PyTorch | 基準線 |
| **支援模型** | GPT-2 / GPT-3 / Llama3 | GPT-2 | 幾乎所有模型 |
| **多 GPU** | NCCL + ZeRO-1 | DDP | FSDP + DeepSpeed |
| **適合** | 想徹底理解訓練的人 | 想快速實驗的開發者 | 生產環境部署 |

## 一句話

「當你的 debug 工具只剩 printf 和 cuda-memcheck，你才會真的「看到」Transformer 裡的每一條資料是怎麼流的。」

## 健康度信號

- ⭐ Stars: ~30.0k（2025 年 5 月）
- 📅 最後 commit: 2025-06-26
- 👥 主要維護者: Andrej Karpathy + 社群貢獻（>50 contributors）
- 🔬 持續支援 GPT-3 系列的 GPT-3 124M/350M/774M/1558M 以及 Llama3

## 我會在後續筆記中回答的問題

- 為什麼 C 版比 PyTorch 還快 7%？抽象稅到底繳在哪？
- 手寫 attention backward 的 online softmax 怎麼反向？
- 沒有 autograd 的情況下，怎麼手動維護所有 grad 的 chain rule？
- 用純 C 做 mixed precision training 的工程挑戰是什麼？
- 一個 30K stars 的「教學 repo」是如何從 1000 行 C 進化到支援 ZeRO + cuDNN 的？
