---
repo: vllm-project/vllm
file: 9-questions
studied_at: 2026-05-24
commit_sha: 0902d8e
---

# vLLM · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 PagedAttention 不採用 CoW（Copy-on-Write）來處理 prefix 修改？**
  - 我目前的推測: 當多個 request 共享一個 prefix block 後，如果其中一個 request 在該 block 內做 KV cache update（attention 的 KV 計算不同，雖然 token 相同），會影響其他共享 request。但 PagedAttention 的 KV cache 是 append-only（每個 request 的運算結果不同），不可能真的「共用」運算結果。Prefix caching 共享的其實是「相同的 token → 相同的 KV 計算結果」，而不是「同一塊記憶體、不同的運算結果」。所以不是 CoW 問題。[UNVERIFIED]
  - 相關程式碼: [`vllm/v1/core/block_pool.py:211-331`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/block_pool.py#L211-L331)

- [ ] **為什麼不進行 prefix cache 去重複的設計決定？**
  - 我目前的推測: 文件說「保持 block table append-only」（`block_pool.py:48-52`），但我沒完全理解為什麼 append-only 是如此重要的不變量。是不是因為一旦 block ID 可以改變（去重複時，一個 block 被釋放、其 ID 被另一個 block 使用），model runner 中已 capture 的 CUDA graph 的 block table pointer 會失效？若是如此，這跟程式碼註解中提到的「block 的 physical ID 不變」這點相關。[UNVERIFIED]
  - 相關程式碼: [`vllm/v1/core/block_pool.py:48-52`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/block_pool.py#L48-L52)

- [ ] **Scheduler 的 preemption 為什麼是重置 `num_computed_tokens=0` 而非只釋放部分 block？**
  - 我目前的推測: 簡化實作。部分 preemption 需要跟 model runner 協商「哪些 token 的 KV cache 保留」，而 model runner 的 block table 管理可能不支援部分釋放。但這對長 prompt 的影響很大（prompt 可能已經 prefill 了 30k tokens，全部報廢）。[UNVERIFIED]
  - 相關程式碼: [`vllm/v1/core/sched/scheduler.py:929-949`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/sched/scheduler.py#L929-L949)

- [ ] **Disaggregated prefill/decode 的實作成熟度？**
  - 我目前的推測: Proxy-based（透過 `entrypoints/serve/disagg/`）的 disaggregation 似乎還在早期開發階段。KV cache transfer 的效能 overhead（序列化 + 傳輸 + 反序列化）在跨節點場景下可能是瓶頸，但我沒看到詳細的 benchmark。[UNVERIFIED]
  - 相關程式碼: [`vllm/entrypoints/serve/disagg/`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/entrypoints/serve/disagg/)

- [ ] **為什麼 AsyncLLMEngine 與 EngineCore 之間選擇 msgspec 而非 gRPC + Protobuf？**
  - 我目前的推測: msgspec 的序列化速度快（~10x vs json），對 Python 的原生 typing 支援好。但 gRPC 提供了 streaming、load balancing、health check 等開箱即用的功能。vLLM 選擇 msgspec 可能因為 (1) EngineCore 跟 frontend 通常在同一機器上； (2) 希望在 in-process mode 時可以跳過序列化直接傳物件。[UNVERIFIED]
  - 相關程式碼: [`vllm/v1/engine/__init__.py:81-198`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/engine/__init__.py#L81-L198)

## 想驗證的論文 claim

- [ ] **論文宣稱 PagedAttention 可以達到 near-zero waste 的記憶體利用率**：從程式碼來看，block size 16 時，最壞情況下每個 request 的最後一個 block 可能浪費 15/16 的空間（~6% waste）。這還不包含 KV cache 之外的其他記憶體（model weights、activations、CUDA context）。實際場景中 memory utilization 是多少？

- [ ] **論文 claim 「比 TensorFlow Serving 和 FasterTransformer 快 2-4x」**：這個比較是在 2023 年的硬體和模型上做的。2026 年的 SGLang 和 TensorRT-LLM 也在不斷進步，實際差距需要重新評估。

## 想問維護者的問題

- PagedAttention 的 block size 為什麼預設是 16？有沒有做過不同 block size（8/32/64）的 ablation study？
- V1 engine 重構中最難處理的問題是什麼？（相容性？效能倒退？分散式除錯？）
- 在支援 200+ 模型架構的過程中，哪個架構是最難支援的？為什麼？
- 對於希望貢獻 kernel 的新手，推薦從哪個方向開始？（Triton kernel？CUDA kernel 的簡單 attention variant？）

## 下次再看時的待辦

- [ ] 深入看 `vllm/v1/core/sched/async_scheduler.py` — 非同步排程器跟同步排程器的實作差異
- [ ] 閱讀 `breakable_cudagraph.py` 的 `@eager_break_during_capture` decorator 實作，理解 runtime 分割的具體機制
- [ ] 分析 `vllm/v1/worker/gpu/model_states/` 的 Mamba hybrid model states 實作
- [ ] 對照 SGLang 的 RadixAttention prefix cache 實作，理解兩者 prefix sharing 的取捨差異
- [ ] 看 `vllm/entrypoints/serve/disagg/` 的 disaggregated serving 流程，理解 KV cache transfer 的實作細節

## 跨專案對照備忘

- **PagedAttention 的 FreeKVCacheBlockQueue（O(1) 中間刪除的雙向鏈結串列）** 跟 `ray` distributed scheduler 中的 ObjectRef 管理有類似的需求：需要從多個位置快速移除物件。 → 可能形成候選 pattern「高效中間刪除的資源池管理」
- **Piecewise CUDA Graph** 跟 PyTorch 的 `torch.compile` + `dynamic=True` 有類似的目標（動態 shape 下的靜態編譯）。但 vLLM 的 piecewise approach 是 hand-tuned segmentation，而 torch.compile 是自動的。 → 可以追蹤在 LLM inference 場景下，兩種方式的效能差距是否隨 torch.compile 的進步而縮小
- **Engine frontend/core 分離（message-passing over shared-state）** 跟 AutoGen 的 actor model runtime 在概念上相似。兩者都選擇了「透過序列化協定隔離元件」的架構。 → 可能形成候選 pattern「production-grade AI 系統的元件隔離策略」
