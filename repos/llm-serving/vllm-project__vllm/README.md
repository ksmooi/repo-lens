---
repo: vllm-project/vllm
type: llm-serving
file: README
studied_at: 2026-05-24
commit_sha: 0902d8e
language: Python + CUDA + Rust
framework: PyTorch
task: text generation / embedding / classification
paper: Efficient Memory Management for Large Language Model Serving with PagedAttention (arXiv:2309.06180)
stars: 80.8k
status: active
---

# vLLM · 概覽

vLLM 是高吞吐量的 LLM 推論與 serving 引擎，起源於 UC Berkeley Sky Computing Lab。它的核心貢獻是 **PagedAttention**——一種受作業系統虛擬記憶體啟發的 KV cache 管理方式，將傳統連續的 KV cache 切成固定大小的 block，大幅減少記憶體碎片並允許跨 request 的 prefix cache 共享。

## 解決什麼問題

LLM serving 的核心矛盾是：**KV cache 佔用大量 GPU 記憶體，而傳統實作（連續記憶體分配）會因碎片化和過度預留而浪費 60-80% 的記憶體**。vLLM 用 PagedAttention 解決這個問題，把 memory utilization 從 ~40% 拉到 ~95%。節省下來的記憶體可以直接轉換為更大的 batch size → 更高的吞吐量。

## 為什麼值得研究

1. **80k+ stars 的生產級專案** — 不是 research code，是大量企業（包括 OpenAI、Anthropic、Meta 等實作與貢獻）實際在 production 使用的引擎。程式碼品質、測試覆蓋、文件完善度都是 open-source ML infra 的標竿
2. **PagedAttention 的完整實作** — 從顯存管理（BlockPool + FreeKVCacheBlockQueue）到排程器（continuous batching + chunked prefill + prefix caching）到 CUDA kernel（attention 後端），全部開源可讀
3. **CUDA graph + torch.compile 的工程整合** — 支援分段式 CUDA graph（piecewise）與完整 CUDA graph（full），以及 Dynamo-based 的自動 compilation，展示了 production-grade inference optimization 的實踐
4. **分散式推論的完整架構** — tensor parallelism（TP）、pipeline parallelism（PP）、expert parallelism（EP）、context parallelism（CP）、data parallelism（DP），以及跨節點的 KV cache transfer（disaggregated prefill/decode）
5. **2025年 V1 引擎重構** — 捨棄舊版 engine core，完全以 V1 架構重寫，展示了「在 production 專案中做大型重構」的工程決策

## 技術棧一句話

`Python 3.10+` + `PyTorch` + `CUDA C/C++` + `Rust` + `FastAPI` + `msgspec` + `flash-attention`

## 跟競品的比較

| 面向 | vLLM | SGLang | TGI | llama.cpp |
|------|------|--------|-----|-----------|
| KV cache 管理 | PagedAttention（block-level） | RadixAttention（trie-based） | 連續記憶體 | 連續記憶體 + mmap |
| Prefix caching | Hash-based block caching | Radix tree prefix sharing | soft | 無 |
| 主要瓶頸 | KV cache 記憶體 | prefix cache miss | 連續記憶體碎片 | CPU/GPU 頻寬 |
| 擅長場景 | 高吞吐量 serving（多 request 交錯） | 結構化輸出 + 大量 prefix 共享 | HuggingFace 生態整合 | 低資源 / 邊緣部署 |
| 分散式推論 | TP/PP/EP/CP/DP 全部支援 | TP/PP/DP | TP/PP | 有限 |
| 模型支援數 | 200+ 架構 | ~50 架構 | ~20 架構 | 廣泛（GGUF） |
| 開發語言 | Python + CUDA + Rust | Python + CUDA + Triton | Python + CUDA + Rust（Candle） | C/C++ |
| 推理加速 | CUDA graph + torch.compile + FlashInfer + CUTLASS | FlashInfer + Triton | FlashAttention + VLLM（相容） | ggml |

## 健康度信號

- ⭐ Stars: ~80.8k（2026年最活躍的 LLM infra 專案之一）
- 📅 最後 commit: 2026-05-23（活躍開發中）
- 👥 主要維護者: 社群驅動（2000+ contributors），原始作者 Woosuk Kwon 仍活躍
- 🏢 底層組織: UC Berkeley Sky Computing Lab → vLLM 專案
- 🔄 commit 頻率: 每天數十個 PR
- ✅ CI: buildkite 完整 CI pipeline，含硬體測試

## 我會在後續筆記中回答的問題

- PagedAttention 的 block 管理為何比連續記憶體好那麼多？
- Continuous batching 的排程器如何決定 prefill/decode 比例？
- GPUModelRunner.execute_model 的完整執行路徑是什麼？
- vLLM V1 引擎的「async scheduling」跟 batch queue 是怎麼設計的？
- 分段式 CUDA graph（piecewise）vs 完整 CUDA graph 的取捨是什麼？
