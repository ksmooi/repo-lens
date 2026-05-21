# llm-serving/ — 模型推論引擎與 Serving 框架

**上層目錄**: [repos/](../README.md)

模型推論引擎、serving 框架、inference optimization。
學習重點在於「如何讓 LLM 在生產環境跑得又快又省」的系統設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **Batching 策略** | continuous batching 怎麼實作?iteration-level scheduling?dynamic batch size? |
| **KV Cache** | PagedAttention 原理?KV cache 的記憶體管理?prefix caching?cross-request sharing? |
| **量化** | GPTQ / AWQ / FP8 / INT4 的精度-速度 trade-off?runtime 量化 vs 離線量化? |
| **Speculative Decoding** | draft model 選擇?acceptance rate 監控?speculative 的 overhead 值不值得? |
| **排程器設計** | request 優先級?preemption 策略?SLO-aware scheduling? |
| **多卡 / 多機** | tensor parallelism?pipeline parallelism?distributed KV cache? |
| **CUDA Kernel** | custom attention kernel?fused ops?Triton vs CUDA C? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| vLLM | 高吞吐推論引擎 | PagedAttention、continuous batching、廣泛 model 支援 |
| SGLang | structured generation 引擎 | RadixAttention prefix cache、constraint decoding |
| TGI | HuggingFace serving | gRPC / HTTP API、token streaming、多框架支援 |
| LitServe | 輕量 serving 框架 | FastAPI-based、batching、async worker |
| Ollama | 本地部署工具 | 使用者友善、自動量化、跨平台 |
| llama.cpp | CPU / Metal 推論 | GGUF 格式、quantized inference、embedded 部署 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 推論引擎、serving 框架 | ✅ `llm-serving` | — |
| LLM 微調 / 訓練框架 | — | ✅ [`llm-training`](../llm-training/) |
| 深度學習框架的 kernel / compiler | — | ✅ [`dl-framework`](../dl-framework/) |
| 量化工具函式庫(非 serving) | — | ✅ [`library`](../library/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
