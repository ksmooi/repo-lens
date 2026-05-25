---
repo: karpathy/llm.c
file: 9-questions
studied_at: 2025-06-26
commit_sha: f1e2ace
---

# llm.c · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 CPU 版的 encoder_backward 用累加（+=）到 dwte/dwpe，而非像 GPU 版那樣用桶排序？**
  - 我目前的推測：CPU 版是 fp32 單執行緒，沒有 race condition 問題。加法的順序固定，結果是確定性的。GPU 版需要桶排序是因為多個 thread 可能並行更新同一個 position。[UNVERIFIED]
  - 相關程式碼：[`train_gpt2.c:60-76`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.c#L60-L76) vs [`encoder.cuh:47-117`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/encoder.cuh#L47-L117)

- [ ] **為什麼選擇 single CUDA stream 而非多 stream pipeline？**
  - 我目前的推測：多 stream 雖然可以重疊 data transfer 和 compute，但 forward/backward 的 kernel 彼此資料相依性強（後一個 kernel 需要前一個的結果），實際重疊效益有限。Karpathy 選擇了程式碼簡單度。[UNVERIFIED]
  - 相關程式碼：[`train_gpt2.cu:1183`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L1183)

- [ ] **為什麼 padded_vocab_size = 50304（% 128 == 0）？為何選擇 128 而非 32 或 64？**
  - 我目前的推測：128 是因為 `Packed128` 在 BF16 下每次載入 8 個 float16（8×16=128 bits），128=8×16=128。所以 128 是向量化載入的自然對齊單位。如果選 64（4×float），在 BF16 下反而沒對齊。[UNVERIFIED]
  - 相關程式碼：[`train_gpt2.cu:583`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.cu#L583)

- [ ] **fused_residual_forward5 的「5」是指哪五個操作被融合？**
  - 我目前的推測：residual_add（skip connection 的 x 加上殘差分支的 dx）+ layernorm 的四個步驟（mean、var、norm、scale+shift）。但實際上 kernel 程式碼中看起來只有 2-3 步。這個命名可能指 kernel 內部處理了 5 種不同的路徑或 case。[UNVERIFIED]
  - 相關程式碼：[`layernorm.cuh:142-219`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/layernorm.cuh#L142-L219)

- [ ] **CPU 版的 `crossentropy_softmax_backward` 公式為何不直接用 `(probs - indicator) * dloss`？**
  - 我目前的推測：CPU 版的實作更接近「先用 softmax + cross-entropy 的解析導數直接計算」，而不是先求 probs 再減 one-hot。可能是為了數值穩定性（避免 probs 太小或太大時的精確度損失）。[UNVERIFIED]
  - 相關程式碼：[`train_gpt2.c:502-521`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.c#L502-L521)

## 想驗證的論文 claim

- [ ] **「llm.c 比 PyTorch 快 ~7%」的真正原因**
  - README claim 是比 PyTorch Nightly 快 7%。我的推測是：1) llm.c 沒有 autograd tape 維護的 overhead；2) cuBLASLt 的 epilogue 融合比 PyTorch 的 `torch.nn.functional.linear + F.gelu` 少一個 kernel launch；3) 自訂的 layernorm kernel 比 PyTorch 的更針對性。[UNVERIFIED]
  - 相關程式碼：README 但需要 benchmark 驗證。

- [ ] **WSD scheduler 在什麼場景下優於 cosine scheduler？**
  - WSD（Warmup-Stable-Decay）的假設是最後 20% 步驟急遽 decay 可以讓模型收斂到更好的局部最小值。這對 llm.c 這樣的教育性 repo 有意義嗎？還是只是為了展示另一種 schedule 實作？[UNVERIFIED]
  - 相關程式碼：[`schedulers.h:64-80`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/llmc/schedulers.h#L64-L80)

## 想問維護者的問題

- 是否有計畫支援 Flash Attention 手寫實作（不用 cuDNN），以便在無 cuDNN 的環境也能用 O(B×NH×T) 的顯存運行 attention？
- `L * B * NH * T * T` 的 attention matrix 實體化在序列長度 4K+ 時完全不可行 — 是否實驗室內部已經有手寫的 flash attention 版本但未合併到 main？
- 為什麼決定不支援 fp8 精度（H100 支援）？是時程優先還是覺得 fp8 對教學 repo 太複雜？

## 下次再看時的待辦

- [ ] 實際跑一次 `make train_gpt2 && ./train_gpt2` 觀察 loss 曲線與取樣文字
- [ ] 對照 `train_gpt2.c` 和 [`train_gpt2.py`](https://github.com/karpathy/llm.c/blob/f1e2ace651495b74ae22d45d1723443fd00ecd3a/train_gpt2.py) 的 loss 曲線是否一致
- [ ] 比較 `test_gpt2.c` 中期望的 loss 序列跟實際用 PyTorch 跑出來的是否吻合
- [ ] 深入看 `zero.cuh` 的 TCP/MPI/File system NCCL ID 分發三種實作差異
- [ ] 讀懂 `dev/cuda/` 目錄下的各檔案單獨 benchmark — 哪些是獨立於主程式可用的 kernel benchmark

## 跨專案對照備忘

- **llm.c 的「all-at-once memory allocation」模式** vs nanoGPT 的 `nn.Module` + PyTorch lazy allocation：線性連續 vs 物件化。值得追蹤 vLLM 或其他 C++ inference engine 是否也用類似手法。
- **llm.c 的 `zero.cuh` 多 GPU reduce 時機選擇**（只在最後一個 micro-step reduce）：跟 DeepSpeed/FSDP 的 reduce bucket 策略不同 — 後者是累積到一定量就發起 reduce。這種「延遲到最後才 reduce」在 gradient accumulation 筆數多時可能導致 NCCL 頻寬閒置。
- **llm.c 的 online softmax 實作**：跟 FlashAttention 論文中的 online softmax 是同一概念。候選 pattern「Online Softmax in CUDA」。
