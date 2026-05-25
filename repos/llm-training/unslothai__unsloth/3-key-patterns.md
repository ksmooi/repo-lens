---
repo: unslothai/unsloth
file: 3-key-patterns
---

# unsloth · 值得偷學的設計

## Pattern 1: Import-time Monkey-patch 作為 Plugin 機制

**是什麼**: Unsloth 不要求使用者繼承任何類別。它在 `from unsloth import FastLanguageModel` 時直接修改 transformers、peft、trl 內部 class 的 `forward` 方法，讓既有程式碼「無痛加速」。

**為什麼有效**:

1. **零侵入性** — 使用者不需要改變程式碼結構，只需要安裝 unsloth 並替換 import
2. **全域生效** — patch 在 module level 發生，同個 process 中所有使用該 class 的程式碼都受影響（例如 transformers pipeline 內部的 attention 呼叫也會被 patch 到）
3. **可控範圍** — patch 在 `pre_patch()` → model load → `post_patch()` → adapter load → `patch_peft_model()` 的精確順序中逐步應用，每個階段有對應的 unwrap 機制

```python
# 關鍵實作模式 (llama.py:2187-2223)
def pre_patch():
    import transformers.models.llama.modeling_llama as LlamaModule
    LlamaModule.LlamaAttention.forward = LlamaAttention_fast_forward
    LlamaModule.LlamaDecoderLayer.forward = LlamaDecoderLayer_fast_forward
```

**替代方案**:
- **Subclass + custom `from_pretrained`**: 更乾淨但更侵入，需要使用者顯式指定類別。HF 的 `trust_remote_code` 機制其實就是這個路線
- **Transformer hook / register_forward_hook**: 更安全（不會破壞原始 class），但 hook 的返回值有限制（不能修改 input tensor 本身），且無法替換 `__init__`
- **Wrapper pattern**: 在 layer 外包一層 class，但無法影響到以原始 class name 做 type check 的程式

**何時可以借用**: 你的 library 想為既有框架提供「加速」或「修正」功能時。適合場景：效能優化、bug workaround、行為增強。**不適合**場景：API 變更、破壞性修改、安全敏感的應用。

**注意事項**: import-time monkey-patch 的除錯難度極高。Unsloth 的解決方案是：
1. 在 patched function 上設定 `__name__` 讓 traceback 可讀
2. 用 `_IS_MLX` 旗標在 import 初期就二分路徑，避免 MLX 跟 CUDA patch 互相干擾
3. 用 `__UNSLOTH_BACKWARDS_COMPATIBLE__` 標記防止 double-patch（`trainer.py:598`）

---

## Pattern 2: LoRA Fusion — 將 LoRA 的 forward/backward 融入 base linear

**是什麼**: 標準 PEFT LoRA 的計算分為 `X @ W` 和 `X @ A @ B @ scale` 兩個分離的矩陣乘法。Unsloth 用單一的 `matmul_lora(X, W, quant_state, A, B, scale)` 取代，對 4-bit 量化權重先 dequantize 再同時計算 full weight + LoRA 的貢獻。

**為什麼有效**:

