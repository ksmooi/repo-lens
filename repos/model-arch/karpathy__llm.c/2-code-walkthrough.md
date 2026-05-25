---
repo: karpathy/llm.c
file: 2-code-walkthrough
studied_at: 2025-06-26
commit_sha: f1e2ace
---

# llm.c · 程式碼追蹤：一個完整的訓練步（GPU 版）

## 追蹤的場景

**場景**：從 random init 開始，一個 GPU training step forward → backward → AdamW update 的完整路徑。

**啟動命令**：
```bash
make train_gpt2cu
./train_gpt2cu -n 1  # 只跑 1 步
```

## 流程圖

```mermaid
sequenceDiagram
    participant Main as main()
    participant DL as DataLoader
    participant FWD as gpt2_forward
    participant CLS as Fused Classifier
    participant BWD as gpt2_backward_and_reduce
    participant CLIP as Gradient Clipping
    participant ADAM as gpt2_update (AdamW)
    participant LOG as Logger / MFU

    Main->>Main: CLI parsing (argv)
    Main->>Main: common_start() → cuBLAS/cuDNN init
    Main->>Main: gpt2_allocate_weights (device)
    Main->>Main: gpt2_allocate_state (device)
    Main->>DL: dataloader_init
    Main->>DL: dataloader_next_batch → inputs, targets
    Main->>Main: gpt2_set_hyperparameters
    Main->>Main: gpt2_allocate_mstate (AdamW m/v)

    loop training loop [step=0..train_num_batches]
        Main->>Main: cudaEvent timing start
        Main->>Main: zero_grad (cudaMemset gradients)
        Main->>FWD: gpt2_forward(inputs, B, T)
        FWD->>FWD: encoder_forward → wte + wpe
        FWD->>FWD: layernorm_forward
        loop L layers
            FWD->>FWD: matmul_cublaslt (qkv)
            FWD->>FWD: attention_forward (QKt/V)
            FWD->>FWD: matmul_cublaslt (attproj)
            FWD->>FWD: fused_residual_forward5
            FWD->>FWD: matmul_cublaslt (fc)
            Note over FWD: GELU fused into cuBLASLt epilogue (H100+)
            FWD->>FWD: matmul_cublaslt (fcproj)
            FWD->>FWD: fused_residual_forward5
        end
        FWD->>FWD: layernorm_forward (lnf)
        FWD->>FWD: matmul_cublaslt (lm_head = wte.T)
        FWD->>CLS: fused_classifier → loss
        CLS-->>FWD: loss value

        Main->>BWD: gpt2_backward_and_reduce(inputs, targets, grad_accum_steps)
        BWD->>BWD: fused_classifier backward (in-place dlogits)
        BWD->>BWD: matmul_backward (dlogits→dwte+dlnf)
        reverse loop L layers [L-1..0]
            BWD->>BWD: matmul_backward (fcproj)
            Note over BWD: if recompute≥1, gelu_forward rerun
            BWD->>BWD: matmul_backward (fc)
            BWD->>BWD: layernorm_backward (ln2)
            BWD->>BWD: matmul_backward (attproj)
            BWD->>BWD: attention_backward
            BWD->>BWD: matmul_backward (qkv)
            BWD->>BWD: layernorm_backward (ln1)
        end
        BWD->>BWD: encoder_backward (dwte+dwpe)

        Main->>CLIP: gpt2_calculate_grad_norm
        Main->>CLIP: gradient clipping (scale global_norm to 1.0)

        Main->>ADAM: gpt2_update (AdamW)
        ADAM->>ADAM: m = β₁·m + (1-β₁)·g
        ADAM->>ADAM: v = β₂·v + (1-β₂)·g²
        ADAM->>ADAM: param -= lr · (m_hat/(√v_hat+ε) + wd·param)

        Main->>LOG: log loss + MFU + timing
    end
```

**圖意說明**：上圖展示單步訓練的完整生命週期。初始化階段（cuBLAS/cuDNN、權重分配、dataloader）完成後，進入訓練迴圈。每個 step：歸零梯度 → forward（encoder → L 層 transformer → classifier）→ loss 計算 → backward（反向遍歷所有層）→ gradient norm + clipping → AdamW 參數更新 → 日誌輸出。

## 逐步追蹤：Forward Pass

### Step 1：Encoder — Token + Position Embedding

[`train_gpt2.cu:680`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L680)

```c
encoder_forward(acts.encoded, inputs, params.wte, params.wpe, B, T, C);
```

每個位置 `(b,t)` 取出 token embedding `wte[input[b,t]]` 和 position embedding `wpe[t]` 相加。這是 memory-bound 的查表操作（`load128cs` streaming load，因為只讀一次）。

### Step 2：第一層 LayerNorm

[`train_gpt2.cu:683`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L683)

```c
layernorm_forward(acts.ln1[0], acts.mean[0], acts.rstd[0], acts.encoded, params.ln1w[0], params.ln1b[0], B, T, C);
```

計算 mean → variance → rstd → normalize → scale+shift。`mean` 和 `rstd` 被快取供 backward 使用。

### Step 3-12：Transformer 層主迴圈

[`train_gpt2.cu:685-751`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L685-L751)

對每一層 `l = 0..L-1`：

**a) QKV Projection** ([`train_gpt2.cu:720`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L720))：`matmul_cublaslt(acts.qkv[l], acts.ln1[l], params.qkvw[l])` — cuBLASLt GEMM，一次產生 Q/K/V 三個投影。

**b) Multi-Head Self-Attention** ([`train_gpt2.cu:721-730`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L721-L730))：Q·K^T → softmax（online softmax、causal mask）→ attention·V。支援 cuDNN 與手寫兩種路徑。手寫版在 `llmc/attention.cuh`。

