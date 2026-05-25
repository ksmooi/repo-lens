---
repo: huggingface/trl
file: README
studied_at: 2026-05-25
commit_sha: 7877695
type: llm-training
language: Python
framework: PyTorch + HuggingFace Transformers
stars: 18.4k
status: active
---

# TRL · Transformers Reinforcement Learning

## 解決什麼問題

LLM 的 post-training（微調後階段）通常需要多種不同的演算法：SFT 做監督式微調、DPO / GRPO 做偏好最佳化、Reward Model 訓練獎勵模型。傳統上這些演算法各自有獨立的實作，整合與切換成本很高。

TRL 把這些演算法統一在同一個 framework 下——基於 HuggingFace `Trainer` 生態系，讓使用者用一致的 API 切換訓練方法，並自然繼承分散式訓練（Accelerate / DeepSpeed / FSDP）、量化（bitsandbytes）、LoRA（PEFT）等基礎設施。

## 為什麼值得研究

- **演算法覆蓋面最廣的 RLHF 函式庫**：從最基礎的 SFT，到 DPO 及其 14 種變體、GRPO 及其 8 種變體、KTO、Reward Model 訓練，全部在單一 codebase 內
- **vLLM 整合的示範級實作**：GRPOTrainer 整合 vLLM 做超快速 rollout generation，且支援 weight-sync + importance sampling correction——這是在單一 codebase 中同時做 training + inference 的稀有參考實作
- **HuggingFace 生態的 extension 範本**：TRL 的設計是「如何擴充 HuggingFace Trainer」的最佳參考——從自訂 Config、自訂 Data Collator、自訂 Loss，到 callback 機制、model card 自動產生
- **多模態支援**：支援 Qwen2.5-VL、Gemma 等 Vision-Language Model 的 post-training

## 任務與資料

| 面向 | 值 |
|------|-----|
| 任務類型 | RL (online) + Supervised (offline) + Preference (offline) |
| 輸入 | 文字 / 對話 / 多模態（文字+圖片） |
| 輸出 | 模型權重更新（相同架構） |
| 主要資料集 | Hub datasets（如 `trl-lib/Capybara`、`trl-lib/ultrafeedback_binarized`） |
| 評估指標 | reward score / accuracy / 自訂 metrics |

## 模型一句話

以 HuggingFace Transformers 為底層的 post-training 框架，支援 SFT / DPO / GRPO / KTO / Reward Modeling 等多種演算法。

## 競品比較

| 面向 | TRL | LLaMA-Factory | Axolotl | OpenRLHF |
|------|-----|---------------|---------|----------|
| 主要抽象 | HuggingFace Trainer extension | 自訂訓練框架 | 自訂訓練框架 | 自訂分散式框架 |
| 支援演算法 | SFT, DPO(14 variants), GRPO(8), KTO, Reward, RLOO | SFT, DPO, PPO, KTO, Reward 等 10+ | SFT, DPO, PPO, KTO, Reward | PPO, GRPO, DPO, Rejection Sampling |
| vLLM 整合 | 原生支援（weight sync + IS correction） | 無 | 無 | 有（分離式） |
| 分散式策略 | Accelerate (DDP/FSDP/DeepSpeed) | DeepSpeed | FSDP/DeepSpeed | 自訂（Ring-Attention） |
| 多模態支援 | 原生（Qwen2.5-VL, Gemma 等） | 有限 | 有限 | 無 |
| CLI 工具 | 有（trl sft / trl dpo / trl grpo） | 有 | 有 | 無 |
| 底層整合 | Transformers + PEFT + Accelerate | Transformers + PEFT | Transformers + PEFT | 自製分散式層 |
| 學習門檻 | 最低（HF 生態熟者） | 中 | 中高 | 高 |
| 主要 trade-off | 依賴 HF Trainer 架構限制（batch size 計算、log 週期等） | 支援模型最多但程式碼品質參差 | 設定檔複雜度高 | 效能最好但學習曲線最陡 |

## 健康度信號

- ⭐ Stars: ~18.4k
- 📅 最後 commit: 2026-05-25（非常活躍）
- 👥 主要維護者: HuggingFace 團隊（kashif、lvwerra 等）
- 🔬 已達 v1 里程碑，有完整 blog post 與文件

## 我會在後續筆記中回答的問題

- TRL 如何在不修改 Transformers Trainer 核心的情況下，擴展出 DPO / GRPO / SFT 等差異極大的演算法？
- GRPO 的 vLLM 整合是怎麼做到的？trainer model 與 vLLM engine 如何共享權重？
- 為何 TRL v1 選擇 GRPO（而非 PPO）作為主力 online RL 演算法？
- 14 種 DPO loss 變體的實作一致性與可取之處。
