---
repo: karpathy/nanoGPT
file: 1-architecture
studied_at: 2026-05-25
commit_sha: 3adf61e
---

# nanoGPT · 架構

## 系統高層圖

nanoGPT 是訓練 + 推論一體的極簡設計。三條主要路徑（訓練、評估、採樣）共享同一個 model forward，只在 loss 計算和 model.eval() 處分岔。

```mermaid
flowchart TB
    subgraph Data["資料層"]
        PREP[prepare.py<br/>raw text → tokenized .bin]
        DB[(train.bin / val.bin<br/>uint16 memmap)]
    end

    subgraph Config["設定"]
        CFG[config/*.py<br/>Python variable overrides]
        CONFG[configurator.py<br/>exec() 動態 override]
    end

    subgraph Model["模型定義"]
        GPT[GPT class<br/>model.py:118]
        GPTCONF[GPTConfig dataclass<br/>model.py:108]
        ATTN[CausalSelfAttention<br/>model.py:29]
        MLP[MLP<br/>model.py:78]
        BLOCK[Block<br/>model.py:94]
    end

    subgraph Train["訓練"]
        TRAIN[train.py]
        DDP[DistributedDataParallel]
        OPT[AdamW<br/>configure_optimizers]
        LOSS[CrossEntropyLoss<br/>model.py:187]
        LR[Cosine LR Scheduler<br/>get_lr()]
    end

    subgraph Eval["評估 / 推論"]
        EVAL[estimate_loss<br/>train.py:216]
        SAMPLE[sample.py]
        GEN[generate<br/>model.py:306]
    end

    PREP --> DB
    DB --> |get_batch| TRAIN
    DB --> |get_batch| EVAL
    CFG --> |exec(open(...).read())| TRAIN
    CONFG --> |command line --key=val| TRAIN
    GPTCONF --> GPT
    GPT --> BLOCK
    BLOCK --> ATTN
    BLOCK --> MLP
    TRAIN --> DDP
    TRAIN --> OPT
    TRAIN --> |model(X,Y)| GPT
    GPT --> |logits, loss| LOSS
    LOSS --> |backward| OPT
    LR --> |set lr each iter| OPT
    EVAL --> |model.eval()| GPT
    SAMPLE --> GPT
    GPT --> GEN
```

### 圖意說明

這張圖展示 nanoGPT 的四個層次：**資料層**（左）把原始文字預先 tokenize 成 .bin 檔，訓練時用 `np.memmap` 零拷貝讀取；**設定層**（右上）以 `exec()` 動態解析 config 檔與 CLI 參數，直接注入 `globals()`；**模型定義**（中）是單一檔案的 GPT-2 完整實作；**訓練/評估/推論**（右）共享同一個 `GPT.forward()`，差別只在於 `targets` 參數的有無與 `model.eval()` / `torch.no_grad()` 狀態。

整體設計哲學：**不引入任何 framework-level 的抽象**。沒有 Trainer class、沒有 Engine、沒有 DataModule。所有流程直接寫在 `train.py` 的全域 scope。

## 資料管線

### 資料來源

