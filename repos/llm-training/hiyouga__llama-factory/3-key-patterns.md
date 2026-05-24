---
repo: hiyouga/LLaMA-Factory
file: 3-key-patterns
---

# LLaMA-Factory · 值得偷學的設計

## Pattern 1: 單一入口 + 策略模式調度

**是什麼**: LLaMA-Factory 使用 `_training_function()` 作為所有訓練階段的單一入口，內部根據 `finetuning_args.stage` + 優先級 flag（`use_hyper_parallel` > `use_mca` > `stage`）進行 dispatch。

**為什麼有效**: 這種設計將「訓練流程管理」的 boilerplate（config parsing、callback assembly、distributed init、checkpoint handling）集中在一個地方，消除了每個訓練階段重複這些邏輯的必要。6 個訓練階段 × 3 個 backend（standard / hyper_parallel / MCA）= 18 種組合，全部透過同一個 130 行的函數管理。

**程式碼位置**: [`src/llamafactory/train/tuner.py:68-137`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/tuner.py#L68-L137)

**何時可以借用**:
- 你的系統有 N 個「流程類別」，每個流程共用 80% 以上的初始化邏輯
- 流程之間的差異可以參數化表達（一個 `stage` 字串或列舉值）
- 新流程的添加頻率適中（不至於每天加一個導致 dispatch 函數膨脹）

**替代方案**:
- **Plugin-based load + register 模式**：每個流程註冊自己的 handler，主調度器遍歷註冊表（如 Django URL dispatcher）。更乾淨，但註冊表的查找時機複雜（何時發現新的流程類別？）。
- **Subclass + factory method**：用 ABC 定義 `TrainingWorkflow`，每個 workflow override `run()`。這是更 OOP 的做法，但如果不搭配 registry，呼叫方仍然需要 if/elif。
- **Config-driven pipeline**：用 YAML 描述 pipeline 步驟（如 Airflow DAG），執行引擎根據描述執行。最靈活但需要 DSL 設計。

**注意事項**: 當分支條件超過 3 層（優先級 × stage × backend），`_training_function()` 的 `if/elif` 鏈開始難以閱讀。LLaMA-Factory 的做法是保持 dispatch 函數只做 routing，不包含超過 5 行的邏輯——每個 case 立刻委派給獨立的 `workflow.py`。

---

## Pattern 2: Patcher 鏈

**是什麼**: LLaMA-Factory 將所有對 HuggingFace 模型、config、tokenizer、processor 的客製化修改組織為一個 patcher 鏈——6 個 patch 函數，在模型生命週期的不同時間點執行。

```
patch_config()     → 載入模型前，修改 config（attention、RoPE、quantization、MoE）
patch_tokenizer()  → 載入 tokenizer 後，修復 pad、max_length、special tokens
patch_processor()  → 載入 processor 後，設定圖像/影片/音訊參數
patch_model()      → 載入模型後，修復 generation、prepare training
patch_valuehead_model() → PPO/RM 用的 value head 代理
```

**為什麼有效**: 這種設計將「對 HF 生態的補丁」從 model 類別中抽離出來，保持 model class 乾淨。更重要的好處是每個 patch 可以獨立測試、獨立開關。例如 `patch_config().configure_quantization()` 可以只設定量化相關的 config，不影響 attention 的選擇。

**程式碼位置**: [`src/llamafactory/model/patcher.py`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/model/patcher.py)（466 行）

**何時可以借用**:
- 你需要修改一個外部 library 的行為（monkey-patch），但又想讓修改可追蹤、可測試
- 修改有明確的生命週期階段（config 在載入前、tokenizer 在載入後、model 在初始化後）
- patch 的數量會隨時間增加（如支援新模型時）

**替代方案**:
- **直接 subclass 第三方類別**：不被推薦，因為 HuggingFace 的類別相互依賴複雜（`Trainer` 內部會建立 `Optimizer`、`Scheduler` 等），subclass 可能遺漏初始化順序。
- **Hook-based 回呼**：讓 HF 的 `from_pretrained()` 接受 callback。HF 不提供這個機制，所以 LLaMA-Factory 的做法是實務上最乾淨的方案。
- **Wrapper/Decorator**：用 decorator 包裝 HF 的方法。適用於單一方法的修改（如 `generate()`），不適用於 config 的多處修改。

**注意事項**: Patcher 鏈的依賴順序是敏感的——`patch_model()` 依賴 `patch_config()` 已設定的 `_attn_implementation`。新增 patch 時要清楚知道自己的插入點。LLaMA-Factory 用註解標記每個 patch 的執行時間點來管理這個複雜度。

---

## Pattern 3: 可插拔 Optimizer 系統

**是什麼**: 透過 override `Trainer.create_optimizer()`，在 HuggingFace Trainer 中嵌入 7 種自訂 optimizer，無需修改訓練迴圈。

```python
# 每個 custom trainer 的 create_optimizer:
def create_optimizer(self):
    custom_optimizer = create_custom_optimizer(
        self.model, self.args, self.finetuning_args, ...
    )
    if custom_optimizer is not None:
        return custom_optimizer
    return super().create_optimizer()
```

**為什麼有效**: 這個模式的核心是「回退鏈」（fallback chain）——先檢查是否匹配 custom optimizer，若不匹配則回退到 HF 預設的 AdamW。這讓新增一個 optimizer 變體只需要實作一個 factory function，註冊進 `create_custom_optimizer()` 的查詢表，不需要接觸 trainer class。

**程式碼位置**:
- Dispatcher: [`src/llamafactory/train/trainer_utils.py:527-549`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/trainer_utils.py#L527-L549)
- Individual optimizers: [`trainer_utils.py:200-525`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/trainer_utils.py#L200-L525)

**何時可以借用**:
- 你的訓練系統有多種 optimizer 選擇，且選擇可以在 config layer 決定
- 使用者需要在不同情境下切換 optimizer 而不改動訓練程式碼
- 每個 optimizer 的參數化開銷不大（提供幾個 config 欄位即可）

**替代方案**:
- **`torch.optim` 的註冊表 + `build_optimizer()`**：PyTorch 2.0+ 的 `torch.optim` 支援依名稱建立 optimizer。LLaMA-Factory 沒選這個方案，因為 GaLore、APOLLO 等 optimizer 需要額外的模型分析（如參數分組）和 state 管理，不是單純的 `cls(params, lr)` 可以表達。
- **Optax-style transform 組合**：JAX 生態的 optimizer 由多個 transform 組合而成（如 `chain(adam(1e-3), clip_by_global_norm(1.0))`）。這在 PyTorch 生態不常見，且與 HuggingFace Trainer 整合更困難。

**注意事項**: 不是所有 optimizer 都適合這個模式。BAdam 需要修改 backward 的順序（逐區塊計算 gradient），所以除了 optimizer 本身還需要一個 `BAdamCallback` 來 hook 訓練迴圈。當 optimizer 需要的修改超出 `optimizer.step()` 的範圍時，就需要擴展 hook 點。

---

## Pattern 4: Registration-based Conversation Template 系統

**是什麼**: 121 個 conversation template 透過 `register_template()` 註冊到全域 `TEMPLATES` 字典，運行時根據模型名稱或 data_args 查找。每個 template 決定 user/assistant/system/tool 訊息在 tokenizer encode 時的格式。

```python
# deepseek3 template 的註冊
register_template(
    "deepseek3",
    format_user=StringFormatter(...),
    format_assistant=StringFormatter(slot="<|assistant|>\n{{content}}<｜end▁of▁sentence｜>"),
    format_system=StringFormatter(slot="<|system|>\n{{content}}"),
    default_system=...,
    stop_words=["<｜end▁of▁sentence｜>"],
    efficient_eos=True,
)
```

**為什麼有效**: 這是一個典型的「聲明式 vs 命令式」的選擇——不是用 if/else 描述「如果 model 是 deepseek3，user 訊息要加什麼 prefix」，而是用資料結構描述「deepseek3 的 user 格式是 X」。這讓新增一個 template 完全不需要改動任何控制流。

**程式碼位置**: [`src/llamafactory/data/template.py:490-562`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/data/template.py#L490-L562)（register_template），各 template 註冊從 line 588 開始

**何時可以借用**:
- 你的系統需要支援多種「格式」，每種格式有固定的 pattern 但細節不同
- 格式數量會隨著生態系統成長而線性增加
- 每種格式的差異可以用參數化的 data class 描述

**替代方案**:
- **Jinja template string**：這是 HuggingFace 的作法（tokenizer 的 `chat_template` 屬性）。好處是動態、不需要 code change。壞處是除錯困難、執行時錯誤多。LLaMA-Factory 兩者都支援——優先使用內建 template，`template=None` 時 fallback 到動態解析 `chat_template`。
- **Hub-based template 下載**：從 HuggingFace Hub 下載特定 repo 的 template。這是一個更 distributed 的方案，但引入了網路依賴和版本管理問題。

**注意事項**: 121 個 template 的維護確實是工作量。LLaMA-Factory 的做法是將 template 定義集中在一個檔案（template.py），每個 model family 約 10-20 行，並提供 `parse_template()` 作為自動化 fallback。如果你的系統只有 5-10 種格式，registration 的開銷值得；若超過 50 種，綜合考慮 build 一個基於 YAML 的外部 template 定義檔案。

---

## Pattern 5: Greedy Knapsack Sequence Packing

**是什麼**: 將多個不等長的訓練 sequence 包裝進同一個 `cutoff_len` 區塊，減少 padding token 的浪費。使用 greedy knapsack 演算法（搭配 binary search 尋找最佳 bin count）來最大化填充效率。

```python
def greedy_knapsack(sequences: list, knapsack_size: int) -> list[list]:
    """Pack sequences into bins using greedy algorithm with binary search."""
    fit_count = search_for_fit(sorted_seq, knapsack_size)
    bins: list[list] = [[] for _ in range(fit_count)]
    bin_weights: list[int] = [0] * fit_count
    for seq in sorted_seq:
        chosen_bin = min(range(fit_count), key=lambda i: bin_weights[i])
        bins[chosen_bin].append(seq)
        bin_weights[chosen_bin] += len(seq)
    return bins
```

**為什麼有效**: 對 LLM 訓練來說，sequence 長度分佈通常非常偏態（多數 sequence 遠短於 cutoff_len）。沒有 packing 時，padding token 可能佔總 token 量的 30-50%。Packing 可以將這個浪費降到接近 0%，且訓練的語意不變（因為每個 sequence 在 packed 區塊內有獨立的 label mask）。

**程式碼位置**: [`src/llamafactory/data/processor/processor_utils.py:48-73`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/data/processor/processor_utils.py#L48-L73)

**何時可以借用**:
- 你的序列長度分佈偏態嚴重（多數短，少數長）
- 你使用 decoder-only transformer，且 attention 沒有封裝進 mask
- 訓練 throughput 是瓶頸，且你有足夠硬體批次處理加速

**替代方案**:
- **Dynamic batching**：每次取樣時依長度分組（sorted batching）。簡單但需要 shuffle-aware 實作，且長尾序列（少數極長）仍然會拖慢 batch。
- **Static padding**：最簡單，直接 pad 到 cutoff_len。最浪費計算資源，但不引入 packing 的複雜度。
- **最佳化 bin packing**：First-Fit Decreasing (FFD) 或更複雜的演算法。LLaMA-Factory 選擇 greedy knapsack 是因為序列長度分佈在 cutoff_len 處被截斷，greedy 在實務上已經足夠好。

**注意事項**:
1. **Cross-sequence attention**：標準 packing 下每個 token 可以 attend 到同 batch 其他 sequence 的 token。多數情況下影響不大（因為 label mask 確保 loss 只在自己的 sequence 上計算），但對 position encoding 有影響。LLaMA-Factory 的 `neat_packing` 模式使用 4D block-diagonal attention mask 避免這個問題。
2. **Multi-modal 相容性**：packing 時圖像/影片/音訊需要標記屬於哪個 sub-sequence。LLaMA-Factory 使用 `PackingParams` 來追蹤這個對應關係。
3. **Packing + FA2**：Flash Attention 2 預設不支援 packed sequence。LLaMA-Factory 透過 `_unpad_packed_features()` 移除 padding token 讓 FA2 可以正確處理。

---

## Pattern 6: 雙層 Config 解析（OmegaConf + HfArgumentParser）

**是什麼**: 先使用 OmegaConf 解析 YAML/JSON 檔案並處理 CLI override，再將結果 dict 餵給 HuggingFace 的 `HfArgumentParser` 產生 typed dataclass。

```python
def read_args(args=None):
    if sys.argv[1].endswith(".yaml"):
        dict_config = OmegaConf.load(Path(sys.argv[1]))
        override = OmegaConf.from_cli(sys.argv[2:])
        return OmegaConf.to_container(OmegaConf.merge(dict_config, override))
```

**為什麼有效**: OmegaConf 提供了方便的 merge 語法和 CLI override 支援（`--key=value`），這不是 HuggingFace 的 `HfArgumentParser` 原生支援的。但最終的 typed dataclass（`TrainingArguments` 等）是 HuggingFace 生態的標準介面。這個兩層架構同時拿到了兩個世界的優點。

**程式碼位置**: [`src/llamafactory/hparams/parser.py:85-99`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/hparams/parser.py#L85-L99)

**何時可以借用**:
- 你的系統使用 HuggingFace 的 `TrainingArguments` 等 dataclass，但需要 YAML config 支援
- 你需要方便的 CLI override 語法（`--key=value`）
- 你的 config 檔案格式需要支援巢狀結構

**替代方案**:
- **只使用 OmegaConf**：全量使用 OmegaConf config，捨棄 `HfArgumentParser`。這需要自行維護 typed field 解析，且無法使用 HuggingFace 的 `TrainingArguments` 的後處理邏輯。
- **只使用 HfArgumentParser**：使用 `parse_yaml_file()` 功能。但不支援巢狀覆蓋、CLI override 語法較笨拙。
- **使用 Hydra**：更完整的 config 管理系統，但引入了較大的依賴。LLaMA-Factory 選擇 OmegaConf 可能是因為它輕量（`pip install omegaconf` 即可）且 merge 邏輯簡單。

---

## ML 工程品味的觀察

**對 abstraction 的態度**: LLaMA-Factory **極度務實**。它不是追求優雅的 OOP 設計，而是追求「讓用戶最少改動就能跑起來」。證據：`find_all_linear_modules()` 的 default 是 `"all"` 而不是 PEFT 預設的 attention-only，因為實證效果更好；Template 系統使用 registration 而非 if/else 是因為數量真的大到需要用 data-driven 方式管理；`patcher.py` 的 monkey-patch 不優雅但確保兼容性。

**對效能跟可讀性的取捨**: LLaMA-Factory 明顯偏向效能。Packing 是大改資料結構來換 throughput；多種 optimizer 實作雖然增加了 `trainer_utils.py` 的長度（978 行），但每一種都是真實的 memory/speed 提升。Web UI 的 `LogCallback` 使用 thread pool 寫檔更是「為了不阻塞訓練什麼都能做」的態度。

**對可重現性的重視程度**: 中等偏上。有 seed 設定、checkpoint resume、`tokenized_path` cache、push to Hub 支援。但沒有資料版本控制（DVC）整合，實驗管理依賴外部工具（W&B / SwanLab）而非框架內建。這符合其定位——微調框架專注於訓練本身，實驗管理留給生態系統。