**c) Attention Output Projection** ([`train_gpt2.cu:733`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L733))：`matmul_cublaslt(acts.attproj[l], acts.atty[l], params.attprojw[l])`

**d) Fused Residual + LayerNorm** ([`train_gpt2.cu:734`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L734))：「residual add with skip connection → layernorm」融合為單一 kernel `fused_residual_forward5`，省去一次 full tensor 寫入。

**e) MLP: fc → GELU → fcproj** ([`train_gpt2.cu:735-736`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L735-L736))：`matmul_cublaslt` 做 fc（4× 擴張），cuBLASLt epilogue 融合 GELU（H100+），然後 `matmul_cublaslt` 做 fcproj（壓回 C）。

**f) Fused Residual + LayerNorm（第二個）** ([`train_gpt2.cu:744-749`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L744-L749))：MLP 後的第二個 residual + layernorm。

### Step 13：Final LayerNorm + LM Head

[`train_gpt2.cu:753`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L753)

```c
matmul_cublaslt(acts.logits, acts.lnf, params.wte, B, T, C, padded_vocab_size);
```

注意這裡用 `params.wte`（token embedding 權重）當作 LM head 的 weight matrix — 這就是 weight tying。

### Step 14：Fused Classifier

[`train_gpt2.cu:754`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L754)

```c
fused_classifier(acts.logits, acts.losses, targets, B, T, padded_vocab_size, vocab_size);
```

同時做三件事：softmax（online 演算法，不實體化完整機率分布）、cross-entropy loss 計算（只對 target token 位置算 loss）、以及 dlogits 初始化（in-place 覆寫 logits 張量）。這個 kernel 用**反向迭代**（從最後一個 token 開始處理），讓前一層 matmul 的資料留在 cache 中。

## 逐步追蹤：Backward Pass

### 初始化

[`train_gpt2.cu:829`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L829)

`dloss = 1.0f / (B * T * grad_accum_steps)` — loss 梯度標準化。

### 反向主迴圈 ([`train_gpt2.cu:849-948`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L849-L948))

從 `l = L-1` 到 `0`，完全鏡像 forward 的順序：

1. **MLP fcproj backward**：`matmul_backward`
2. **GELU backward**：若 `recompute≥1`，先重跑 `gelu_forward` 重新計算 `fch_gelu`（原來沒存），再執行 backward
3. **MLP fc backward**：`matmul_backward`
4. **LayerNorm backward (ln2)**：`layernorm_backward`
5. **Attention projection backward**：`matmul_backward`
6. **Attention backward**：手寫或 cuDNN
7. **QKV backward**：`matmul_backward`
8. **LayerNorm backward (ln1)**：`layernorm_backward`

### Encoder backward

[`train_gpt2.cu:949-950`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L949-L950)

`encoder_backward` 是最複雜的 backward 之一：
- `dwpe`：每個 `(t,c)` 唯一 thread 更新（確定性保證）
- `dwte`：因為同一 token 可能出現多次，用 CPU 桶排序 + GPU block 處理每個桶（避免 `atomicAdd`）

### 多 GPU 梯度 reduce

[`train_gpt2.cu:929-947`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L929-L947)

只有 `grad_accum_steps` 中的最後一個 micro-step 才發起 NCCL async all-reduce，減少通訊開銷。

## 逐步追蹤：Optimizer Step

### Gradient Clipping

[`train_gpt2.cu:1849`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1849)

`gpt2_calculate_grad_norm` 計算所有梯度的 global L2 norm，若 >1.0 則縮放。

### AdamW Update

[`train_gpt2.cu:1852`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1852)

[`adamw.cuh`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/adamw.cuh) 中的 `adamw_kernel3` 做：
```c
m = beta1*m + (1-beta1)*g
v = beta2*v + (1-beta2)*g²
m_hat = m / (1-beta1^t)
v_hat = v / (1-beta2^t)
param -= lr * (m_hat/(√v_hat+ε) + wd*param)
```

支援 master weights（FP32 權重副本）和 ZeRO sharding（多 GPU）。

## 想學更多時，在哪裡下中斷點

| 想觀察什麼 | 檔案與行號 |
|---|---|
| 一個 batch 的 input/target 長什麼樣 | [`train_gpt2.cu:1829-1836`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1829-L1836) — dataloader_next_batch 之後 |
| 某層的 QKV 投影輸出 | [`train_gpt2.cu:720`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L720) — matmul_cublaslt 呼叫之後 |
| Attention softmax 前的 scores | [`attention.cuh:217`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/attention.cuh#L217) — preatt 內容 |
| Loss 值（cross-entropy） | [`fused_classifier.cuh:84-86`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/fused_classifier.cuh#L84-L86) — 目標位置的 loss 計算 |
| 梯度 global norm | [`train_gpt2.cu:1847`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1847) — gpt2_calculate_grad_norm 之後 |
| AdamW 更新參數 | [`adamw.cuh:38`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/adamw.cuh#L38-L46) — master weight → floatX 轉換點 |

## 沒追蹤到但值得留意

- **Checkpoint 載入/儲存路徑**（`train_gpt2.cu:430-517`）：支援 resume 訓練，header 包含 256 個 int32 的 metadata。
- **HellaSwag 評估路徑**（`train_gpt2.cu:757-786`）：使用 multi-choice 評分機制（logits 聚合 → 計算 perplexity）。
- **MFU 估算**（[`mfu.h`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/mfu.h)）：計算 Model FLOPS Utilization = (實際 FLOPS) / (GPU 理論 FLOPS)，用來評估 kernel 效率。
- **CPU 版訓練路徑** (`train_gpt2.c`，1182 行)：與 GPU 版完全不同的實作方式，所有 forward/backward 在單一檔案內用 plain C loops + OpenMP 完成，無 CUDA 依賴。
