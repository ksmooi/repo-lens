---
repo: karpathy/llm.c
file: 1-architecture
studied_at: 2025-06-26
commit_sha: f1e2ace
---

# llm.c · 架構

## 系統高層圖

llm.c 的架構可以分為三層：**訓練主檔**（train_gpt2.c / .cu）、**CUDA kernel 庫**（llmc/），以及**基礎設施元件**（dataloader、tokenizer、scheduler 等）。

```mermaid
flowchart TB
    subgraph "Train Scripts"
        CPU[train_gpt2.c<br/>CPU 參考實作<br/>1182 lines]
        GPU[train_gpt2.cu<br/>GPU 主要訓練<br/>1904 lines]
        PY[train_gpt2.py<br/>PyTorch 參考<br/>860 lines]
    end

    subgraph "CUDA Kernel Library (llmc/)"
        ATT[attention.cuh<br/>Multi-head Attention]
        LN[layernorm.cuh<br/>LayerNorm + Residual]
        ENC[encoder.cuh<br/>Token + Position Embedding]
        MUL[matmul.cuh<br/>GEMM + GELU Fusion]
        FC[fused_classifier.cuh<br/>Softmax + Cross-Entropy]
        ADAM[adamw.cuh<br/>AdamW Optimizer]
        GN[global_norm.cuh<br/>Gradient Norm]
        GELU[gelu.cuh<br/>GELU Activation]
        ZERO[zero.cuh<br/>ZeRO + Multi-GPU]
    end

    subgraph "Infrastructure"
        DL[dataloader.h<br/>Data Loading / Shuffle]
        SCH[schedulers.h<br/>LR Schedules]
        TOK[tokenizer.h<br/>GPT-2 BPE Tokenizer]
        LOG[logger.h<br/>Metrics Logger]
        SAM[sampler.h<br/>Token Sampling]
        MFU[mfu.h<br/>Model FLOPS Utilization]
    end

    CPU -->|OpenMP parallel| CPU
    GPU -->|calls| ATT
    GPU -->|calls| LN
    GPU -->|calls| ENC
    GPU -->|calls via cuBLASLt| MUL
    GPU -->|calls| FC
    GPU -->|calls| ADAM
    GPU -->|calls| GN
    GPU -->|calls| GELU
    GPU -->|calls| ZERO
    GPU -->|uses| DL
    GPU -->|uses| SCH
    GPU -->|uses| TOK
    GPU -->|uses| LOG
    GPU -->|uses| SAM
    GPU -->|uses| MFU
    PY -->|generates| CKPT[Checkpoint .bin files]
    CKPT -->|loaded by| CPU
    CKPT -->|loaded by| GPU

    style CPU fill:rgba(8,51,68,0.4),stroke:#22d3ee
    style GPU fill:rgba(8,51,68,0.4),stroke:#22d3ee
    style PY fill:rgba(8,51,68,0.4),stroke:#22d3ee
```

**圖意說明**：上圖展示 llm.c 的三層架構。訓練主檔（train_gpt2.c / .cu / .py）是三個平行實作，共用相同的超參數和資料格式。GPU 版本引用了 llmc/ 下 14+ 個 CUDA kernel 模組，並依賴 cuBLASLt 處理所有線性層矩陣乘法。基礎設施元件（dataloader、scheduler 等）被 CPU 和 GPU 版本共用。PyTorch 版本負責產生 checkpoint .bin 檔，供 C/CUDA 版本載入驗證。

### 資料流圖

