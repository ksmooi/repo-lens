---
repo: unslothai/unsloth
file: 9-questions
---

# unsloth · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 DPO/KTO trainer 的 patch 幾乎是空的？**
  - `unsloth/models/dpo.py` 只有 26 行，`PatchDPOTrainer()` 和 `PatchKTOTrainer()` 都是空的 `return`
  - 但 `rl_replacements.py` 中有 `dpo_trainer_fix_columns` 和 `dpo_trainer_vision_process_row` 等函式
  - [UNVERIFIED] 推測 DPO 訓練只需要 LoRA kernel 的加速（MLP/QKV fusion 已涵蓋），不需要 GRPO 那種深度 code injection。DPO 的 loss 函式（`DPOLoss` / `KTOLoss`）沒有被 patch 的跡象

- [ ] **Offloaded gradient checkpointing 的效能 profile 是什麼？**
  - `apply_unsloth_gradient_checkpointing()` 在 seq >= 512 時啟用 offloaded gc，但閥值 512 的依據是什麼？
  - [UNVERIFIED] 推測這個數字與 GPU 的 HBM 容量 ÷ sequence length ÷ hidden dimension 的經驗公式有關
  - `unsloth_zoo.gradient_checkpointing` 的原始碼不在這個 repo 內，無法驗證 offloading 的具體實作（寫入 CPU pin memory vs disk？compression？）

- [ ] **`FP8` 在 consumer GPU 上的降級策略為何先 probe 後才決定？**
  - `fp8.py#test_has_fbgemm()` 在 runtime probe GPU 來判斷是否能執行 FBGEMM
  - [UNVERIFIED] 推測是 FBGEMM 在某些 consumer GPU（RTX 4090/5090）上因 compute capability 限制會 crash。probe 時用 `suppress_cuda_printf` 來防止錯誤訊息塞滿 console

- [ ] **model registry (`unsloth/registry/`) 的實際用途是什麼？**
  - registry.py 定義了 `ModelInfo` 和 `MODEL_REGISTRY`，且 `_check_model_info()` 會驗證 HF Hub 上的存在性
  - 但不確定這個 registry 在哪裡被消費——`loader.py` 中使用的是 `get_model_name()`（hardcode 映射表）而非 registry
  - [UNVERIFIED] 推測 registry 主要供 Unsloth Studio 的 Web UI 使用（顯示支援的模型清單給使用者選擇）

## 想驗證的論文 claim

- [ ] **「2x 訓練加速 + 70% VRAM 節省」的 benchmark 設定是什麼？**
  - README 聲稱的數字，需要確認對比的設定（LoRA rank、sequence length、GPU 型號、batch size）
  - 哪些數據來自 Unsloth 自測、哪些來自第三方（如 HF 的 benchmark）？
  - 加速在 4-bit QLoRA vs 16-bit full fine-tuning 上是否有差異？

## 想問維護者的問題

- 為什麼選擇 monkey-patch 而不是 subclass + transformer hook？hook 不破壞原始 class 但你們還是選了更 invasive 的做法，是因為 hook 無法修改 `__init__` 嗎？
- code injection 維護起來感覺如何？每當 TRL 更新時，你們需要手動檢查 template 中的每一個 `{placeholder}` 嗎？
- `uns_loth_zoo` 被設計成分離套件的理由是為了讓 MLX 路徑和其他工具共用？還是有其他考量？

## 下次再看時的待辦

- [ ] 讀 `unsloth_zoo` 的 `patching_utils.py` 和 `gradient_checkpointing.py`，這兩個才是實際執行 patching 的核心
- [ ] 追蹤 `kernels/fast_lora.py` 中 `LoRA_QKV` 的數學推導，看 QKV triples 的 LoRA fusion 跟 MLP 有何不同
- [ ] 實際跑一次 GRPO 訓練，觀察 `UnslothGRPOTrainer` 在 runtime 被動態建立的行為
- [ ] 比較 `fast_lora.py` 中的 `apply_lora_mlp_swiglu` vs `apply_lora_qkv` vs `apply_lora_o` 的實作差異

## 跨專案對照備忘

- **import-time monkey-patch**: 跟 `vLLM` 的 `patch_llama_attention()` 和 `FastChat` 的 model patching 是同一類模式 → 候選 pattern `import-time-model-patching.md`
- **Code injection for trainer extension**: `TRL` 的 `GRPOTrainer` + Unsloth 的模板注入，跟 `Axolotl` 的 `trainer_builder.py` 模式類似 → 候選 pattern（需在更多 repo 觀察到）
- **Offloaded gradient checkpointing**: 跟 `DeepSpeed` 的 `activation offloading` 概念類似，但實作層級不同（module level vs framework level）
