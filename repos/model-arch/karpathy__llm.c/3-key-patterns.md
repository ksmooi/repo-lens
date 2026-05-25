---
repo: karpathy/llm.c
file: 3-key-patterns
studied_at: 2025-06-26
commit_sha: f1e2ace
---

# llm.c · 值得偷學的設計

## Pattern 1: 三層快取提示（Cache Hint）的細粒度管理

**是什麼**：llm.c 的 CUDA kernel 不只用 `__syncthreads()`，還對每筆 global memory 存取仔細選擇 cache 策略 — `load128cs`（streaming load, bypass L1）、`store128cs`（streaming store, bypass L1）、`store128cg`（bypass L1, keep L2）、`store128`（保留 L1+L2）。

**為什麼有效**：GPU 的 L1 cache 是有限資源（每 SM 通常 48-128KB）。對「只讀一次」的資料（attention 的 QKV 輸入、softmax 後的 att matrix）使用 streaming load，可以避免把這些不會再用的資料擠進 L1，留出空間給更常重用的資料（如 weight matrix）。這對 memory-bound 的 transformer training 特別重要 — 每個 kernel 的 L1 hit rate 直接影響頻寬利用率。

**程式碼位置**：
- `load128cs`：[`cuda_utils.cuh:54`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/cuda_utils.cuh#L54-L61)
- `store128cs`：[`cuda_utils.cuh:63`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/cuda_utils.cuh#L63-L70)
- `store128cg`：[`cuda_utils.cuh:72`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/cuda_utils.cuh#L72-L76)

**使用模式**：
| Cache 提示 | 使用時機 | 使用位置 |
|---|---|---|
| `__ldcs` (streaming) | attention softmax input、encoder embedding lookup、GELU backward input | `attention.cuh`、`encoder.cuh`、`gelu.cuh` |
| `__stcs` (streaming) | softmax backward in-place 覆寫、GELU backward 輸出 | `attention.cuh`、`gelu.cuh` |
| `__stcg` (L2 only) | layernorm backward 的 dinp 寫回 | `layernorm.cuh` |
| 普通 store | GELU forward 輸出（下層 matmul 會讀） | `gelu.cuh` |

**替代方案**：
- **不管理 cache（naive CUDA）**：`float4` 直接讀寫，交給編譯器和硬體自動管理。簡單但浪費 cache。
- **cache 最佳化編譯器旗標**：`-Xptxas -dlcm=ca`（L1+ L2）或 `-Xptxas -dlcm=cg`（L2 only）。全局統一策略，粒度粗。
- **`__ldg()` (read-only cache)**：自動使用 `ldg` 指令進 texture cache，但只適用於 forward-pass 的唯讀權重。

llm.c 的作法在所有三種方案中提供了最細粒度的控制，代價是程式碼可讀性降低（需要記住每個指標的存取模式）。

**何時可以借用**：
- 當你的 CUDA 程式達頻寬瓶頸（ncu/size 分析確認），且不同 buffer 的重用性差異明顯。
- 適合 kernel 數多、每個 kernel 存取模式明確的系統（transformer 訓練是典型場景）。

**不適用的情境**：
- kernel 很少（1-3 個），不值得做 micro-optimization。
- 使用 PyTorch/JAX 等框架，無法直接控制 CUDA 記憶體指令。

---

## Pattern 2: 反向迭代（Reverse Grid Iteration）的 Cache 親和力

**是什麼**：llm.c 的 multiple kernel 以**反向 block 索引**遍歷張量，即 `blockIdx.x = gridDim.x - i - 1`，而非預設的 `blockIdx.x = i`。這樣做讓後一個 kernel 先處理前一個 kernel 剛寫入的記憶體區域，提高 L2 cache hit rate。

**為什麼有效**：在 GPU 的 L2 cache 中，最近寫入的資料通常還在 cache 中（若尚未被驅逐）。如果 kernel A 從位置 N-1 開始寫入、kernel B 也從 N-1 開始讀取，B 的 L2 hit 機率比從 0 開始高。這對 transformer training 的 backward pass 特別有意義 — forward 從位置 0 到 N-1 計算，backward 從 N-1 到 0 讀取，如果 backward 也從 0 開始，那它先讀的是 forward 最早寫的資料（已被驅逐）。

**程式碼位置**：
- [`attention.cuh:100`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/attention.cuh#L100) — attention softmax kernel：`idx = (gridDim.x - blockIdx.x - 1) * num_warps + warp_id`
- [`fused_classifier.cuh:76`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/fused_classifier.cuh#L76) — fused classifier kernel：`idx = gridDim.x - (blockIdx.x + 1)`
- [`layernorm.cuh:158`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/layernorm.cuh#L158) — fused residual forward kernel：`ix = gridDim.x - (blockIdx.x + 1)`

**替代方案**：
- **不做優化**：直接用 `blockIdx.x`。簡單，但 L2 miss 率約高 10-20%。
- **軟體 prefetch**：在 kernel 中提前發出 `__isGlobal()` + `__ldg()`，讓硬體先行載入。但需手動計算 prefetch distance，複雜度高。
- **增大 L2 分區**：部分 GPU 支援 L2 分割區設定，但需要 root 權限且影響系統中所有程式。

**何時可以借用**：
- 你的 pipeline 中 kernel 是串聯的（A→B→C），且每個 kernel 都遍歷整個張量。
- 特別是 training loop 中 forward/backward 成對的情況。

**不適用的情境**：
- Kernel 之間無依賴關係（可以並行）。
- 張量大小遠大於 L2 cache（>40MB for H100），反向迭代無明顯效果。

---

## Pattern 3: Online Softmax（單次 Pass 的 Softmax）

**是什麼**：傳統 softmax 需要兩次遍歷（第一次找 max，第二次算 exp/sum/normalize），online softmax 在一次遍歷中同時維護 `maxval` 和 `sum`，完成「更新 max → 更新 exp → 更新 sum」的增量計算。

**為什麼有效**：一次遍歷省一半的 global memory 讀取。對 attention 的 softmax（T=1024, NH=12, B=64 → 約 3M 個 softmax 實例）來說，可以省下可觀的頻寬。更重要的是，online softmax 讓 softmax 的 backward 也可以只用一次遍歷（反向 + in-place），進一步減少存取。

**程式碼位置**：
- Online softmax forward：[`attention.cuh:85-150`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/attention.cuh#L85-L150)
- Fused classifier 的 online softmax：[`fused_classifier.cuh:19-63`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/fused_classifier.cuh#L19-L63)

**關鍵程式碼**：
```c
// Online softmax core (attention.cuh:85-150)
float maxval = -INFINITY;
float sum = 0.0f;
for (int i = 0; i < V; i += x128::size) {
    float x = load128cs(&in[t * V + i]);  // streaming load
    float new_max = fmaxf(maxval, x);
    sum = sum * expf(maxval - new_max) + expf(x - new_max);
    maxval = new_max;
}
float inv_sum = __frcp_rn(sum);
for (int i = 0; i < V; i += x128::size) {
    float x = load128cs(&in[t * V + i]);
    store128cs(&out[t * V + i], expf(x - maxval) * inv_sum);
}
```

**替代方案**：
- **標準兩次遍歷**：`maxval = reduce_max(in)` → `sum = reduce_sum(exp(in - maxval))` → `out = exp(in - maxval) / sum`。簡單直觀，但讀兩次記憶體。
- **Warp-level softmax**（cuDNN / FlashAttention 方式）：在 warp 內做 softmax，利用 shuffle 指令減少 shared memory 使用。適合 block size ≤ 32 的情況。
- **`cudnnSoftmaxForward`**：NVIDIA 提供的 library function，封裝良好但自訂性低。

**何時可以借用**：
- 你需要在自己的 CUDA kernel 中實作 softmax 或類似的 normalize operation。
- 資料量夠大（vocab_size > 256）讓兩次遍歷的頻寬浪費可見。

---

## Pattern 4: 確定性反向傳播 — 用 CPU 預處理取代 atomicAdd

**是什麼**：在 wte（word token embedding）的 backward 中，同一個 token 的梯度需要匯總到同一個 embedding 向量上。naive 做法是 `atomicAdd`（非確定性），llm.c 則在 CPU 上對 `(batch, token_id, position)` 做桶排序分類，讓每個桶（一個 token_id 的 embedding gradient）被唯一一個 GPU block 處理，保證確定性。

**為什麼有效**：GPGPU 的浮點運算是非結合性的 — `a + b + c` 和 `a + c + b` 可以產生不同的結果（特別是在混合精度下）。使用 `atomicAdd` 時，不同執行緒的加法順序不同，導致每次執行結果稍有不同。llm.c 的做法保證 100% 可重現。

**程式碼位置**：
- WTE backward：[`encoder.cuh:47-117`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/encoder.cuh#L47-L117)
- CPU 桶排序構建：[`encoder.cuh:188-206`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/encoder.cuh#L188-L206)
- 桶大小降序排列：[`encoder.cuh:201-206`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/encoder.cuh#L201-L206)

**流程**：
1. Forward pass 時記錄每個 `(batch, token_id)` 的出現位置
2. CPU 將所有位置依 token_id 分桶（bucket sort）
3. 桶大小降序排序（最大桶先被 GPU 處理，提高佔用率）
4. `cudaMemcpyAsync` 將桶資訊送到 GPU
5. 每個 GPU block 處理一個桶（一個 token embedding 的所有梯度）

**替代方案**：
- **`atomicAdd`**：最簡單，非確定性但通常「足夠接近」。PyTorch 就是這樣做的。
- **`__umul64hi` 自旋鎖 + 共享記憶體歸約**：在 shared memory 中用 warp-level reduction 後只用一次 `atomicAdd`。減少但未消除非確定性。
- **Warp-aggregated atomic**：用 warp shuffle 先做 warp 內歸約，再用 `atomicAdd`。減少競爭但仍是 non-deterministic。

**何時可以借用**：
- 你的 embedding layer 需要精確可重現的結果（測試、科學計算）。
- token embedding 的 vocabulary 大小適中（< 100K），桶數量可控。
- 你已經有 CPU 端可以執行預處理（對 GPU-only pipeline 不適用）。

**不適用的情境**：
- Vocab 極大（> 1M），桶數量超過 GPU block 上限。
- 每個 batch 的 token 分布極不均勻（少數 token 出現數千次），桶大小差異過大導致 GPU 利用率不佳。

---

## Pattern 5: 線性層 backward 的「先 bias 再 dinp」順序

**是什麼**：llm.c 的 `matmul_backward` 先計算 `dbias` 再設 `dbias = NULL` 讓 cuBLASLt 不要重複計算 bias gradient，然後才計算 `dinp`（覆寫）和 `dweight`（累加）。

**為什麼有效**：cuBLASLt 的 epilogue 可以同時做 `GEMM + bias grad`（`CUBLASLT_EPILOGUE_BGRADB`），但一次只能計算一個。如果不先取出 bias gradient，cuBLASLt 的 epilogue 在做 `dinp` 時可能順便也算了一次 bias（浪費計算且引起 double counting）。先計算 `dbias` 再設為 NULL，讓後續的 GEMM 只做矩陣乘法。

**程式碼位置**：[`matmul.cuh:244-290`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/matmul.cuh#L244-L290)

```c
// Step 1: compute dbias first, then set to NULL
matmul_backward_bias_kernel9<<<grid, block, 0, stream>>>(dbias, dinp, ...);
dbias = NULL;  // prevent cuBLASLt from recomputing

// Step 2: the actual gradient computation with cuBLASLt
matmul_backward_cublaslt(dinp, dweight, ...);  // dinp overwrites, dweight accumulates
```

**替代方案**：
- **分離的兩次 cuBLASLt 呼叫**：一次做 `dinp`（用 `BGRADB` epilogue），一次做 `dweight`。多一次 kernel launch。
- **自訂 backward kernel**：完全不用 cuBLASLt，自己寫 GEMM backward。控制力最強但開發成本高。

**何時可以借用**：
- 你封裝了 cuBLAS/cuBLASLt 並且需要計算完整的 backward（dinp + dweight + dbias）。
- 使用 cuBLASLt epilogue 做 fused 操作時。

---

## Pattern 6: All-at-once Memory Allocation

**是什麼**：llm.c 的所有參數、激活值、梯度都在訓練開始時被一次性分配為連續的 device memory 區塊，然後用 `malloc_and_point_*` 函式把每個 tensor 的指標指向正確的 offset。

**為什麼有效**：
1. 減少記憶體碎片（GPU 驅動的 memory management 在大量小 allocation 下會產生碎片）。
2. 簡化 checkpoint I/O（一次大塊的 `cudaMemcpy` 比多次小塊快）。
3. 記憶體用量清晰可計算（所有 tensor 的 size 在 `fill_in_*_sizes` 中明確定義）。

**程式碼位置**：
- Parameter sizing：[`train_gpt2.cu:87-168`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L87-L168)
- Activation sizing：[`train_gpt2.cu:170-322`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L170-L322)
- Allocation：[`train_gpt2.cu:324-428`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L324-L428)

**替代方案**：
- **逐個分配**（PyTorch 方式）：每個 tensor 獨立 `cudaMalloc`。簡單但碎片多、checkpoint 慢。
- **Pool allocator**（CUDA 11+ `cudaMallocAsync`）：驅動自動管理記憶體池，減少碎片。但較新 GPU 才支援。
- **Unified Memory**：`cudaMallocManaged`，讓驅動處理 page migration。簡單但 implicit migration 有開銷。

**何時可以借用**：
- 你的 CUDA 程式 tensor 數量固定、size 可在初始化時計算（訓練模型是典型場景）。
- 需要高效 checkpoint/save 支援。

---

## 工程品味的觀察

1. **對 abstraction 的態度**：Karpathy 在 llm.c 中展現了對 abstraction 的極簡主義。CPU 版所有 forward/backward 在同一個檔案（1182 行），不隱藏任何細節。GPU 版雖然拆成 14 個 kernel 檔案，但每個檔案只包含一個 layer 的 forward + backward，沒有 class hierarchy，沒有 virtual dispatch。這種「檔案即模組」的組織方式，比 OOP 抽象更容易理解單一 operation。

2. **對效能與可讀性的取捨**：在關鍵路徑（attention、layernorm）上，程式碼做了大量 micro-optimization：128-bit 向量化、反向迭代、streaming cache hints、shared memory 預載入。但在非關鍵路徑（GELU、encoder lookup）上，kernel 程式碼保持簡單。這是一種「80/20 法則」的工程應用 — 20% 的程式碼消耗 80% 的時間，把精力花在那 20% 上。

3. **對確定性的重視**：llm.c 在三個地方（wte backward、wpe backward、stochastic rounding）做了額外工作來保證結果確定性。這對教學 repo 特別重要 — 每次跑同一指令應該得到完全相同的 loss 曲線，否則學習者無法區分「我的修改是否對了」和「隨機波動導致不一樣」。

4. **優雅降級模式**：多處使用執行期偵測 + fallback 機制（cuBLASLt epilogue 不支援 → 分離 GELU kernel、shared memory 太大 → 簡化 kernel、cuDNN 不可用 → 手工 attention）。這讓同一個 binary 能在不同世代的 GPU 上運行，而不需要編譯多個版本。