```mermaid
flowchart LR
    subgraph "Training Step (single iteration)"
        DL1[DataLoader] -->|tokens ids<br/>int[B,T+1]| ENC[Encoder<br/>wte + wpe]
        ENC -->|encoded<br/>float[B,T,C]| LAYER[Transformer Block<br/>× num_layers]
        LAYER -->|residual<br/>float[B,T,C]| LNF[Final LayerNorm]
        LNF -->|normalized<br/>float[B,T,C]| LM[LM Head<br/>= wte.T]
        LM -->|logits<br/>float[B,T,Vp]| CLS[Fused<br/>Classifier]
        CLS -->|loss<br/>float[]| BACKWARD[Backward Pass<br/>= reverse forward]
        BACKWARD -->|gradients<br/>float[n_params]| CLIP[Gradient<br/>Clipping]
        CLIP -->|clipped grads| ADAMW[AdamW<br/>Update]
        ADAMW -->|updated params| ENC
    end

    subgraph "CUDA Kernels used"
        ENC -.->|encoder_forward| KERN1[llmc/encoder.cuh]
        LAYER -.->|layernorm_forward<br/>matmul_forward<br/>attention_forward<br/>gelu_forward<br/>residual_forward| KERN2[llmc/layernorm.cuh<br/>llmc/matmul.cuh<br/>llmc/attention.cuh<br/>llmc/gelu.cuh]
        CLS -.->|fused_classifier| KERN3[llmc/fused_classifier.cuh]
        ADAMW -.->|adamw_kernel3| KERN4[llmc/adamw.cuh]
        CLIP -.->|global_norm_squared| KERN5[llmc/global_norm.cuh]
    end

    style DL1 fill:rgba(76,29,149,0.4),stroke:#a78bfa
    style KERN1 fill:rgba(6,78,59,0.4),stroke:#34d399
    style KERN2 fill:rgba(6,78,59,0.4),stroke:#34d399
    style KERN3 fill:rgba(6,78,59,0.4),stroke:#34d399
    style KERN4 fill:rgba(6,78,59,0.4),stroke:#34d399
    style KERN5 fill:rgba(6,78,59,0.4),stroke:#34d399
```

**圖意說明**：一個 training step 的資料流程。編碼器將 token ids 轉為 embedding 向量，經過 L 層 transformer block 後，通過 final layernorm 和 LM head (weight-tied with wte) 得到 logits。Fused classifier 同時計算 cross-entropy loss 並初始化 dlogits。反向傳播依 forward 的逆序進行，最後以 AdamW 更新參數。圖右側標註每個步驟使用的 CUDA kernel 來源。

## GPU 訓練主程式 (train_gpt2.cu) 結構

train_gpt2.cu（1904 行）是整個 repo 的中樞，分為以下區段：

| 區段 | 行號 | 內容 |
|---|---|---|
| Includes + 全域變數 | 1–82 | 引入所有 llmc/ kernel + 設定 main_stream |
| 模型定義 (GPT2Config, ParameterTensors) | 87–168 | 12-layer GPT-2 的 16 個參數張量的 size 與佈局 |
| 激活值定義 (ActivationTensors) | 170–322 | 21 個激活值張量的 size 與指標指向 |
| 記憶體配置 | 324–428 | 連續 device memory 分配 + 各 tensor 指標指向 |
| Checkpoint I/O | 430–642 | checkpoint 讀寫、權重初始化 |
| Forward pass ([`train_gpt2.cu:644`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L644-L755)) | 644–755 | encoder → L 層 transformer → final LN → logits |
| Validation | 757–786 | HellaSwag 等多選題評估 |
| Backward pass ([`train_gpt2.cu:788`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L788-L973)) | 788–973 | 完整反向傳播 + 梯度累加 |
| Gradient norm + AdamW update | 992–1153 | gradient clipping + optimizer step |
| GPU 初始化 (`common_start`) | 1170–1206 | cuBLAS/cuBLASLt/cuDNN 初始化 + device query |
| **main() 訓練迴圈** ([`train_gpt2.cu:1419`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1419-L1904)) | 1419–1904 | 參數解析 → 初始化 → for each step { val, sample, train, log } |

### main() 訓練迴圈 ([`train_gpt2.cu:1419`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1419-L1904))

