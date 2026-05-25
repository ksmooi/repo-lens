---
repo: karpathy/nanoGPT
file: 2-code-walkthrough
studied_at: 2026-05-25
commit_sha: 3adf61e
---

# nanoGPT · 程式碼追蹤

## 追蹤的場景

**場景**: 從 random init 開始，在 8×A100 GPU 上用 DDP 訓練一次完整的梯度更新（含 gradient accumulation）。

**啟動命令**:
```bash
torchrun --standalone --nproc_per_node=8 train.py config/train_gpt2.py
```

## 流程圖

```mermaid
sequenceDiagram
    participant CLI as torchrun (8 proc)
    participant Train as train.py globals
    participant Conf as configurator.py
    participant Model as model.py
    participant Data as data (memmap)
    participant DDP as DDP Wrapper
    participant Opt as AdamW

    CLI->>Train: torchrun spawns 8 processes
    Train->>Conf: exec(configurator.py) ← overrides from cmd/config
    Note over Train,Conf: 全域變數即為 config
    Train->>Train: DDP init (init_process_group)
    Train->>Data: get_batch('train') — 首次取樣
    Data-->>Train: x, y tensors (b, t)
    Train->>Model: GPT(gptconf) from scratch
    Model-->>Train: model with inited params
    Train->>Model: torch.compile(model)
    Train->>DDP: DDP(model, device_ids=[rank])
    Note over Train,Opt: 訓練主迴圈

    loop per iteration (max 600k)
        Train->>Opt: get_lr(iter_num) → set lr
        Note over Train: 每 eval_interval 做 eval + checkpoint

        loop micro_step (gradient_accumulation_steps)
            Train->>DDP: skip grad sync on non-final steps
            Train->>Model: with autocast: model(X, Y)
            Model-->>Train: logits, loss
            Train->>Train: loss /= grad_accum_steps
            Train->>Data: get_batch('train')  ← async prefetch next
            Train->>Train: scaler.scale(loss).backward()
        end

        Train->>Train: clip_grad_norm_
        Train->>Opt: scaler.step(optimizer)
        Train->>Opt: scaler.update()
        Train->>Train: optimizer.zero_grad(set_to_none=True)
        Train->>Train: log (lossf, time, MFU)
    end
```

### 圖意說明

這是一個完整的 training step sequence。圖中展示 nanoGPT 最大的特色：**所有控制流都直接在 `train.py` 的全域 scope 中執行**，沒有任何間接的抽象層（沒有 Trainer class、沒有 Engine、沒有 callback）。DDP 的 gradient sync skip 是手動透過 `require_backward_grad_sync` 屬性控制的。模型的 forward/backward 在 `torch.cuda.amp.autocast` 下執行以啟用混合精度。

## 逐步追蹤

### Step 1: 進入點與 config 載入

[`train.py:76`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L76-L78)

```python
config_keys = [k for k,v in globals().items() if not k.startswith('_') and isinstance(v, (int, float, bool, str))]
exec(open('configurator.py').read())
config = {k: globals()[k] for k in config_keys}
```

這是 nanoGPT 最 unusual 的設計。`train.py` 的前 75 行定義了所有超參數作為**全域變數**（不是 dict、不是 dataclass、不是 argparse namespace）。然後：

1. 先 snapshot 所有 scalar 變數的 key 到 `config_keys`
2. `exec(open('configurator.py').read())` 執行 configurator 邏輯
3. 再 snapshot 一次，得到最終的 config dict

