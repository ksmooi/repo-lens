---
repo: EleutherAI/lm-evaluation-harness
file: 9-questions
---

# LM Evaluation Harness · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 `multiple_choice` 要產生 K 個獨立的 loglikelihood requests 而不是用 batch 內的 masking 處理？**
  - 目前對每個選項產生一個獨立的 `Instance(request_type="loglikelihood")`，然後所有選項一起送給 LM。理論上可以用更高效的方式——在一個 forward pass 中計算所有選項的 logprob，透過 attention mask 區分不同選項的 context
  - 我目前的推測：[UNVERIFIED] 可能是為了保持 LM 介面的簡潔——`loglikelihood` 只需要處理 `(context, continuation)` 配對，不需要知道 evaluation 的結構
  - 相關程式碼：[`evaluator.py:560-562`](https://github.com/EleutherAI/lm-evaluation-harness/blob/95d5806/lm_eval/evaluator.py#L560-L562)

- [ ] **13,556 個 YAML task 檔案的維護成本有多高？**
  - 這麼多 YAML 檔案意味著幾乎沒有人 review 過所有 task 的正確性。`lm-eval validate` 的存在暗示了這個問題——但 validate 只能檢查結構正確性，不能檢查 prompt 設計是否合理
  - 我目前的推測：[UNVERIFIED] task 的正確性主要依賴於「在特定模型上跑出預期分數」作為回歸測試，而不是程式碼審查
  - 相關程式碼：[`_cli/validate.py`](https://github.com/EleutherAI/lm-evaluation-harness/blob/95d5806/lm_eval/_cli/validate.py)

- [ ] **`acc_norm` 的 normalization 是標準化「per continuation length」還是「per token」？**
  - 閱讀 evaluator 和 task.py 的 process_results，看到 `acc_norm` 的計算方式是「將 logprob 除以 continuation 的 token 數」。但不同 tokenizer 對同一個 continuation 的 token 數可能不同（例如 BPE vs Unigram），這會影響 `acc_norm` 的跨 tokenizer 可比性
  - 我目前的推測：[UNVERIFIED] 應該是除以 `len(tokenizer.encode(continuation))`，但我不確定是否在所有 LM 實作中都一致
  - 相關程式碼：[`api/task.py`](https://github.com/EleutherAI/lm-evaluation-harness/blob/95d5806/lm_eval/api/task.py)

- [ ] **Rank 0 的 `gather_object` 在大型 task + 多 GPU 時會不會 OOM？**
  - 目前的架構中，所有 rank 的結果（sample-level 資料、raw metrics）都會被 gather 到 rank 0。當 task 有 10 萬個 doc + 32 個 GPU，rank 0 的記憶體使用量 = 1 份正常結果 + 31 份完整結果。對於包含 prompt 字串的 sample logs，這可能很大
  - 相關程式碼：[`evaluator.py:667-697`](https://github.com/EleutherAI/lm-evaluation-harness/blob/95d5806/lm_eval/evaluator.py#L667-L697)

- [ ] **cache 的 hash key 碰撞機率與 invalidate 策略是什麼？**
  - 使用 `hash_string()` 對 request 參數做 hash 作為 cache key。但 prompt template 的小改動會改變 hash，導致 cache miss。同時，同一個模型對同一個 prompt 的 loglikelihood 應該是確定性的，但 float precision 差異可能導致 hash 不同
  - 相關程式碼：[`caching/cache.py`](https://github.com/EleutherAI/lm-evaluation-harness/blob/95d5806/lm_eval/caching/cache.py)

## 想問維護者的問題

- `lm_eval` 的 base install 在 0.4 版移除了 `transformers`/`torch` 依賴。為什麼選擇這個時機做這個 breaking change？是因為使用者反饋還是內部架構考量？
- 新增一個 output type（例如 code execution evaluation）的完整流程是什麼？需要修改哪些檔案？
- 有沒有計畫支援非同步 streaming evaluation（邊評估邊輸出結果）？還是目前「batch → gather → output」的設計有意為之？
- `!function` YAML tag 是如何處理 circular dependency 或 import error 的？有沒有 sandbox 機制？

## 下次再看時的待辦

- [ ] 探索 `lm_eval/tasks/leaderboard/` 的 Open LLM Leaderboard 任務群組結構 — 這些是「任務的任務」（groups of tasks），group aggregation 的實作值得深入
- [ ] 對照 DeepEval 的 metric 註冊系統和 Harness 的 Registry，看兩者的 trade-off
- [ ] 實際跑一次 `lm-eval run --model hf --model_args pretrained=gpt2 --tasks hellaswag`，觀察 log 輸出、執行時間、結果 JSON 格式
- [ ] 研究 `hf-multimodal` 和 `vllm-vlm` 的實作，看它們如何擴充 LM 抽象以支援圖片輸入

## 跨專案對照備忘

- `!function` YAML tag 的設計跟 Hydra/OmegaConf 的 `@_here_` resolver 類似 → 值得觀察是否演變為通用 design pattern
- Instance-based request/response decoupling（Pattern 2）跟 Ray 的 `ObjectRef` 概念類似 → 但 Harness 更輕量（shared mutable object vs 分散式 future）
- Registry-based lazy loading（Pattern 3）跟 `pytest` 的 entry point plugin system 以及 `setuptools` 的 `entry_points` 屬於同一類設計 → 候選 cross-repo pattern，已觀察到 2 個 repo（lm-evaluation-harness + pytest）