```python
main():
 |-- CLI 參數解析（32+ 個 -x 參數）
 |-- multi_gpu_config_init + common_start(cuBLAS/cuDNN)
 |-- 計算 grad_accum_steps = total_batch_size / (B * T * num_processes)
 |-- 建構 GPT2 model（resume / .bin / descriptor 三種模式）
 |-- init DataLoader + Tokenizer + LR Scheduler
 |-- init AdamW state (m, v)
 |
 |-- for step = 0 .. train_num_batches:
 |     |-- 每 val_loss_every step: HellaSwag eval
 |     |-- 每 sample_every step: generate text
 |     |-- 每 checkpoint_every step: write checkpoint
 |     |
 |     |-- ## TRAINING SECTION
 |     |-- for micro_step = 0 .. grad_accum_steps:
 |     |     |-- dataloader_next_batch()
 |     |     |-- gpt2_forward(x, y)        # 累加 loss
 |     |     |-- gpt2_backward_and_reduce() # 累加 gradients
 |     |-- gradient clipping + gpt2_update(AdamW)
 |     |-- logging + MFU estimation
```

## 關鍵設計決策與取捨

### 決策 1：cuBLASLt 為基礎 + 自訂 kernel 為輔

**選擇**：所有線性層（QKV projection、attention output、MLP 的 fc/fcproj、lm head）都透過 cuBLASLt API 執行，只有 layernorm、attention softmax、activation、classifier、optimizer 是用自訂 CUDA kernel。

**理由**：cuBLASLt 的矩陣乘法經過 NVIDIA 深度優化（tensor core、autotuning、epilogue fusion），自己寫一個能匹配的 GEMM kernel 難度極高。而 layernorm 等是 memory-bound 操作，自訂 kernel 可以針對特定資料佈局（B×T×C）和精度做優化。

**取捨**：cuBLASLt 是 proprietary library，不是所有平台都有。對比之下，CPU 版的純 C 實作完全自含，可移植性最高。

