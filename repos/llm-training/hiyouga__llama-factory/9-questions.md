---
repo: hiyouga/LLaMA-Factory
file: 9-questions
---

# LLaMA-Factory · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼架構中同時保留 v1 與 v2？v1 的實際使用狀況如何？**
  - v1 使用 plugin-based 架構（data_plugins、model_plugins、trainer_plugins），v2 則是完全扁平化的 modular design。
  - 根據 `cli.py:19-22`，v2 是預設路徑，v1 需要 `USE_V1=1` 環境變數啟用。
  - 我無法在 codebase 中找到明確的 v1 → v2 遷移時間點或文件，也無法判斷 v1 是否已被社群基本棄用。
  - 相關程式碼: [`src/llamafactory/cli.py:19-22`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/cli.py#L19-L22), [`src/llamafactory/v1/`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/v1/)
  - 我的推測: v1 可能是早期版本（論文發表時）的架構，後來被重寫為 v2，保留 v1 是為了向後相容。[UNVERIFIED]

- [ ] **`use_hyper_parallel` 和 `use_mca` 的優先級為何是 hyper_parallel > MCA？**
  - 在 `_training_function()` 中，`use_hyper_parallel` 首先被檢查（[tuner.py:91](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/tuner.py#L91)），然後才是 `use_mca`（[tuner.py:98](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/tuner.py#L98)）。
  - 但 `use_hyper_parallel` 只能搭配 SFT，而 `use_mca` 支援 PT / SFT / DPO。
  - 我不確定這個優先級是否有實際衝突的情境（用戶同時啟用兩者），還是純粹因為 hyper_parallel 整合更早完成。
  - 相關程式碼: [`src/llamafactory/train/tuner.py:91-112`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/tuner.py#L91-L112)

- [ ] **`ReporterCallback` 被固定在 callback 鏈的最後一位，背後的理由是什麼？**
  - [`tuner.py:89`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/tuner.py#L89) 明確標註 `# add to last`。
  - ReporterCallback 的 `on_train_begin` 會將 config push 到 W&B/TrackIO/SwanLab。放在最後一個執行是否確保其他 callback 不會干擾它的 state？
  - 相關程式碼: [`src/llamafactory/train/callbacks.py:433-488`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/train/callbacks.py#L433-L488)

## 想驗證的論文 claim

- [ ] **論文說支援 100+ LLM，但 codebase 中 template 是 121 個、mm_plugin 是 33 種 VLM。LLM 覆蓋是真實的 100+ 還是包含了不同 size 的同一架構？**
  - 從 template 數量來看（121），LLaMA-Factory 對「支援一個模型」的定義包含了不同版本（如 llama3、llama3.1、llama3.2）和不同微調變體（如 chat、instruct）。
  - 若以「獨立的模型架構家族」計算，數字更接近 40-50。
  - 相關程式碼: [`src/llamafactory/data/template.py`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/data/template.py)

- [ ] **論文中展示的 performance 數據是在哪種 packing 設定下取得的？neat_packing 還是標準 packing？**
  - packing 策略（`neat_packing` vs 標準 packing）對 throughput 和 loss curve 有顯著影響。
  - 論文中沒有明確標註使用的是哪種 packing 模式。
  - 相關程式碼: [`src/llamafactory/data/collator.py:481-538`](https://github.com/hiyouga/LLaMA-Factory/blob/16ff5a23/src/llamafactory/data/collator.py#L481-L538)

## 想問維護者的問題

- 為什麼選擇 OmegaConf 而非 Hydra？是否有特別的理由（如 dependency size 或使用體驗）？
- v1 的 plugin 架構為何被棄用？是 plugin 的 API 太複雜、還是維護負擔太重？
- `_training_function()` 的 dispatch 會隨著訓練階段增加而繼續使用 if/elif 鏈，還是計畫改成 registry-based dispatch？

## 下次再看時的待辦

- [ ] 讀懂 `greedy_knapsack()` 的 `search_for_fit()` 二進位搜尋演算法，確認其在 edge case（極短序列、極長序列）的行為
- [ ] 比較 `neat_packing` 和標準 packing 對 loss curve 的影響——預期 neat_packing 的 loss 更穩定，但 throughput 更低
- [ ] 深入讀 `ppo/trainer.py` 的 `ppo_train()` 迴圈，理解 PPO 在 LLaMA-Factory 中與標準 HF Trainer 的分離程度
- [ ] 查看 `dpo/trainer.py` 的 `compute_preference_loss()`，理解六種 preference loss（DPO/ORPO/SimPO/BCO/CPO/KTO）的實作差異

## 跨專案對照備忘

- **Template registration 系統** 與 Axolotl 的 `tokenizer_config` 概念相似。Axolotl 使用 JSON config 指定 tokenizer 的行為（如 `train_on_inputs`、`sequences_length`），而 LLaMA-Factory 使用更結構化的 Template class。
- **Patcher 鏈模式** 與 vLLM 的 `model_loader` + `model_patcher` 系統非常相似。vLLM 也有類似的 config-level 和 model-level 分層 patching 設計。→ **候選 pattern**: `patched-model-loading`（在 model load 的不同階段應用 patch）
- **Greedy knapsack packing** 已在 `_patterns/` 中有 `sequence-packing.md`（若存在）。此實現的獨特之處在於 multi-modal 支援（`PackingParams` 追蹤 media 的 sub-sequence 歸屬）。
