---
repo: karpathy/nanoGPT
file: 9-questions
studied_at: 2026-05-25
commit_sha: 3adf61e
---

# nanoGPT · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 vocab_size 挑 50304 這個數字？**
  - GPT-2 原 vocab_size 是 50257，50304 是最接近的 64 倍數。但為什麼是 64？128 或 32 不行嗎？
  - 我目前的推測：nanoGPT 的 `n_embd=768`，head_dim = 768/12 = 64。矩陣乘法的 tensor core 偏好 alignment 到 8 或 16 的倍數，但更可能只是因為 `int(50257/64)*64 + 64 = 50304` 這個計算最直覺 [UNVERIFIED]
  - 相關程式碼: [`model.py:111`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L111)

- [ ] **Weight tying 跟 torch.compile 的警告**
  - `model.py:135-137` 註解提到 "functional_call was passed multiple values for tied weights. This behavior is deprecated and will be an error in future versions"，但作者標了 `TODO investigate`，沒有進一步處理
  - 這是 PyTorch 2.0 compile 跟 weight tying 的已知衝突嗎？具體的錯誤場景是什麼？
  - 相關程式碼: [`model.py:135-138`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L135-L138)

- [ ] **`crop_block_size()` 為什麼只截斷 attention bias 而不處理 position embedding 的對應？**
  - 等等，wpe.weight 有被截斷（`[:block_size]`）。但 attention bias 的截斷方式 `[:,:,:block_size,:block_size]` 是對的嗎？block_size 減少時，左上角的 submatrix 確實是正確的 causal mask。沒問題。

- [ ] **為什麼 `from_pretrained()` 只允許 override dropout？**
  - `override_args` 的 `assert all(k == 'dropout' for k in override_args)` [model.py:211](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L211)
  - 如果要 override 其他參數（如改變 block_size 或 dropout 以外的東西），只載入 weights 再手動調整可行嗎？
  - 我目前的推測：weight mapping 的 assert 邏輯需要 pair-wise 比對 key 的 shape，任何跟 checkpoint 不同的 shape 都會使 assert 失敗。所以只允許 dropout 這種不影響 shape 的 override。[UNVERIFIED]

## 想驗證的論文 claim

- [ ] **Scaled residual init 的實際影響**
  - GPT-2 paper 提到了 residual projection 的 scaled init（`std=0.02/√(2*n_layer)`），但沒有 ablation study
  - 論文說這樣可以讓 12+ layer 的 transformer 在訓練初期更穩定。但能否在 8 layer 的模型上觀察到差異？
  - 相關程式碼: [`model.py:143-145`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L143-L145)

- [ ] **Weight tying 在 124M 模型上的實際效益**
  - 論文宣稱 weight tying 能改善 perplexity（尤其對小模型）
  - nanoGPT 的 124M 參數中，weight tying 節省約 38M 參數。如果 disable weight tying（讓 lm_head 獨立），val loss 會變化多少？
  - 這個實驗很容易做：註解掉 [`model.py:138`](https://github.com/karpathy/nanoGPT/blob/3adf61e/model.py#L138) 這行，跑 eval 比較就行

## 想問維護者的問題

- nanochat 跟 nanoGPT 的主要架構差異是什麼？為什麼需要一個新的 repo 而不是在 nanoGPT 上改版？
- 有沒有考慮過把 KV cache 加進 `generate()`？還是覺得 inference 的效能瓶頸不影響這個 repo 的定位？
- deprecated 之後，nanoGPT 還會接受 bug fix PR 嗎？

## 下次再看時的待辦

- [ ] 對照 HuggingFace transformers 的 `GPT2Model.forward()`，看看 nanoGPT 省略了哪些功能（gradient checkpointing？KV cache integration？tie_weights 的差異？）
- [ ] 讀一下 `bench.py` 的實作，跟 `train.py` 的 training loop 有多大的差別
- [ ] 實驗：disable weight tying 跑一次 eval，觀察 val loss 變化
- [ ] 讀 `nanogpt` 的 `transformer_sizing.ipynb` 和 `scaling_laws.ipynb`，看 Karpathy 對 scaling law 的模擬

## 跨專案對照備忘

- **exec() config pattern**：nanoGPT 的 `configurator.py` 用 `exec()` 直接修改 globals。這在 paper_lens 的 repo 中還沒看到類似做法。跟 dlib 的 C++ `exec()`-like config 是同一類設計。
- **gradient accumulation + DDP sync skip**：HuggingFace transformers Trainer 用 `model.no_sync()` context manager 實現相同的效果。nanoGPT 直接設 `require_backward_grad_sync` 屬性，是更簡潔但依賴內部 API 的方式。
- **np.memmap DataLoader**：跟 MosaicML StreamingDataset 的設計理念不同——後者為了 streaming 而設計（網路讀取 + 即時 decompress），nanoGPT 的 memmap 假設資料在本地 NVMe SSD 上。