1. **減少 I/O**: 標準做法中，weight 從 HBM 讀兩次（一次 base、一次 LoRA），fusion 後只讀一次
2. **減少 kernel launch**: 對 SwiGLU 的 gate/up/down 三個 projection + 各自的 LoRA adapter，標準做法是 6 個分離的 matmul + 1 個 SwiGLU activation + 1 個 matmul（down）。Unsloth 的 `LoRA_MLP.forward` 做到 3 個 fused matmul + 1 個 activation kernel
3. **LoRA backward 不 materialize 全矩陣**: [`fast_lora.py#L172-L173`](https://github.com/unslothai/unsloth/blob/eeb49d5/unsloth/kernels/fast_lora.py#L172-L173) 用 `addmm_` 直接計算 `dA @ B^T` 和 `A^T @ dY^T` 作為 LoRA 的 gradient，不建構完整的 merged weight 矩陣

```python
# 關鍵實作 (fast_lora.py:69-96)
class LoRA_MLP(torch.autograd.Function):
    def forward(ctx, X, gateW, gateW_quant, gateA, gateB, gateS, ...):
        # 三個 fused matmul + SwiGLU activation
        e = matmul_lora(X, gateW, gateW_quant, gateA, gateB, gateS)  # gate proj
        g = matmul_lora(X, upW, upW_quant, upA, upB, upS)            # up proj
        h = swiglu_fg_kernel(e, g)                                    # activation
        i = matmul_lora(h, downW, downW_quant, downA, downB, downS)   # down proj
```

**替代方案**:
- **標準 peft LoRA**: 用 `lora_A`、`lora_B` 層包在 base linear 前後。清楚但較慢
- **TorchAO / INT4 weight-only**: 只量化不 fusion，是正交技術
- **Fully-sharded fine-tuning**: 把 LoRA 丟了做全參數微調。記憶體成本高但無 LoRA 整合複雜度

**何時可以借用**: 你的系統有「小 adapter + 大 base weight」的組合場景，且 adapter 的計算不能忽略時。

**注意事項**: `matmul_lora` 內對 4-bit weight (`Bnb_Linear4bit`) 透過 `fast_dequantize()` 先 dequantize 回 float 再計算（`kernels/utils.py#L408`）。對於全精度（fp16/bf16）的 base weight，fusion 的收益非 kernel launch 而是減少 memory traffic。

---

## Pattern 3: Offloaded Gradient Checkpointing — 啟動值分流到 CPU/Disk

**是什麼**: 傳統 gradient checkpointing 在前向拋棄中間啟動值，反向時從最近的 checkpoint 重新計算。Unsloth 的 offloaded gc 將中間啟動值**寫到 CPU 記憶體或磁碟**，而非拋棄。

**為什麼有效**:

1. 重新計算需要完整的 forward pass，代價是時間（約 `num_layers * (forward_time)`）。Offloading 的代價是記憶體頻寬（D2H/H2D copy），通常比重新計算快
2. 當序列長（max_seq_length >= 512）時，offloading 的每層共享啟動值很大，D2H 頻寬利用率高，offloading overhead 被分攤到每層
3. [UNVERIFIED] Unsloth 實作從 `unsloth_zoo.gradient_checkpointing` 匯入 `Unsloth_Offloaded_Gradient_Checkpointer`，但筆者未追蹤此模組原始碼

```python
# 關鍵邏輯 (_utils.py:195-226)
def apply_unsloth_gradient_checkpointing(use_gradient_checkpointing, max_seq_length, dtype):
    if use_gradient_checkpointing == "unsloth":
        if max_seq_length < 512:
            return True  # 短序列 → 原生 gc
        patch_unsloth_smart_gradient_checkpointing(dtype)
        return "unsloth"
```

**替代方案**:
- **PyTorch 原生 `torch.utils.checkpoint`**: 標準 re-computation 策略
- **Activation checkpointing (FSDP)**: FSDP 的 model sharding 內含 activation offloading
- **Recompute + activation compression**: 量化中間啟動值（如 GACT），減少 D2H 頻寬需求

**何時可以借用**: 你有比 GPU 空閒的 CPU 記憶體（通常是的）、且 sequence 夠長讓啟動值總量大過 offloading 的 latency 時。

**注意事項**: DDP 下 offloaded/re-entrant gc 會引發 `"marked ready twice"` 錯誤（`vision.py#L1586-L1599`），Unsloth 在 vision 模型中強制使用 non-reentrant 原生 gc。

---

## Pattern 4: Code Injection 做 Trainer 擴展（GRPO 模板引擎）

**是什麼**: Unsloth 的 GRPO 支援不是繼承 `trl.GRPOTrainer`，而是用字串模板動態產生新的 `UnslothGRPOTrainer` class。

**為什麼有效**:

1. **TRL 版本迭代極快**（GRPO 從 TRL 0.20 到 0.28 API 改了多次）。用 code injection 可以只替換變動的方法，保留大部分原始碼
2. **可以修改 Private 方法** — 那些以 `_` 開頭的內部方法（如 `_generate_and_score_completions`）無法透過繼承覆寫，但 code injection 可以直接改
3. **可注入 module-level helper function** — `RL_PRE_ITEMS` 插入的函式可以在 trainer class 外部定義，不污染 namespace

```python
# 模板引擎概觀 (rl.py:746-880)
def _patch_trl_rl_trainers_impl(trainer_file):
    # 1. 掃描原始 trainer 的原始碼
    source = inspect.getsource(trl.GRPOTrainer)
    # 2. 取代特定方法 signature + body
    for old, new in RL_FUNCTIONS.items():
        source = source.replace(old, new)
    # 3. 注入 helper functions
    source = RL_PRE_ITEMS + source
    # 4. 用 create_new_function() compile
    exec(source, globals())
    return locals()["UnslothGRPOTrainer"]
```

**替代方案**:
- **Task-specific subclass**: 更乾淨，但無法觸及 private 方法
- **Metaclass + decorator**: 比 code injection 安全但更複雜
- **Wrapper pattern**: 在 trainer 外包一層，但無法改動 `trl.trainer` module 的內部類別註冊

**何時可以借用**: 你的擴展需要修改一個快速迭代的第三方套件內部，且 subclass 不夠用。**不適用**場景：安全性敏感的環境（exec 原始碼是潛在風險）。

**注意事項**: `create_new_function()` 來自 `unsloth_zoo` — 底層使用 `exec()`。這意味著如果 TRL 的新版本在原始碼中引入了安全漏洞或惡意程式碼，code injection 不會提供任何保護。另需注意 template 相容性：TRL 版本變動導致函式 signagure 改變時，template 中的 `{placeholders}` 會斷掉。

實際的 template 內容定義在 [`rl_replacements.py`](https://github.com/unslothai/unsloth/blob/eeb49d5/unsloth/models/rl_replacements.py#L53-L58)，每個 replacement function（如 `grpo_trainer__generate_single_turn`）都是純字串函式，回傳替換用的程式碼片段。

---

## Pattern 5: 版本感知的相容性矩陣

**是什麼**: Unsloth 在 import time 檢查 transformers 版本，用 `SUPPORTS_*` 常數決定支援哪些模型架構和功能。

```python
# loader.py:68-83
SUPPORTS_FOURBIT = transformers_version >= Version("4.37")
SUPPORTS_GEMMA = transformers_version >= Version("4.38")
SUPPORTS_GEMMA2 = transformers_version >= Version("4.42")
SUPPORTS_QWEN3 = transformers_version >= Version("4.50.3")
SUPPORTS_GEMMA4 = transformers_version >= Version("5.5.0")
_NEEDS_ROPE_FIX = transformers_version >= Version("5.0.0")
```

**為什麼有效**: HuggingFace transformers 的版本迭代非常快（每週 release），新模型追加新 class、舊 class 的 API 也在變。用版本常數做條件式 import 和條件式 patch，讓 Unsloth 的單一版本可以支援 transformers 4.37 到 5.5+ 的巨大範圍。

**替代方案**:
- **try/except import**: 常見做法，但無法區分「class 不存在」跟「other import error」
- **動態 `__import__` + `hasattr`**: 同上，精細度不夠
- **pip `install_requires` 鎖死版本**: 會讓使用者無法更新 transformers

**何時可以借用**: 你的 library 深度依賴一個快速迭代的上游套件，且需要支援多個大版本跨度。

---

## ML 工程品味的觀察

Unsloth 對 abstraction 的態度很特別：**在 API 表面極簡，在內部極度 hacky**。API 只有 `from_pretrained` 和 `get_peft_model` 兩步驟；而內部 monkey-patch 到 transformers 的底層 class、code injection 到 trl 的原始碼、用 exec 動態建立 class。

這是對「使用者體驗」和「實作優雅」之間清晰的取捨——他們完全偏向前者。這跟 LLaMA-Factory 的 YAML config 路線形成對比：LLaMA-Factory 是「可讀的設定檔」，Unsloth 是「跑就對了」。

另一個觀察是 **unsloth_zoo 的分離策略**：kernel 實作在 `unsloth/kernels/`，但 patch orchestration（找正確的 loss function、管理 patch lifecycle）放在 `unsloth_zoo`。這讓 `unsloth_zoo` 可以獨立於 GPU 架構存在，MLX 路徑也能共用部分工具函式。
