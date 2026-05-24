---
repo: vllm-project/vllm
file: 3-key-patterns
studied_at: 2026-05-24
commit_sha: 0902d8e
---

# vLLM · 值得偷學的設計

## Pattern 1: PagedAttention — 作業系統風格的顯存管理

**是什麼**: 受 OS 虛擬記憶體啟發的 KV cache 管理方式。將連續的 KV cache 切成固定大小的 block（預設 16 tokens），透過 block table 進行 logical-to-physical 的間接定址。不需要為整條序列預留連續記憶體，而是按需分配 block。

**為什麼有效**: 傳統的 KV cache 實作有兩個問題：(1) 內部碎片 — 為最大可能序列長度預留記憶體，但實際序列較短；(2) 外部碎片 — 不同 request 的 KV cache 長度不同，在連續記憶體中交錯分配會產生無法使用的碎片。PagedAttention 的 block-level 分配解決了這兩個問題，memory utilization 從 ~40% 提升到 ~95%。

**關鍵實作**:
- BlockPool: [`vllm/v1/core/block_pool.py:149-182`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/block_pool.py#L149-L182)
- FreeKVCacheBlockQueue（雙向鏈結串列）: [`vllm/v1/core/kv_cache_utils.py:164-372`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/kv_cache_utils.py#L164-L372)
- KVCacheManager.allocate_slots: [`vllm/v1/core/kv_cache_manager.py:236-427`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/kv_cache_manager.py#L236-L427)

**替代方案**:
- **連續記憶體（TGI 的作法）**: 為每個 request 預留 `max_model_len × num_layers × 2 × dtype_size` 的連續記憶體。實作簡單，但記憶體利用率低（60-80% 被浪費）。適用在 request 數量少的場景（batch size << 10）。
- **RadixAttention（SGLang 的作法）**: 用 trie（前綴樹）管理 KV cache，共享粒度是完整的 prefix 路徑。prefix cache hit 率更高（因為可以共享任意長度的 prefix 而非只有 block-aligned），但 trie 的節點 overhead 和分岔點的記憶體管理更複雜。

**何時可以借用**: 任何需要管理大量固定大小物件的高併發系統 — 不只是 KV cache，也可以是 connection pool、buffer pool、file descriptor table。核心模式是：logical-to-physical indirection + fixed-size blocks + free list。

**注意事項**: Block size 的選擇是關鍵 trade-off。太小 → 增加 block table overhead 和 prefix cache 開銷；太大 → 內部碎片增加。建議從應用場景的典型物件大小分布來決定。

---

## Pattern 2: Chunked Prefill — 長請求的搶佔式處理

**是什麼**: 不是等到整個 prompt 的 prefill 完成才開始 decode，而是將長 prompt 切成多個 chunk，每個 chunk 的 prefill 之間可以穿插 decode request。由 `max_num_scheduled_tokens` 和 `long_prefill_token_threshold` 兩個參數控制。

**為什麼有效**: 如果一個長 prompt（例如 32k tokens）需要連續 prefill 而不中斷，這段期間（數百毫秒到數秒）所有其他 request 的 decode 都會被 blocking。Chunked prefill 讓 decode latency 不會因為某個長 prompt 而飆升，穩定 p99 latency。

**關鍵實作**:
- Token budget 計算: [`vllm/v1/core/sched/scheduler.py:385-395`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/sched/scheduler.py#L385-L395)
- Chunked prefill 條件判斷: [`vllm/v1/core/sched/scheduler.py:659-667`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/sched/scheduler.py#L659-L667)

**替代方案**:
- **全 prefill（disable chunked prefill）**: 簡單，每條 prompt 一次處理完。適用在 request 量少或 prompt 短的場景（如示範 demo）。
- **Pause-based 策略**: prefill 跑到一定階段後暫停，先做 decode，再回來繼續 prefill。實作更複雜但能更細粒度控制延遲。

**何時可以借用**: 任何「處理長任務時不能 blocking 短任務」的系統 — 資料庫查詢排程、多租戶 batch processing、CDN 的請求優先級管理。

**注意事項**: Chunked prefill 增加每個 prefill chunk 的 overhead（每次 schedule 都要重新配置 kernel launch）。當 `enable_chunked_prefill=False` 時，一條 prompt 的 prefill 一定在單次 schedule 中完成，對於短 prompt 或低併發場景是更簡單的選擇。

---

## Pattern 3: Piecewise CUDA Graph — 動態 Shape 下的靜態編譯

**是什麼**: 將模型的 forward 計算圖分段 capture 為多個 CUDA graph segments，而不是 capture 整個 forward。每個 segment 可以有不同的 shape，且 segments 之間透過 eager-mode 的 kernel 連接。支援兩種實作：Dynamo-based（編譯期分割）和 breakable CUDA graph（runtime 分割）。

**為什麼有效**: 傳統 CUDA graph 的限制是 graph 必須有固定的 shape（固定的 batch size、seq len、num tokens）。這對 prefill 完全不適用（shape 變化大），對 decode 也只有 batch size 固定時才有效。Piecewise CUDA graph 解決了這個問題：attention + KV cache 操作（shape 變化大）用 eager 執行，其餘計算（MLP、layer norm — shape 固定）用 CUDA graph。這樣可以 capture ~70% 的計算量，比全 eager 快 15-30%。

**關鍵實作**:
- CudagraphDispatcher: [`vllm/v1/cudagraph_dispatcher.py:239-328`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/cudagraph_dispatcher.py#L239-L328)
- Piecewise backend: [`vllm/compilation/piecewise_backend.py`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/compilation/piecewise_backend.py)
- Breakable CUDAGraph: [`vllm/compilation/breakable_cudagraph.py`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/compilation/breakable_cudagraph.py)

**替代方案**:
- **Full CUDA graph**: capture 整個 forward。最有效率（0 kernel launch overhead），但只能用在 uniform decode（固定 batch size + token count）。vLLM 同時支援此模式，但只在特定條件下觸發。
- **Triton-based compilation**: 使用 Triton JIT compile 每個 kernel，不需要 CUDA graph。更靈活且跨平台，但 per-kernel launch overhead 仍存在。
- **Eager mode**: 無 CUDA graph。最通用，啟動最快，但 GPU 利用率最低。

**何時可以借用**: 任何需要在 GPU 上重複執行相似但 shape 略有不同的計算時。不只是 LLM inference — 也可以是 video encoding、image processing pipeline、科學計算。

**注意事項**: Piecewise graph 的 segment 分割點選擇是藝術。太多 segment → 失去 CUDA graph 的好處；太少 → 有些 dynamic shape 操作被包進 graph 導致 capture 失敗。vLLM 的 Dynamo-based approach 自動決定分割點，而 breakable 方案則透過 `@eager_break_during_capture` decorator 讓開發者手動標記。

---

## Pattern 4: Single Writer Multi-Reader 的 Engine Frontend/Core 分離

**是什麼**: Engine Frontend（AsyncLLMEngine）與 EngineCore 分屬不同執行緒/進程，透過序列化協定（msgspec msgpack）通訊。一個 EngineCore 可以服務多個 frontend，每個 frontend 獨立處理 input/output。

**為什麼有效**: Llama-3.1-405B 這樣的大模型，一個 inference step 需要數十毫秒。如果 EngineCore 的排程循環跟 HTTP 處理在同一個 event loop 中，HTTP request handling 會被 blocking 而無法 accept 新請求。Frontend/Core 分離讓 HTTP server（async）和 inference engine（sync worker）各自在適合的執行緒模型下運行。

**關鍵實作**:
- EngineCoreRequest/EngineCoreOutput 通訊協定: [`vllm/v1/engine/__init__.py:81-198`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/engine/__init__.py#L81-L198)
- EngineCoreClient 抽象: [`vllm/v1/engine/core_client.py`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/engine/core_client.py)
- Batch queue pipeline: [`vllm/v1/engine/core.py:469-585`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/engine/core.py#L469-L585)

**替代方案**:
- **同進程同步模型**: scheduler 跟 HTTP server 在同一個 thread 中輪流執行。實作最簡單，但無法利用多核心，且 scheduler 執行時 HTTP 無法 accept。
- **同進程 async 模型**: scheduler 和 HTTP handler 在 async event loop 中共存，但 scheduler 的計算（特別是 GPU 操作）會 blocking event loop。

**何時可以借用**: 任何「有狀態的核心處理引擎 + 無狀態的 API 層」的架構 — 遊戲 server、trading engine、資料庫 query processor。

**注意事項**: Frontend/Core 之間的通訊序列化是效能瓶頸。vLLM 選擇 msgspec（msgpack）而非 JSON 是刻意的 — 在高頻次（每個 inference step 都有資料交換）場景，序列化速度的差異顯著。

---

## Pattern 5: 三層委派的 KV Cache 管理

**是什麼**: KV cache 的管理不是單一類別，而是三層委派架構：`KVCacheManager → KVCacheCoordinator → SingleTypeKVCacheManager(s) → BlockPool`。

**為什麼有效**: 這種分層設計讓不同層級關注不同議題：
- `KVCacheManager`（最上層）: 負責單個 request 的生命週期 — allocate、free、prefix cache lookup
- `KVCacheCoordinator`（中間層）: 管理多個 KV cache type 的協調（例如 hybrid attention 模型需要不同 block size 的多組 KV cache）
- `SingleTypeKVCacheManager`（實作層）: 處理單一 block type 的分配細節
- `BlockPool`（底層）: 實際管理 GPU 記憶體中的 block

當模型只有一種 attention type（如標準 Llama），KVCacheManager 直接對應 SingleTypeKVCacheManager；當模型有 hybrid attention（如全注意力 + sliding window + Mamba），Coordinator 負責協調不同 block size 的分配。

**關鍵實作**:
- KVCacheManager: [`vllm/v1/core/kv_cache_manager.py:110`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/kv_cache_manager.py#L110)
- KVCacheCoordinator: [`vllm/v1/core/kv_cache_coordinator.py`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/kv_cache_coordinator.py)
- BlockPool: [`vllm/v1/core/block_pool.py:149`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/block_pool.py#L149)
- LCM block alignment: [`vllm/v1/core/kv_cache_coordinator.py:471-477`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/kv_cache_coordinator.py#L471-L477)

**替代方案**:
- **單一管理器**: 所有邏輯寫在一個類別。適用於只有一種 KV cache 類型的簡單模型，但在 hybrid 模型時會變成一個巨無霸類別。
- **插件式 hook**: 在 scheduler 中直接 callback 分配邏輯。更靈活但責任不清，難以測試。

**何時可以借用**: 任何需要管理多種資源類型但不確定未來擴充方向的系統。

**注意事項**: 三層委派增加了 tracing 和理解難度。在 vLLM 的場景中，這種複雜度是 justified 的（因為支援 200+ 模型架構，包含 hybrid attention），但在只有單一資源類型的系統中，單層管理器就夠了。

---

## Pattern 6: 自由串列的 O(1) 中間刪除

**是什麼**: vLLM 不使用 Python 的 `list` 或 `collections.deque` 來管理 free block queue，而是手動實作了一個雙向鏈結串列（`FreeKVCacheBlockQueue`），支援 O(1) 的中間刪除、O(1) 的頭尾新增移除。

**為什麼有效**: Block pool 的 free queue 需要在任意位置刪除一個 block（block 可能因為 cache hit 而從中間被移除），Python 的 list 的中間刪除是 O(n)，deque 不支援中間刪除。自製的雙向鏈結串列用 `prev_free_block` / `next_free_block` 指標實現，避免了 Python object 的 allocation overhead。

**關鍵實作**:
- [`vllm/v1/core/kv_cache_utils.py:164-372`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/kv_cache_utils.py#L164-L372)

**替代方案**:
- **Python list + lazy invalidation**: 不真的刪除，而是標記為 invalid，分配時跳過。讀取時 O(1)，但 invalid 的 entries 累積後需要 GC。適合刪除頻率低的場景。
- **Linked list via Python objects**: 每個節點是 Python object。簡單但 GC overhead 大。

**何時可以借用**: 任何需要高效中間刪除的 collection — LRU cache、memory pool、task queue 中的 priority reordering。

**注意事項**: 這種模式不適用於多線程場景（Python GIL 保護不夠）。vLLM 的 scheduler 是單線程的，因此沒有 thread safety 問題。

---

## ML 工程品味的觀察

vLLM 的程式碼有一些值得注意的設計傾向：

1. **對 abstraction 的態度務實**: 不迷信 design pattern。例如 kv_cache 管理雖然有三層委派（KVCacheManager → Coordinator → BlockPool），但每層的介面都很小且具體。沒有不必要的抽象工廠或策略模式。

2. **對效能極度偏執**: 例如 FreeKVCacheBlockQueue 寧可自己實作雙向鏈結串列也不使用 Python built-in 的 deque，因為需要 O(1) 中間刪除。這種偏執只在 profiling 後 justified 的地方出現。

3. **對型別系統的謹慎**: 使用 `msgspec.Struct` 而非 `dataclass` 或 `pydantic.BaseModel` 來定義 protocol — 因為序列化效能差異。這在「每個 step 都需要序列化大量資料」的場景下是正確的選擇。

4. **V1 重構的勇氣**: 核心開發者選擇完全重寫 engine core（從 v0 到 v1），捨棄了與舊 API 的完全相容，這在 80k+ stars 的專案中需要巨大的工程信心。V1 重構把 engine core 從「monolithic class 內混合 schedule + execute + client interaction」改為「frontend/EC 分離 + 明確的序列化介面」，本質上是從 shared-state concurrency 轉向 message-passing。

---

## 深入追蹤：Unified num_computed_tokens 模型

**這是 vLLM 連續 batch 排程的核心抽象，值得深入說明。**

傳統的 LLM serving 系統區分 prefill 階段和 decode 階段，各有獨立的排程邏輯。vLLM V1 scheduler 不做這種區分 — 每個 request 只有兩個計數器：

- `num_tokens_with_spec`：這個 request 在這次排程中總共需要處理多少 token（prefill 時等於 prompt 長度 + spec decode tokens；decode 時等於 1 + spec decode tokens）
- `num_computed_tokens`：已經完成了多少 token 的計算

每次 `schedule()` 時，scheduler 嘗試讓 `num_computed_tokens` 追上 `num_tokens_with_spec`。兩者的差距 `num_new_tokens` 就是這次需要處理的 token 數量。

**為什麼有效**:
- 消除 prefill 和 decode 的二元區分 → 同一個 schedule 步驟可以同時處理正在 prefill 的 request 和正在 decode 的 request（真正的 continuous batching）
- Chunked prefill 是這個模型的自然結果：如果 `num_new_tokens` 超過 token budget，就只處理 budget 個 token，剩下的下次再做
- Speculative decoding 也自然地融入：decode 時 `num_tokens_with_spec` = 1 + spec_length，prefill 時等於 prompt 長度

**關鍵實作**:
- Token budget 計算: [`vllm/v1/core/sched/scheduler.py:385-395`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/sched/scheduler.py#L385-L395)
- Post-schedule 更新: [`vllm/v1/core/sched/scheduler.py:951-991`](https://github.com/vllm-project/vllm/blob/0902d8e/vllm/v1/core/sched/scheduler.py#L951-L991)

**何時可以借用**: 任何需要「多種不同處理階段但共用同一套排程邏輯」的系統。不只是 LLM — pipeline processing、data streaming、batch job scheduler 都能採用。

**不適用的情境**: 如果各階段的資源需求差異極大（例如 prefill 需要 100x 計算但 decode 只需要 1x），統一的 token budget 模型可能無法有效分配。此時可能需要 weighted budget。