`configurator.py` 的執行流程 [`configurator.py:20-47`](https://github.com/karpathy/nanoGPT/blob/3adf61e/configurator.py#L20-L47)：

- 每個 CLI 參數檢查是否含 `=`
- 不含 `=` 的（如 `config/train_gpt2.py`）→ 視為 config file，`exec(open(file).read())` 直接執行該 Python 檔案
- 含 `=` 的（如 `--batch_size=32`）→ 用 `ast.literal_eval` 解析值、檢查型別符合後覆蓋 `globals()`

**注意型別檢查** [`configurator.py:42`](https://github.com/karpathy/nanoGPT/blob/3adf61e/configurator.py#L42)：`assert type(attempt) == type(globals()[key])`。這要求 CLI override 的型別必須跟預設值完全一致。例如 `--batch_size=32`（int）不能覆蓋一個本來是 float 的變數。這是一個簡陋但有效的防呆機制。

### Step 2: DDP 初始化

[`train.py:82-100`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L82-L100)

```python
ddp = int(os.environ.get('RANK', -1)) != -1
if ddp:
    init_process_group(backend=backend)
    ddp_rank = int(os.environ['RANK'])
    ddp_local_rank = int(os.environ['LOCAL_RANK'])
    ddp_world_size = int(os.environ['WORLD_SIZE'])
    device = f'cuda:{ddp_local_rank}'
    gradient_accumulation_steps //= ddp_world_size  # 關鍵：縮放 micro-batch
```

`torchrun` 會設定 `RANK`、`LOCAL_RANK`、`WORLD_SIZE` 三個環境變數。nanoGPT 透過檢查 `RANK` 是否存在來判斷是否為 DDP run。Gradient accumulation 根據 world size 自動縮放：如果原本總 grad_accum 是 40，在 8 GPU 上每個 GPU 只做 5 個 micro step。

**潛在問題**：`assert gradient_accumulation_steps % ddp_world_size == 0` 要求除法必須整除，否則 crash。在 `train_gpt2.py` 中 `gradient_accumulation_steps = 40`，40 % 8 == 0，所以沒問題。但自訂 config 時要注意。

### Step 3: 資料載入

**資料預處理**：執行 `data/openwebtext/prepare.py` 後產生 `train.bin` 和 `val.bin`。OpenWebText 原始約 40GB，tokenize 後約 19GB 的 uint16 陣列。

[`train.py:116-131`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L116-L131) — `get_batch()`：

```python
def get_batch(split):
    data = np.memmap(os.path.join(data_dir, 'train.bin'), dtype=np.uint16, mode='r')
    ix = torch.randint(len(data) - block_size, (batch_size,))
    x = torch.stack([torch.from_numpy((data[i:i+block_size]).astype(np.int64)) for i in ix])
    y = torch.stack([torch.from_numpy((data[i+1:i+1+block_size]).astype(np.int64)) for i in ix])
    if device_type == 'cuda':
        x, y = x.pin_memory().to(device, non_blocking=True), y.pin_memory().to(device, non_blocking=True)
```

重點：
- `np.memmap` 讓檔案的存取像 numpy array — 你隨機 index 的 segment 才真的從磁碟讀取
- 每次呼叫都重新建立 memmap 以避免 numpy 的 memory leak（每次新 process 重新 mmap 檔案，舊的會自動 GC）
- `non_blocking=True` 搭配 `pin_memory()` 讓 CPU→GPU 傳輸與計算 overlap
- X 是 `data[i:i+block_size]`，Y 是 `data[i+1:i+1+block_size]`，所以 targets 永遠是 input 右 shift 一位的下一個 token

### Step 4: 模型構造

[`train.py:146-192`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L146-L192)

```python
model_args = dict(n_layer=n_layer, n_head=n_head, n_embd=n_embd, block_size=block_size,
                  bias=bias, vocab_size=None, dropout=dropout)

if init_from == 'scratch':
    gptconf = GPTConfig(**model_args)
    model = GPT(gptconf)
```

`GPT.__init__()` [`model.py:120-148`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L120-L148) 的初始化順序：

1. 建立 `ModuleDict` 包含 `wte`（token embedding）、`wpe`（position embedding）、`drop`、`h`（N 層 Block）、`ln_f`
2. 建立 `lm_head`（Linear from n_embd to vocab_size）
3. **Weight tying**：`self.transformer.wte.weight = self.lm_head.weight` — token embedding 和 output projection 共享同一份 weight
4. 初始化權重：所有 Linear/Embedding 用 `N(0, 0.02)`，residual projection（`c_proj.weight`）用 `N(0, 0.02/√(2*n_layer))` — 這是 GPT-2 paper 的特殊 scaled init

### Step 5: 一個 training step 的詳細追蹤

#### 5.1 設定 learning rate

[`train.py:258-260`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L258-L260)

```python
lr = get_lr(iter_num) if decay_lr else learning_rate
for param_group in optimizer.param_groups:
    param_group['lr'] = lr
```

`get_lr()` [`train.py:231-242`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L231-L242) 是 classic cosine decay with linear warmup：

```
iter < warmup_iters:    lr = base_lr * (iter+1)/(warmup_iters+1)
iter > lr_decay_iters:  lr = min_lr
otherwise:              lr = min_lr + 0.5*(1+cos(π*decay_ratio))*(base_lr - min_lr)
```

**注意**：`lr_decay_iters` 預設 `= max_iters` [`train.py:67`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L67)，也就是 Cosine 會 decay 到最後一步。這符合 Chinchilla paper 的 recommendation。

#### 5.2 評估 / checkpoint（每 eval_interval 執行）

[`train.py:263-288`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L263-L288)

`estimate_loss()` [`train.py:216-228`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L216-L228) 在 `torch.no_grad()` 和 `model.eval()` 下執行 `eval_iters` 次（預設 200）forward pass，回傳 train/val loss 的平均值。

Checkpoint 存 [`train.py:277-286`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L277-L286)：
```python
checkpoint = {
    'model': raw_model.state_dict(),
    'optimizer': optimizer.state_dict(),
    'model_args': model_args,
    'iter_num': iter_num,
    'best_val_loss': best_val_loss,
    'config': config,
}
```

#### 5.3 Gradient accumulation 微批次迴圈

[`train.py:292-305`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L292-L305)

這是整個訓練中最關鍵、最容易出錯的地方：

```python
for micro_step in range(gradient_accumulation_steps):
    if ddp:
        model.require_backward_grad_sync = (micro_step == gradient_accumulation_steps - 1)
    with ctx:  # autocast
        logits, loss = model(X, Y)
        loss = loss / gradient_accumulation_steps
    X, Y = get_batch('train')  # async prefetch next batch
    scaler.scale(loss).backward()
```

三個要點：

1. **DDP gradient sync skip** [`train.py:298`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L298)：在前 `N-1` 個 micro step 關閉 DDP 的 all-reduce gradient sync（`model.require_backward_grad_sync = False`），只在最後一個 micro step 才同步。這避免每個 micro step 都做一次昂貴的 all-reduce，而只做一次。

2. **Loss scaling** [`train.py:301`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L301)：`loss = loss / gradient_accumulation_steps`。因為每個 micro step 的 loss 是獨立算的，累積 N 個 gradient 後，如果沒做除法，等效 batch size 會放大 N 倍，learning rate 需要配合調整。除以 N 後，gradient 的平均值跟單次 full-batch 的 gradient 統計上是同一個量級。

3. **Async prefetch** [`train.py:303`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L303)：在 loss.backward() 還在 GPU 上跑的時候，CPU 已經開始準備下一批資料。因為 `get_batch` 的 main latency 是 `np.memmap` 的隨機讀取 + pinned memory transfer，這個 overlap 對總 throughput 有可量化的幫助。

#### 5.4 Gradient clipping 與 optimizer step

[`train.py:307-314`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L307-L314)

```python
if grad_clip != 0.0:
    scaler.unscale_(optimizer)  # 必須在 clip 前 unscale
    torch.nn.utils.clip_grad_norm_(model.parameters(), grad_clip)
scaler.step(optimizer)
scaler.update()
optimizer.zero_grad(set_to_none=True)
```

**注意**：`set_to_none=True` 比傳統的 `zero_grad()` 更快（PyTorch 官方建議），因為它直接將 gradient tensor 設為 None 而非填零，減少 memory write。

### Step 6: MFU 估算

[`train.py:324-326`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L324-L326)

```python
if local_iter_num >= 5:
    mfu = raw_model.estimate_mfu(batch_size * gradient_accumulation_steps, dt)
    running_mfu = mfu if running_mfu == -1.0 else 0.9*running_mfu + 0.1*mfu
```

`estimate_mfu()` [`model.py:289-303`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L289-L303) 實作 PaLM paper 的 formula：

```text
flops_per_token = 6*N + 12*L*H*Q*T
flops_per_fwdbwd = flops_per_token * T
flops_per_iter = flops_per_fwdbwd * (fwdbwd_per_iter)
mfu = (flops_per_iter / dt) / A100_peak_flops
```

其中 `N` = 參數量、`L` = layer 數、`H` = head 數、`Q` = head dimension、`T` = sequence length。

這個公式的上半部（`6*N`）對應一次 forward+backward 的 matrix multiply 總操作數（每個參數在 forward 被讀一次、backward 被讀一次、gradient 被寫一次，所以 6 次）。下半部（`12*L*H*Q*T`）對應 attention 的 FLOPS。

**running_mfu 使用 EMA**（0.9 × old + 0.1 × new），因為單次 iter 的 dt 有雜訊（排程、系統抖動），平滑後更穩定。開頭 5 個 iter 不計算，讓 GPU warmup 穩定。

## 想學更多時，在哪裡下中斷點

- **想看一個 batch 的實際內容**: `train.py:123`（get_batch 中的 ix 取完後），print `x.shape` 和 `x[0, :10]`
- **想看 forward 內部各層輸出**: `model.py:180`（for block loop 內），hook 某個 block 的輸出
- **想看 gradient 大小**: `train.py:309`（optimizer.step() 前），印出各層 gradient norm
- **想看 checkpoint 內容**: `train.py:285`（torch.save 前），檢查 model_args 一致性
- **想看 attention pattern**: `model.py:64`（scaled_dot_product_attention 前），dump q, k 的統計量

## 沒追蹤到但值得留意

- **`bench.py`**：提供不含 training overhead 的純 forward/backward benchmark，對 debug 效能問題有用
- **`from_pretrained()` 的 Conv1D→Linear transpose**：OpenAI 的 GPT-2 checkpoint 用 Conv1D（weight shape = [out, in]），nanoGPT 用 Linear（[in, out]），載入時需要 transpose [`model.py:250-254`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L250-L254)
- **`crop_block_size()`**：model surgery 縮短 context length 的實作 [`model.py:195-204`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L195-L204)，直接 truncate wpe.weight 和 attn.bias 矩陣
