# model-arch/ — 模型架構實作

**上層目錄**: [repos/](../README.md)

特定模型架構的 reference implementation 或論文復現。
學習重點在於「論文的核心思想如何轉化成乾淨、可讀的程式碼」。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **論文對應** | 哪些設計 1:1 對應論文?哪些做了工程上的妥協?偏差的理由是什麼? |
| **關鍵元件** | attention 變體(MHA / GQA / MLA)?position encoding(RoPE / ALiBi / NoPE)?normalization(RMSNorm / LayerNorm)? |
| **Hyperparameter** | 關鍵超參數的具體數值?跟論文的差異?scaling 行為? |
| **Forward 流程** | 一次 forward pass 資料怎麼流動?哪裡有 reshape?哪裡有 einsum? |
| **Inference 路徑** | eval mode 跟 train mode 的差異?KV cache 怎麼整合?sampling 策略? |
| **可讀性 vs 效能** | 是否為了可讀性犧牲效能?哪些地方有 fused op?哪些刻意保持 naive? |
| **依賴最小化** | 外部依賴有多少?能不能只用 PyTorch / JAX 跑起來? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 架構 | 特色 |
|------|------|------|
| nanoGPT | GPT-2 | 極簡、高可讀、Karpathy 出品、400 行訓練完 |
| RWKV | RNN + attention hybrid | 線性複雜度、無限 context、可平行訓練 |
| Mamba | State Space Model | selective scan、硬體高效、優於 attention in long context |
| DiT | Diffusion Transformer | 把 U-Net 換成 Transformer、scalable diffusion |
| nanoDiffusion | minimal diffusion | 教學向、DDPM 最小實作 |
| ModernBERT | encoder-only | RoPE + Flash Attention、2024 BERT 現代化版本 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 以模型架構為核心的論文復現 | ✅ `model-arch` | — |
| 含完整訓練框架(FSDP / DeepSpeed) | — | ✅ [`llm-training`](../llm-training/) |
| 含 serving / batching 優化 | — | ✅ [`llm-serving`](../llm-serving/) |
| 視覺 + 語言多模態架構 | — | ✅ [`multimodal`](../multimodal/) |
| 框架層 kernel 設計(Triton / CUDA) | — | ✅ [`dl-framework`](../dl-framework/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
