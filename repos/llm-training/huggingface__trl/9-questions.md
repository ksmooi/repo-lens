---
repo: huggingface/trl
file: 9-questions
studied_at: 2026-05-25
commit_sha: 7877695
---

# TRL · 未解問題

## 還沒搞懂的設計決策

- [ ] **DPO 的 14 種 loss 變體，有多少在實務上真的有效？**
  - 我目前的推測：[UNVERIFIED] `sigmoid`（原始 DPO）和 `ipo`（Identity Preference Optimization）可能是最常用的，其他如 `sppo_hard`、`discopop`、`aot` 更像是研究用。但沒有實驗證據判斷
  - 相關程式碼：[`trl/trainer/dpo_trainer.py:1254`](https://github.com/huggingface/trl/blob/7877695/trl/trainer/dpo_trainer.py#L1254)

- [ ] **GRPO 的 `num_iterations` 與 `steps_per_generation` 為什麼分開？**
  - 兩者都控制「一次 generation 對應多少次 gradient update」，但 `num_iterations` 是 iteration count loop，`steps_per_generation` 是 gradient accumulation 層級的次數。我猜 `num_iterations` 是偏早期的實作（從原始論文來），而 `steps_per_generation` 是後來為 DeepSpeed 相容性加入的
  - 相關程式碼：[`trl/trainer/grpo_trainer.py:555-570`](https://github.com/huggingface/trl/blob/7877695/trl/trainer/grpo_trainer.py#L555)

- [ ] **為什麼 TRL v1 選擇捨棄 PPO、全面擁抱 GRPO？**
  - TRL 早期（v0.x）的核心是 PPOTrainer。v1 移除 PPO main trainer，保留 `trl/experimental/ppo/`。推測原因：GRPO 不需要 critic network，記憶體省很多、實作也乾淨很多。但 PPO 在某些場景（stateful RL、有 environment interaction 的場景）仍然是必要的
  - 相關程式碼：`trl/experimental/ppo/` 目錄

- [ ] **padding_free=True 為何在 v1 被標記為不可用？**
  - 程式碼中有註解說「needs a refactor to be used again」。padding-free 訓練（用 FlashAttention 的 packed sequences）理論上能節省大量記憶體，但跟 DPO 的 concat batch 策略、DataCollator 的動態 padding 衝突。這是架構層級的問題
  - 相關程式碼：[`trl/trainer/dpo_trainer.py:618`](https://github.com/huggingface/trl/blob/7877695/trl/trainer/dpo_trainer.py#L618)

## 想驗證的論文 claim

- [ ] **「GRPO memory efficient because it doesn't need a critic network」**
  - 這是真的——GRPO 用 group normalization 取代 critic，省下跟 policy model 差不多大小的 critic model 記憶體。但 GRPO 增加了 `num_generations` 倍的生成計算和 rollout buffer 儲存。在 large G 場景下（G > 16），rollout 的記憶體用量可能超過 critic 的節省
  - [UNVERIFIED] TRL 預設 G=8，這是個合理的平衡點

- [ ] **「vLLM integration speeds up GRPO training by 5-10x」**
  - 這是只比較 generation 階段的加速。整個 training step 包含 generation + reward + advantage + loss backward，vLLM 只加速 generation 的部分。整體加速倍率取決於 generation time 佔總訓練時間的比例

## 想問維護者的問題

- TRL v1 為什麼選擇用 `_LazyModule` 而不是 Python 3.7+ 的 `__getattr__` module-level 機制？
- `_BaseTrainer` vs transformers `Trainer` 的設計邊界在哪？什麼情況下會考慮直接 fork Trainer 而不是繼承？
- 未來是否計畫支援 vLLM 的多節點 generation（訓練在 GPU 0-3，vLLM 在 GPU 4-7）？
- 14 種 DPO loss 之中，團隊內部推薦哪些組合？

## 下次再看時的待辦

- [ ] 跑一次 GRPO 訓練，觀察 KL penalty 隨 `beta` 值變化的曲線
- [ ] 對照 LLaMA-Factory 的 GRPO 實作，看 TRL 跟它的設計差異
- [ ] 深入研究 `trl/experimental/` 中的 CPO 和 ORPO 實作，看它們跟 DPO 的關聯
- [ ] 追蹤 `vllm_generation.py` 的 `sync_weights()` 在 FSDP 下的行為——每次 sync 是否真的能及時 capture LoRA weights 的變化？

## 跨專案對照備忘

- **Trainer-Config 配對模式**：TRL: `DPOTrainer` + `DPOConfig`、`SFTTrainer` + `SFTConfig`。LLaMA-Factory 用單一 `dict` + `stage` flag 切換演算法。兩種做法的取捨——配對模式讓演算法參數隔離但重複較多，dict 模式讓切換簡潔但參數組織較亂。→ 候選 pattern（已在 LangChain、AutoGen 等看過類似的 Config per component 設計）
- **Lazy Module loading**：TRL 的 `_LazyModule` 在 transformers 生態中很常見（transformers 本身就用），但 PyTorch 生態（torchvision、torchaudio）多用 `__getattr__`。兩者在 import time 和 IDE 支援的取捨不同。→ 觀察中，尚未達到 3 repo 門檻
