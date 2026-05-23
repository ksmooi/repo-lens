---
repo: tinygrad/tinygrad
file: README
type: dl-framework
studied_at: 2026-05-23
commit_sha: 149a87d
language: Python (CUDA/Metal shaders in .cu/.metal files)
framework: 自製 (zero external dependencies)
stars: ~32.7k
status: active
---

# tinygrad · 極簡深度學習框架

## 解決什麼問題

tinygrad 定位在 **PyTorch（太大、太複雜）** 跟 **micrograd（太小、不能做 real training）** 之間。它是一個端到端的深度學習棧——從 tensor 運算、autograd、JIT 編譯，到 GPU kernel 生成與執行——全部由 ~158K LOC 的 Python 完成，且**零外部相依**（`dependencies = []`）。

這讓 tinygrad 獨一無二：你可以在一個週末內讀懂完整的編譯器 pipeline 怎麼把 `a.dot(b)` 變成 GPU kernel 指令。

## 為什麼值得研究

- **完整的 DL stack 濃縮在一個 repo** — 從 tensor API 到 autograd、JIT、codegen 到 kernel launch，全都可以在 `tinygrad/` 下找到
- **UOp 作為唯一的 IR** — 不像 PyTorch 有多層 IR（TorchScript / FX / ATen），tinygrad 用一個 `UOp` DAG 貫穿排程、編譯、執行全流程。這是「IR 設計最小化」的極佳案例
- **全自製 runtime** — 支援 CUDA/Metal/AMD/NV(QMD 硬體 queue)/Qualcomm/WebGPU 共 8+ 後端，沒有依賴 cuDNN 或 MKL 等廠商函式庫
- **PatternMatcher 驅動** — 整個系統的變換（調度、優化、代碼生成）全部用同一個模式匹配引擎完成，是「rewrite rule 驅動的編譯器」的參考實作
- **George Hotz（geohot）主導** — 非常活躍的開發（每日 ~30 commit），coding style 極端追求簡潔

## 一句話架構

```
Tensor API → UOp DAG (lazy) → Schedule (kernel graph) → Codegen (full_rewrite_to_sink) → Compiled Kernel → Runtime Execution
```

## 跟競品的比較

| 面向 | tinygrad | PyTorch | JAX | micrograd | MLX |
|------|----------|---------|-----|-----------|-----|
| LOC 估計 | ~50K core | ~2M+ | ~500K+ | ~200 | ~150K |
| 外部依賴 | 無 | torch + CUDA 等 | jax + XLA | 無 | mlx (C++) |
| IR 層數 | 1（UOp） | 多層（ATen/FX/TorchScript/Dynamo） | 1（JAXPR+HLO） | N/A | MPS graph |
| Autograd | UOp AD（PatternMatcher） | Tape-based | trace-based | Tape-based | tape-based |
| JIT | TinyJit（capture + replay） | torch.compile (Dynamo) | jit (XLA) | 無 | compile |
| GPU 後端數 | 8+（自製） | 大量（cuDNN/cuBLAS 等） | XLA（TPU/GPU） | 無 | Apple Silicon only |
| Codegen | 自製 PatternMatcher → C/LLVM | Triton / NVFuser / Inductor | XLA/MLIR | 無 | Air (自製) |
| BEAM Search | ✅ 內建 | ❌ | ❌ | ❌ | ❌ |
| 主要維護者 | George Hotz / tiny corp | Meta | Google | Andrej Karpathy | Apple |
| 可讀性 / 可駭性 | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

tinygrad 的獨特優勢在於 **BEAM search** — 它會嘗試數十種 kernel schedule、實際在 GPU 上計時、選最快的，這在主流框架中不存在。代價是編譯時間較長，適合 inference 或固定 kernel 形狀的場景。

## 健康度信號

- ⭐ Stars: ~32.7k
- 📅 最後 commit: 2026-05-23（今天，非常活躍）
- 👥 主要維護者: George Hotz + tiny corp
- 🔬 v0.13.0，MIT License，每日 ~30 commits
- 💬 Discord 社群活躍（41k+ 成員）
- 🧪 CI: GitHub Actions（test.yml），pytest + hypothesis 測試

## 我會在後續筆記中回答的問題

- UOp IR 為什麼選擇單一 DAG 而非多層 IR？trade-off 是什麼？
- HCQ（Hardware Command Queue）模式跟傳統 CUDA/Metal 直接封裝的設計考量
- PatternMatcher 作為編譯器變換引擎的優缺點
- BEAM search 的實際效益與成本
- tinygrad 的 lazy evaluation 跟 JAX/PyTorch 的差異
