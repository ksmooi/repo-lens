---
repo: tinygrad/tinygrad
file: 9-questions
studied_at: 2026-05-23
commit_sha: 149a87d
---

# tinygrad · 未解問題

## 還沒搞懂的設計決策

- [ ] **為何不考慮用 MLIR 作為 IR？**
  - 我目前的推測：[UNVERIFIED] George Hotz 在訪談中提過 MLIR 的 C++ 依賴太重，不符合 tinygrad「零外部相依」的哲學。但 MLIR 的 dialect 機制（linalg → scf → llvm）其實跟 tinygrad 的 UOp rewrite 精神類似，差別在於實作語言（C++ vs Python）和生態系整合。
  - 相關程式碼: 整個 `tinygrad/uop/` 目錄是這個決策的結果

- [ ] **UOp 的 frozen dataclass + 弱引用 cache（ucache）在多大規模下會失效？**
  - 我目前的推測：[UNVERIFIED] `ucache` 使用 `weakref.WeakValueDictionary`（[`uop/ops.py:89`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/uop/ops.py#L89)），當 UOp DAG 成長到數百萬節點時，cache lookup 和 weak reference GC 壓力可能成為瓶頸。tinygrad 的 DAG 通常在數千到數萬節點，還未到這個規模。
  - 相關程式碼: [`uop/ops.py:85-105`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/uop/ops.py#L85)

- [ ] **`_function` decorator 的 implicit buffer detection 機制如何保證正確性？**
  - 我目前的推測：[UNVERIFIED] 透過 PatternMatcher（`pm_ctx` 在 [`function.py:19-23`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/function.py#L19)）找出所有非 PARAM 的 BUFFER/BIND/AFTER/CONTIGUOUS UOp 作為 implicit inputs。這依賴於 UOp DAG 的拓撲完整性 — 若 DAG 不完整（例如某個 buffer 被 GC 掉了），implicit detection 可能 miss。但因為 `all_tensors` 維持了所有 scope 內的 Tensor 參考，在 practice 中沒遇過問題。
  - 相關程式碼: [`function.py:66-74`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/function.py#L66)

## 想驗證的架構 claim

- [ ] **BEAM search 的實際加速效果**：README 宣稱 matmul 可以被 fuse 成單一 kernel。想知道 BEAM=1 vs heuristic-only 在常見操作（matmul、conv2d、attention）上的 real-world benchmark 數據。目前只看到 `test/speed/` 有部分 benchmark。
  - 相關檔案: [`test/speed/`](https://github.com/tinygrad/tinygrad/tree/149a87d/test/speed)

- [ ] **HCQ 模式的 overhead**：用 timeline signal 取代 blocking sync 會增加多少 latency？對 single kernel 的場景（non-batched）可能是 net negative 嗎？
  - 相關程式碼: [`runtime/support/hcq.py:383`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/runtime/support/hcq.py#L383)

## 想問維護者的問題

- [ ] UOp 原本是設計給"Universal Operations"還是"Universal Operations"之外的什麼縮寫？（我在源碼中沒看到全稱）
- [ ] 在 HCQ 模式中選擇 timeline signal 而非 CUDA events / Metal shared events，背後的設計取捨是什麼？
- [ ] 考慮過支持 Apple MLX 嗎？或者 MLX 對 tinygrad 的 Metal backend 是否有影響？
- [ ] 對 dynamic shapes 的計畫？目前 JIT 在 shape 改變時需要 recapture，這是已知限制嗎？

## 下次再看時的待辦

- [ ] 實際執行一次 `BEAM=2 python3 -c "from tinygrad import Tensor; ..."`，觀察 BEAM search 的行為與輸出
- [ ] 深入研究 `schedule/indexing.py` 的 `run_rangeify`，特別是 W/AW (Write-After-Read) dependency resolution 的實作細節
- [ ] 對照 MLX 的 JIT compile 機制（MJIT），看跟 tinygrad 的 TinyJit 有何異同
- [ ] 讀懂 `runtime/support/nv/nvdev.py` 的 GSP firmware 互動 — 這是目前最深的硬體抽象層
- [ ] 用 `sys.setprofile` 或 `py-spy` 分析一次 `.realize()` 呼叫的 time breakdown（排程 vs. 編譯 vs. 執行）

## 跨專案對照備忘

- **UOp 作為唯一 IR → RL 或 DSL 中的 AST**: tinygrad「單一 IR + PatternMatcher」的做法跟 LLVM 的「single IR + pass manager」理念相同，但實作在 Python 上且 level 高很多。不過 LLVM 的 IR 更注重 SSA 形式和型別系統，UOp 則以 pragmatism 為主。值得追蹤是否會在 **其他 DL compiler**（如 Triton 的內部 IR）看到類似的簡化策略。→ 候選 pattern：「單一 IR 的 compiler 設計」。

- **HCQ timeline signal → 分散式系統的 barrier 設計**: HCQ 用 timeline signal 進行 GPU queue 間同步的做法，本質上跟分散式系統的 Lamport clock / vector clock 同步是同一個 pattern。如果用更統一的語言描述，可以抽象成「用 monotonic counter 進行非阻塞事件同步」。→ 候選 pattern：「Timeline-based async synchronization」。

- **BEAM search inside compiler**: 直接在編譯器內建 autotuning，而不是像 TVM 那樣分離成 AutoTVM（外部工具）。這簡化了 workflow 但增加了編譯時間。我目前只在 tinygrad 看到這個做法 — 如果在其他 compiler（如 Halide、Triton、Ansor）也觀察到，值得昇級為 `_patterns/` 的正式 pattern。
