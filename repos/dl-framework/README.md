# dl-framework/ — 深度學習框架核心

**上層目錄**: [repos/](../README.md)

深度學習框架核心、autograd 引擎、compiler / JIT、CUDA kernel、operator 設計。
學習重點在於「框架底層如何把數學抽象轉成高效硬體指令」的系統設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **Autograd** | 計算圖怎麼建?forward / backward 的 dispatch 機制?higher-order gradient? |
| **Tensor 抽象** | tensor 的記憶體 layout(stride / view)?dispatch key 設計?device / dtype 抽象? |
| **Compiler / JIT** | torch.compile / XLA / JAX jit 的 trace 機制?fusion 策略?graph capture 限制? |
| **Dispatcher** | operator 的 dispatch 表怎麼組織?custom op 怎麼註冊?backend 切換機制? |
| **CUDA Kernel** | Triton DSL vs CUDA C 的取捨?warp-level primitive 使用?shared memory 策略? |
| **分散式通訊** | NCCL / Gloo 怎麼整合?collective op 的 async 設計?gradient 壓縮? |
| **擴充性** | 自訂 backend?自訂 autograd function?torch.library 使用方式? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| PyTorch | 主流深度學習框架 | eager mode + compile、廣泛生態、研究友善 |
| JAX | 函數式 DL 框架 | XLA JIT、vmap / pmap、pure function 設計 |
| Triton | GPU kernel DSL | Python-like 語法、tile-based 程式設計、自動優化 |
| tinygrad | 極簡框架 | 3000 行、lazy evaluation、多 backend 支援 |
| TorchDynamo | PyTorch compiler | bytecode 分析、graph capture、backend 抽象 |
| Flashattention | 高效 attention kernel | IO-aware、tiling、2-4x 速度提升 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 框架核心(autograd / compiler / kernel) | ✅ `dl-framework` | — |
| 以框架為工具的訓練 codebase | — | ✅ [`llm-training`](../llm-training/) |
| 以框架為工具的推論引擎 | — | ✅ [`llm-serving`](../llm-serving/) |
| 特定模型架構復現(用 PyTorch 寫) | — | ✅ [`model-arch`](../model-arch/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
