---
repo: karpathy/nanoGPT
file: 3-key-patterns
studied_at: 2026-05-25
commit_sha: 3adf61e
---

# nanoGPT · 值得偷學的設計

## Pattern 1: 單一全域變數 Config + exec() 動態覆蓋

**是什麼**: 把 config 參數宣告為 Python 全域變數，用 `exec(open('configurator.py').read())` 在 runtime 動態執行 config file 和 CLI override，直接修改 `globals()`。

**為什麼有效**:
- 零 boilerplate：不用定義 argparse parser、不用寫 YAML schema、不用宣告 dataclass
- 型別從預設值繼承：`type(attempt) == type(globals()[key])` 確保 override 型別正確
- Config file 是任意 Python 程式碼 → 可以放入條件式、迴圈、從環境變數取值

**程式碼位置**: [`configurator.py`](https://github.com/karpathy/nanoGPT/blob/3adf61e/configurator.py)（47 行）

**何時可以借用**:
- 個人專案或 < 3 人團隊的 research code
- 單一 script 且 config 參數數量 < 50
- 不需要 config schema 版本管理或繼承機制

**替代方案**:

| 方案 | 優點 | 缺點 |
|------|------|------|
| **exec() globals**（nanoGPT） | 47 行搞定，零依賴 | 不可靜態檢查，命名衝突風險 |
| **argparse** | 標準庫，靜態定義 | 巢狀 config 難處理 |
| **Hydra / OmegaConf** | YAML config + 任意層級 override + 命令列自動補全 | 學習曲線，import 依賴 |
| **Pydantic Settings** | 型別安全，IDE autocomplete | 樣板較多 |

**不適用情境**: 多人協作的大型專案、需要 config 版本化管理、config 參數 > 100 的專案。在這些情境下，`exec()` 的不可追蹤性會變成真正的技術債。

---

## Pattern 2: 貧民版 DataLoader — `np.memmap` + `pin_memory`

**是什麼**: 捨棄 `torch.utils.data.DataLoader`，直接用 `np.memmap` 讀取預先 tokenize 的 uint16 binary，隨機 index 後 non-blocking 搬到 GPU。

**為什麼有效**:
- `np.memmap` 讓 OS 處理實際 I/O，不需要 producer-consumer queue
- 每次重新 mmap 解決 numpy 的 memory leak 問題
- `pin_memory()` + `non_blocking=True` 讓 CPU→GPU 傳輸與下一個 forward 計算 overlap

**程式碼位置**: [`train.py:116-131`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L116-L131)

**何時可以借用**:
- 資料可以完全離線 tokenize（一次寫入 .bin）
- 不需要即時 augmentation / preprocessing
- 不需要 multi-worker prefetch（模型計算夠快吃掉 I/O latency）

**替代方案**:
- **`torch.utils.data.DataLoader`**：更通用，支援 worker 多進程、collate function、sampler。但對於 nanoGPT 這種只需要 random slice 的場景來說，多了不必要的抽象層
- **WebDataset / Mosaic StreamingDataset**：對真正的大規模訓練（>1TB）有必要，對 nanoGPT 的 ~19GB OpenWebText 來說 overkill

**不適用情境**: 需要即時 tokenization、資料 augmentation、動態 padding / bucketing 的場景。例如多模態訓練（文字+圖片）就不能用這個方式。

---

## Pattern 3: 不完整的 DDP Gradient Sync Pattern

**是什麼**: 在 gradient accumulation 中，只在最後一個 micro step 做 DDP 的 gradient all-reduce，前 N-1 個 micro step 透過 `model.require_backward_grad_sync = False` 跳過同步。

**為什麼有效**:
- Gradient accumulation 的意義是累積多個 micro-batch 的 gradient 後一次更新
- 只有在最後一個 micro step 才需要 `all-reduce` 來讓所有 GPU 的 gradient 一致
- 跳過前 N-1 次的 all-reduce 節省約 `(N-1)/N` 的通訊量

**程式碼位置**: [`train.py:292-305`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L292-L305)

```python
model.require_backward_grad_sync = (micro_step == gradient_accumulation_steps - 1)
```

**替代方案**:
- **`model.no_sync()` context manager**：HuggingFace Transformers 的 Trainer 用這個方式，需要寫成 `with model.no_sync(): ...`，nanoGPT 的作者認為這「bloats the code and forces us to repeat code」[`train.py:296`](https://github.com/karpathy/nanoGPT/blob/3adf61e/train.py#L296-L297)，所以直接用屬性設定
- **FSDP**：更先進的 sharding 策略，但對 GPT-2 124M（12 layers, 768 hidden）來說 DDP 已經足夠

**注意事項**: `require_backward_grad_sync` 是 DDP wrapper 的內部屬性，不是公開 API。在未來 PyTorch 版本中可能改變。對 production code 建議用 `model.no_sync()`。

---

## Pattern 4: Weight Tying + 殘差投影的 Scaled Init

**是什麼**: 兩個初始化的細節設計：
1. **Weight tying**: token embedding (`wte`) 和 output projection (`lm_head`) 共享同一份 weight matrix
2. **Scaled residual init**: `c_proj.weight`（殘差連接的 output projection）用 `N(0, 0.02/√(2*n_layer))` 而非預設的 `N(0, 0.02)`

**為什麼有效**:

**Weight tying** [`model.py:138`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L138)：
- 節省大量參數（vocab_size × n_embd，對 GPT-2 124M 來說約 38M 參數，佔總參數的 ~30%）
- 有論文指出 weight tying 可以改善語言模型的 perplexity（尤其對小模型）

**Scaled residual init** [`model.py:144`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L144)：
- 標準 GPT-2 init（`N(0, 0.02)`）用在每個 sublayer 的 output projection 上時，隨著 layer 增加，殘差流中的 variance 會累積增長
- 除以 `√(2*n_layer)` 讓 variance 在 deep network 中不會爆炸
- 跟 Pre-LN 的架構搭配：Pre-LN 已經控制 gradient flow，但這個 init 進一步控制 forward variance

**程式碼位置**: [`model.py:138`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L138) | [`model.py:144`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L144)

**替代方案**:
- **不使用 weight tying**（如 LLaMA）：vocab_size 夠大時，weight tying 的效益不明顯
- **DeepNet init**（如 DeepSeek-V2）：更複雜的 init 策略，用 `α` 控制 residual 路徑

**不適用情境**:
- 如果 vocab_size 跟 hidden_size 的比例懸殊（例如 vocab 很大但 hidden 很小），weight tying 可能成為 bottleneck
- 對非 residual 架構（如 RNN、SSM）不需要 scaled residual init

---

## Pattern 5: 二維/非二維參數分離的 Weight Decay

**是什麼**: 在 `configure_optimizers()` 中，將參數分成兩組——`dim >= 2` 的參數（weight matrices + embeddings）套 weight decay，`dim < 2` 的（biases + layernorm weights）不套 [model.py:270-274](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L270-L274)。

```python
decay_params = [p for n, p in param_dict.items() if p.dim() >= 2]
nodecay_params = [p for n, p in param_dict.items() if p.dim() < 2]
```

**為什麼有效**:
- Weight decay 的目的是讓 weight matrices 的值趨向於零（L2 regularization）
- Bias 和 layernorm 的 scale/shift 參數通常很小，對它們施加 weight decay 不會改善泛化，只會增加訓練困難
- 以維度判斷比以名稱判斷更穩健：不需要手動維護「哪些參數名稱不 decay」的列表

**程式碼位置**: [`model.py:263-287`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L263-L287)

**替代方案**:
- **按名稱過濾**：HuggingFace Transformers 的 `AdamW` 常用 `no_weight_decay` 列表指定
- **全部 decay**：對所有參數統一 weight decay（不推薦，會讓 bias 飄）

**不適用情境**:
- 如果有 1D 的 convolutional weights（如 `Conv1d`），它們的 `dim() == 1`，但理應要 decay。這個 pattern 對 pure transformer（沒有 conv layer）沒問題，但對混合架構要小心

---

## ML 工程品味的觀察

1. **對 abstraction 的態度**：nanoGPT 極度排斥間接層。沒有 Trainer、沒有 DataModule、沒有 Engine。連 config 都用 `exec()` 直接改 globals。這是 Karpathy 的一貫風格——程式碼層數越少，閱讀者越能追蹤控制流。但這也讓程式碼難以被重複使用或整合到更大的系統中。

2. **對效能的態度**：能用的加速手段都有——Flash Attention、torch.compile、TF32、autocast、fused AdamW、non_blocking transfer。但這些加速都是「一行 code 就能加」的程度，沒有引入任何複雜的效能優化（如 kernel fusion、triton 自訂 kernel）。這是刻意取捨：可讀性優先，但該快的地方還是要快。

3. **對可重現性的態度**：偏弱。沒有 pinned dependencies、沒有 `torch.backends.cudnn.deterministic`、checkpoint 不含 training configuration 的完整快照。這反映專案定位是「教育 + 展示」而非「實驗管理」。

4. **文件與程式碼的一致性**：程式碼內的註解品質極高——`from_pretrained()` 的 weight mapping 邏輯、`estimate_mfu()` 的 FLOPs formula 推導、`configurator.py` 開頭的「I know people are not going to love this」——這些註解同時是文件、設計決策記錄、以及作者的個人風格。這比寫一份獨立的設計文檔更有價值。