- **資料集**: OpenWebText（open reproduction of WebText）
- **下載腳本**: [`data/openwebtext/prepare.py`](https://github.com/karpathy/nanoGPT/blob/3adf61e/data/openwebtext/prepare.py)
- **資料格式**: uint16 binary file（`train.bin` / `val.bin`），預先 tokenize 完的一維 token 序列

### 預處理

完整離線。`prepare.py` 把原始文字用 GPT-2 BPE tokenizer（tiktoken）轉成 token ID 陣列，存成單一連續的 uint16 binary。訓練時不需任何即時 tokenization。

### DataLoader 設計 — 最關鍵的簡化

nanoGPT 的 DataLoader 是「poor man's data loader」[`train.py:116`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L116-L131)：

```python
def get_batch(split):
    data = np.memmap(os.path.join(data_dir, 'train.bin'), dtype=np.uint16, mode='r')
    ix = torch.randint(len(data) - block_size, (batch_size,))
    x = torch.stack([torch.from_numpy((data[i:i+block_size]).astype(np.int64)) for i in ix])
    y = torch.stack([torch.from_numpy((data[i+1:i+1+block_size]).astype(np.int64)) for i in ix])
```

**設計取捨**：
- ✅ **零拷貝讀取**：`np.memmap` 讓 OS 的 page cache 處理 I/O，首次訪問才有實際磁碟讀取
- ✅ **每次重新 mmap**：每呼叫 `get_batch` 就重新建立 memmap object，避免 numpy 已知的 memory leak（[SO reference](https://stackoverflow.com/questions/45132940/numpy-memmap-memory-usage-want-to-iterate-once/61472122#61472122)）
- ❌ **無 DataLoader / worker 多進程**：完全沒有 `torch.utils.data.DataLoader`，所有的 I/O 都發生在主進程。因為 `np.memmap` + random index 已經是兩次記憶體拷貝（memmap → numpy array → torch tensor），對 8x A100 的吞吐量來說夠快
- **Trade-off**：捨棄了 worker prefetch、bucketing、dynamic padding 等進階功能，換來 15 行的極簡實作。對 GPT-2 124M 這種 medium-sized 模型綽綽有餘，但對需要複雜資料預處理的場景就不夠

## 模型架構

### 整體結構

```text
GPT.forward(idx, targets=None):
    1. wte(idx) → token embeddings  (b, t, n_embd)
    2. wpe(pos) → position embeddings (t, n_embd)
    3. tok_emb + pos_emb → drop
    4. for block in h: block(x)
    5. ln_f(x)  → final layer norm
    6a. if targets: lm_head(x) → logits (b, t, vocab) → cross_entropy → loss
    6b. if not targets: lm_head(x[:, [-1], :]) → logits (b, 1, vocab) [inference opt]
    return logits, loss
```

### 關鍵元件

| 元件 | 位置 | 備註 |
|------|------|------|
| CausalSelfAttention (MHA) | [`model.py:29`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L29-L76) | Flash Attention (PyTorch 2.0) / 手動 masked attention 雙模式 |
| LayerNorm (pre-norm) | [`model.py:18`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L18-L27) | 自訂版，支援 bias=False |
| Learned Positional Embedding | [`model.py:128`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L128) | wpe (word position embedding) |
| GELU Activation | [`model.py:83`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L83) | PyTorch 內建 nn.GELU |
| MLP 擴張比 | [`model.py:82`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L82) | 4x (`c_fc` 從 n_embd → 4*n_embd) |

### 關鍵超參數（GPT-2 124M）

```python
# 從 GPTConfig 預設值 [model.py:108]
block_size = 1024    # 最大 context length
vocab_size = 50304   # GPT-2 原 50257 rounded up to multiple of 64
n_layer = 12         # transformer block 數量
n_head = 12          # attention head 數量
n_embd = 768         # embedding dimension
dropout = 0.0        # pretraining 不開 dropout
bias = True          # GPT-2 原版有 bias
```

### 跟原論文的差異

nanoGPT 基本上是 GPT-2 paper 的 faithful implementation，但有幾個工程上的偏離：

- **vocab_size padding** [`model.py:111`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L111)：GPT-2 的 vocab_size 是 50257，但 nanoGPT 預設用 50304（最接近的 64 的倍數）。目的是為了 GPU tensor core 的 alignment 效率。損失極小的額外參數換取 ~5-10% 的 matrix multiply 加速。[UNVERIFIED]
- **Bias 可選** [`model.py:116`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L116)：原始 GPT-2 在 Linear 和 LayerNorm 都有 bias，但 nanoGPT 把它參數化。實務上關掉 bias 可以節省少量參數並略微提升速度，對 final loss 幾乎沒影響。
- **Flash Attention 自動偵測** [`model.py:45`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L45)：在 PyTorch 2.0+ 自動使用 `scaled_dot_product_attention`（內部呼叫 Flash Attention / memory-efficient attention），否則 fallback 到手動 masked softmax attention。這在原始的 GPT-2（2019）還不存在。

## 訓練流程

### 訓練腳本入口

[`train.py`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py) — 約 336 行，全域 scope 的訓練腳本。啟動方式：

```bash
# 單卡 debug
python train.py --batch_size=32 --compile=False

# 8 卡 reproduce GPT-2
torchrun --standalone --nproc_per_node=8 train.py config/train_gpt2.py
```

### 訓練迴圈核心

```
for iter in range(max_iters):
    1. 設定 learning rate (get_lr)
    2. 每 eval_interval: evaluate loss → checkpoint
    3. for micro_step in gradient_accumulation_steps:
         a. forward(model(X, Y))
         b. loss /= grad_accum_steps
         c. async prefetch next batch
         d. backward()
    4. gradient clipping
    5. optimizer.step() + scaler.update()
    6. optimizer.zero_grad(set_to_none=True)
    7. logging (loss, time, MFU)
```

### Optimizer & Scheduler

- **Optimizer**: AdamW [`model.py:284`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L284)，可選 fused AdamW（CUDA 平台自動啟用）
- **Learning rate schedule**: Cosine decay with linear warmup [`train.py:231`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L231-L242)
  - 預設 2000 steps warmup → cosine decay to min_lr (= lr / 10)
  - min_lr 跟 max_lr 的 10x ratio 來自 Chinchilla 論文的建議
- **Weight decay 策略**: 所有 2D 參數（weight matrices + embeddings）decay，1D 參數（bias + layernorm weights）不 decay [`model.py:270`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L270-L274)

### Loss

Standard `F.cross_entropy(logits.view(-1, vocab_size), targets.view(-1), ignore_index=-1)` [`model.py:187`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L187)。`ignore_index=-1` 讓 padding token 不貢獻 gradient。

### 分散式訓練

- **策略**: PyTorch DDP（DistributedDataParallel）
- 自動偵測 `RANK` 環境變數來判斷是否為 DDP run [`train.py:82`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L82)
- gradient accumulation steps 會根據 `ddp_world_size` 自動縮放 [`train.py:94`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L94-L95)
- 在 gradient accumulation 的 micro step 中，只在最後一個 step sync gradient [`train.py:298`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L298)
- **Mixed precision**: auto-detect bfloat16 / float16，float16 自動啟用 GradScaler [`train.py:73`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L73)

### Checkpoint 機制

- **存檔頻率**: 每 `eval_interval` steps（預設 1000 for GPT-2 config）
- **存什麼**: model state_dict / optimizer state_dict / model_args / iter_num / best_val_loss / config [`train.py:277`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L277-L286)
- **Resume 邏輯**: 從 checkpoint 載入時，會檢查 `_orig_mod.` prefix（`torch.compile` 引入的 wrapper 殘留）並 strip 掉 [`train.py:174`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L174-L177)

## 推論

- **Inference 腳本**: `sample.py`（89 行）
- **跟 training forward 的差異**:
  - `model.eval()` + `torch.no_grad()`
  - forward 時 `targets=None`，只對最後一個 position 做 `lm_head` [`model.py:190`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L190)
  - Sampling 支援 temperature scaling + top-k filtering
  - **無 KV cache**：每次 `generate()` 都重新跑完整的 forward

### KV cache 的顯著缺席

nanoGPT 的 `generate()` [`model.py:306`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L306-L330) 沒有實作 KV cache。這在 inference 時造成 O(context_length²) 的浪費。例如生成 500 tokens 時，第一次 forward 跑 1 token，第二次跑 2 tokens，⋯，第 500 次跑 500 tokens。總計算量大約是 O(T²/2)。這對一個「展示用」的 script 是可接受的簡化，但對 production inference 來說是重大瓶頸。（這個 trade-off 在 README 的效率 notes 中沒有被討論，是一項值得注意的缺失。）

## Config 系統 — nanoGPT 最有爭議的設計

[`configurator.py`](https://github.com/karpathy/nanoGPT/blob/3adf61e/configurator.py)（47 行）的設計：

1. 先掃描 `train.py` 的 globals() 中所有 scalar 變數當作預設值
2. 執行 `exec(open('configurator.py').read())`，這會依序處理 CLI 參數：
   - 無 `=` 的參數 → 視為 config file → `exec(open(file).read())`
   - 有 `=` 的 `--key=val` → `literal_eval(val)` 後直接覆蓋 globals()
3. 最終把所有 config keys snapshot 成一個 dict 供 logging

**為什麼用 `exec()` 而不是 argparse / Hydra / YAML？** Karpathy 在 docstring 直接說：「I know people are not going to love this, I just really dislike configuration complexity。」這是刻意的 trade-off：

| 面向 | exec() 方案 | argparse / Hydra |
|------|------------|-----------------|
| 維護性 | 幾乎為零。無法靜態檢查，命名衝突風險 | 良好，可型別驗證 |
| 彈性 | 極高。config file 是任意 Python code | 受限於 schema 定義 |
| 可讀性 | 不透明的 magic。新手會困惑 `exec()` 做了什麼 | 明確的 API |
| 對小專案 | 47 行解決一切問題 | 引入依賴 + 樣板 |
| 對大專案 | 災難。誰改過什麼 config 完全不可追蹤 | 有 diff、有繼承、有 override 追蹤 |

**我的判斷**：對 nanoGPT 這種單一腳本的 research code 來說是合理的選擇。但如果專案成長到 5+ 個 config file、需要多層 override 時，應該轉換到 Hydra 或 dataclass-based config。

## 可重現性

- **Random seed**: `torch.manual_seed(1337 + seed_offset)`（DDP 各 process 有不同 seed） [`train.py:106`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L106)
- **Determinism flags**: 無 `torch.backends.cudnn.deterministic`。使用 TF32 但未做完全的 determinstic 設定
- **環境鎖定**: `requirements.txt` 僅是說明文件中的 `pip install ...` 列表，沒有 pinned versions
- **資料版本控制**: 無。train.bin / val.bin 是 locally generated
- **觀察**: nanoGPT 的可重現性不如 research standard。沒有 pinned dependencies、沒有環境鎖定檔。對一個教育/展示為目的的 repo 來說可以接受，但若要作為實驗管理的 baseline 需要補強

## 測試

- **無單元測試**。整個 repo 沒有任何 test file