**位置**：[`matmul.cuh:109`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/matmul.cuh#L109-L228)

### 決策 2：Activation recomputation（0/1/2）

**選擇**：GPU 版支援三種 activation recompute 層級 — `0`（全存）、`1`（重算 GELU）、`2`（重算 GELU + LayerNorm）。

**理由**：GPT-2 124M 的 activation memory 約爲 (B×T×C)×L×ops ~ 數 GB。以算力換取 VRAM，讓相同 GPU 可以跑更大的 batch 或更長的序列。

**取捨**：recompute=2 會增加 ~30% 的 forward 運算量（因為 backward 中要重算 GELU 和 LN），但 VRAM 節省可能超過 40%。對 compute-bound 的訓練來說是合理的取捨。

**位置**：[`train_gpt2.cu:1406`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1406)

### 決策 3：Fused operations 集中在三個點

**選擇**：三個關鍵融合：
1. `fused_residual_forward5` — residual add + layernorm 融合成單一 kernel
2. `fused_classifier` — softmax + cross-entropy + dlogits 初始化融合
3. cuBLASLt epilogue — GEMM + GELU activation 融合（H100+）

**理由**：這三個點都是 memory-bound 操作，融合可以減少 global memory 讀寫次數。`fused_residual_forward5` 一次省掉一個 full (B,T,C) tensor 的寫入，對大模型訓練的頻寬節省顯著。

**取捨**：融合 kernel 的 shared memory 用量更大，老 GPU（compute capability < 7.5）可能不支援。程式碼中做了自動 fallback 處理。

**位置**：[`layernorm.cuh:142`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/layernorm.cuh#L142-L219)

### 決策 4：Single CUDA stream（main_stream）

**選擇**：整個訓練使用單一 CUDA stream，而非多 stream 並行。

**理由**：簡化同步邏輯，所有 cudaMemcpy、kernel launch、NCCL ops 都在同一個 stream 上。對於管理多個 concurrent kernel 的複雜度，Karpathy 選擇「先求正確再求快」。

**取捨**：單一 stream 無法 pipeline kernel execution 和 data transfer。但考慮到訓練 loop 中所有 kernel 依賴前一個的產出（forward → backward → update），多 stream 的效益有限。

**位置**：[`train_gpt2.cu:1183`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1183)

### 決策 5：Weight tying + padded vocab

**選擇**：LM head 直接使用 wte 的權重（weight tying），且 vocab size 從 50257 填充至 50304（128 的倍數）。

**理由**：Weight tying 是 GPT-2 論文標準做法。Vocab padding 是為了 CUDA 的 warp/memory 對齊 — 50304 是 128 的倍數，讓 softmax kernel 的循環不需要邊界檢查。

**取捨**：padded vocab 浪費約 (50304-50257) × 768 × 4 bytes ≈ 144KB 的記憶體，但換來所有 kernel 的一致性（無需特殊處理 boundary）。

**位置**：[`train_gpt2.cu:583`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L583)

### 決策 6：Stochastic rounding for mixed precision

**選擇**：在 FP32→BF16/FP16 轉換時使用 stochastic rounding，而非簡單的 truncation 或 round-to-nearest-even。

**理由**：Stochastic rounding 可以減少低精度訓練的數值偏差，特別是在 gradient 很小的時候。llm.c 用 SquirrelNoise5 雜湊函數產生可重現的亂數種子，確保每次運行的結果確定性。

**取捨**：引入隨機性可能讓偵錯更困難，但 llm.c 用了確定性 seed（每個 thread + 每個參數 index），所以結果仍然是確定性的。

**位置**：[`cuda_utils.cuh:269`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/cuda_utils.cuh#L269-L284)

### 決策 7：手寫 vs cuDNN attention

**選擇**：支援兩種 attention 路徑 — 編譯時透過 `#ifdef ENABLE_CUDNN` 切換，預設關閉。

**理由**：cuDNN flash attention（`ENABLE_CUDNN=1`）的 memory 用量是 O(B×NH×T)，而手工實作（非 flash）是 O(B×NH×T×T)，因為需要實體化 T×T 的 attention matrix。

**取捨**：cuDNN 顯存更省但編譯時間大幅增加（從幾秒到一分鐘），且需要安裝 cuDNN frontend。手工版在 T=1024 時 memory 還可接受，但 T 更長時就需要 cuDNN 了。

**位置**：[`train_gpt2.cu:52-58`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L52-L58)

## CUDA Kernel 目錄架構

llmc/ 目錄下的 23 個檔案可劃分為三層：

| 層級 | 檔案 | 職責 |
|---|---|---|
| **基礎設施** | `cuda_common.h`、`cuda_utils.cuh`、`cublas_common.h` | 128-bit 向量化載入（Packed128）、warp/block 歸約、cuBLASLt handle、隨機捨入 |
| **Kernel 實作** | `attention.cuh`、`matmul.cuh`、`layernorm.cuh`、`encoder.cuh`、`gelu.cuh`、`fused_classifier.cuh`、`adamw.cuh`、`global_norm.cuh` | 每個運算的 forward + backward CUDA kernel |
| **訓練基礎設施** | `dataloader.h`、`schedulers.h`、`tokenizer.h`、`sampler.h`、`logger.h`、`rand.h`、`mfu.h`、`outlier_detector.h`、`zero.cuh` | 資料載入、LR schedule、tokenizer、MFU 計算、多 GPU |

## 設計哲學總結

Karpathy 在 llm.c 中的架構選擇反映了一個明確的偏好：**在關鍵路徑上追求效能，在其他地方保持簡單**。

具體來說：
- **attention + layernorm + classifier**：手寫高度優化的 CUDA kernel（向量化、cache hints、反向迭代）
- **矩陣乘法**：外包給 cuBLASLt（不自己寫 GEMM）
- **wte backward**：用 CPU 桶排序避免 atomicAdd（確定性優先）
- **AdamW update**：自訂 kernel（因為 optimizer 是 memory-bound，不是 compute-bound）
- **Data loading**：純 C 實作（不需要 GPU 加速）

這種分層取捨讓 repo 既是教學材料（可以讀懂每一行 kernel），又是可實際訓練的系統（效能接近甚至超越 PyTorch）。
