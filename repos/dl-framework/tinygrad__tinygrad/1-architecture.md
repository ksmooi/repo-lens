---
repo: tinygrad/tinygrad
file: 1-architecture
studied_at: 2026-05-23
commit_sha: 149a87d
---

# tinygrad · 架構

## 系統高層圖

```mermaid
flowchart TB
  subgraph "User-Facing API"
    T[Tensor API<br/>tensor.py]
    NN[nn / optim / datasets<br/>nn/]
    JIT[TinyJit<br/>engine/jit.py]
  end

  subgraph "IR Layer"
    UOP[UOp DAG<br/>uop/ops.py]
    FXN[_function decorator<br/>function.py]
    GRAD[Gradient / AD<br/>gradient.py]
  end

  subgraph "Schedule Pipeline"
    SCH[create_linear_with_vars<br/>schedule/__init__.py]
    RNG[run_rangeify<br/>schedule/indexing.py]
    MEM[memory_plan_rewrite<br/>schedule/memory.py]
    MULTI[multi-device resolve<br/>schedule/multi.py]
  end

  subgraph "Codegen Pipeline"
    CG[full_rewrite_to_sink<br/>codegen/__init__.py]
    OPT[BEAM / Heuristic Opt<br/>codegen/opt/]
    EXP[Expander<br/>codegen/late/expander.py]
    DEVEC[Devectorizer<br/>codegen/late/devectorizer.py]
    LINEARIZE[Linearizer<br/>codegen/late/linearizer.py]
  end

  subgraph "Renderer & Runtime"
    REN[Renderer<br/>renderer/]
    COMP[Compiler<br/>runtime/support/compiler_*.py]
    DEV[Device / Buffer<br/>device.py]
    OPS[ops_*.py<br/>runtime/ops_*.py]
  end

  subgraph "Hardware Backends"
    CUDA --> GPU[CUDA GPU]
    METAL --> MTL[Apple GPU]
    NV --> NVGPU[NVIDIA (QMD)]
    AMD --> AMDGPU[AMD GPU]
    CPU --> HOST[CPU]
    CL --> OCL[OpenCL]
    QCOM --> ADR[Adreno GPU]
  end

  T --> UOP
  JIT --> UOP
  UOP --> GRAD
  UOP --> FXN
  UOP --> SCH --> RNG --> MEM
  SCH --> MULTI
  RNG --> CG --> OPT --> EXP --> DEVEC --> LINEARIZE
  LINEARIZE --> REN --> COMP
  COMP --> OPS
  OPS --> CUDA & METAL & NV & AMD & CPU & CL & QCOM
  DEV --> OPS
```

```mermaid
flowchart LR
  subgraph "Lazy Evaluation Flow"
    A[User calls t.dot(b)] --> B[Creates UOp DAG<br/>no computation yet]
    B --> C{realize() called?}
    C -->|Yes| D[create_linear_with_vars]
    C -->|No| B
  end

  subgraph "Full Pipeline Trigger"
    D --> E[schedule/indexing.py<br/>run_rangeify]
    E --> F[schedule/memory.py<br/>memory_plan_rewrite]
    F --> G[codegen/__init__.py<br/>full_rewrite_to_sink]
  end

  subgraph "Codegen Passes"
    G --> H[Early movement ops<br/>→ index arithmetic]
    H --> I[Postrange optimization<br/>BEAM search or heuristic]
    I --> J[Expander<br/>UNROLL → vectorize]
    J --> K[Devectorizer<br/>vector → scalar LOAD/STORE]
    K --> L[Linearizer<br/>toposort → instruction list]
  end

  subgraph "Output"
    L --> M[Renderer<br/>UOp → source code]
    M --> N[Compiler<br/>source → binary]
    N --> O[ExecContext<br/>run_linear dispatches]
  end
```

### 圖意說明

第一張圖展示 tinygrad 的五層架構：從最頂層的 Tensor API 到底層的硬體後端。關鍵觀察是 **UOp DAG 是唯一的 IR**——排程（schedule）、編譯（codegen）、執行（runtime）全部共用同一種資料結構，差別只在於 DAG 中包含的操作類型不同（SINK/LINEAR 等 schedule ops vs. LOAD/STORE/ALU 等 codegen ops vs. PROGRAM 等 execution ops）。

第二張圖聚焦 lazy evaluation 的觸發路徑。使用者呼叫 `t.dot(b)` 時**不進行任何計算**——只建立 UOp DAG。直到 `.realize()` 被呼叫（或 JIT capture 觸發），才啟動完整的排程 → 編譯 → 執行管線。

## 跨模組通訊模式

tinygrad 所有模組間的「通訊」只有一種方式：**UOp DAG 的圖變換**。

- 排程器輸入是 `Ops.SINK` DAG，產出是 `Ops.LINEAR` DAG
- 編譯器輸入是 `Ops.LINEAR` + `Ops.STORE`/`Ops.PROGRAM`，產出是 `Ops.PROGRAM` + `Ops.SOURCE`
- 執行器輸入是 `Ops.LINEAR` + `Ops.PROGRAM`，透過 `pm_exec` 分派給各 handler

沒有 event bus、沒有 message queue、沒有 function call 鏈——全部是 `graph_rewrite` + `PatternMatcher`。

## 核心層: UOp IR

**位置**: [`tinygrad/uop/ops.py`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/uop/ops.py)

UOp（Universal Operation）是 tinygrad 唯一的中間表示。它是 frozen dataclass：

```python
@dataclass(frozen=True, eq=False)
class UOp:
  op: Ops      # 操作類型 (ADD, REDUCE, LOAD, STORE, ...)
  dtype: DType # 資料類型
  src: tuple   # DAG 的 source 節點
  arg: Any     # 操作參數
```

**Ops 分類**（[`uop/__init__.py:13`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/uop/__init__.py#L13)）:

| 類別 | 包含的 Ops | 用途 |
|------|-----------|------|
| Defines | `DEFINE_VAR, SPECIAL, DEFINE_LOCAL` | Kernel 參數宣告 |
| Load/Store | `LOAD, STORE, INDEX` | 記憶體存取 |
| ALU | `ADD, MUL, MAX, WHERE, CAST, ...` | 算術 / 邏輯運算 |
| Schedule | `SINK, LINEAR, CALL, AFTER, BARRIER` | 排程邊界 |
| Graph-only | `RESHAPE, PERMUTE, REDUCE, EXPAND, PAD, ...` | Pre-codegen 高階運算 |
| Program | `PROGRAM, SOURCE, BINARY` | 編譯後產物 |

**為什麼只用一種 IR**? 這是 tinygrad 最核心的設計決策。對比 PyTorch 的多層 IR 策略（TorchScript → ATen → Triton IR → LLVM IR），tinygrad 的選擇帶來：

- ✅ 極簡的 codebase — 不需要 IR 轉換層
- ✅ 統一的 rewrite engine — `PatternMatcher` 可在任何 stage 共用
- ✅ Debug 容易 — 整個 pipeline 的狀態是同一個資料結構
- ❌ **代價**: 跨 stage 的語意資訊無法獨立 clean up（codegen 階段的 Ops 混在 schedule 階段的 Ops 中）
- ❌ **代價**: 無法針對特定 stage 做深度優化（沒有 LLVM 的 mem2reg / SSA 等經典 pass）

這個 trade-off 是 **tinygrad 選擇 hackability 勝過 peak performance** 的具體體現。

## 排程層: Lazy Evaluation → Kernel Graph

**入口**: [`schedule/__init__.py:122`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/schedule/__init__.py#L122) `create_linear_with_vars`

Tensors lazy 構建 UOp DAG，直到 `.realize()` 才觸發排程。排程管線：

1. **`run_rangeify()`** ([`schedule/indexing.py:148`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/schedule/indexing.py#L148)) — 核心階段。反向遍歷 UOp DAG，為每個節點附加 range metadata，決定哪些中間結果需要 materialize（buffer allocation），並將 movement ops（RESHAPE/PERMUTE/SHRINK 等）轉換為 index 變換
2. **`create_schedule()`** ([`schedule/__init__.py:21`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/schedule/__init__.py#L21)) — Kahn's algorithm 拓撲排序，產出 `Ops.LINEAR`（kernel 的展平序列）
3. **`memory_plan_rewrite()`** ([`schedule/memory.py:20`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/schedule/memory.py#L20)) — TLSF（Two-Level Segregated Fit）子分配器，將 buffer 分配到共享 arena

**設計決策**: 為什麼不直接用 PyTorch 的 fuse-then-recompute？tinygrad 選擇在排程階段就決定所有 buffer 的位置（而非在 backward 時決定），這讓 backward 的記憶體模式是可預測的，但犧牲了像 PyTorch 的 gradient checkpointing 那樣的靈活性。

## 編譯層: full_rewrite_to_sink

**入口**: [`codegen/__init__.py:27`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/codegen/__init__.py#L27)

編譯器管線是一個由多個 `PatternMatcher` 串聯的 rewrite chain：

1. **Early Movement Ops** → 將 RESHAPE/PERMUTE 等轉為 index 算術
2. **Postrange Optimization** — 關鍵階段:
   - **BEAM search**（`BEAM >= 1`）: [`codegen/opt/search.py:114`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/codegen/opt/search.py#L114) — 遍歷數十種 schedule 變體，實際編譯後在 GPU 上計時，選取最快的。每一步保留 `amt` 個最佳候選
   - **Heuristic**（`NOOPT == False`）: [`codegen/opt/heuristic.py:8`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/codegen/opt/heuristic.py#L8) — 手寫規則：Tensor Core 匹配、UPCAST、LOCAL memory、GROUP_REDUCE、UNROLL
3. **Expander** → UNROLL 展開為向量化操作
4. **Devectorizer** → 向量→scalar LOAD/STORE，加入 GPU dims（threadIdx/blockIdx）
5. **Lowering** → unsupported ops 分解、index dtype 降低
6. **Linearizer** → 優先級拓撲排序，產出線性指令列表

**BEAM search 的設計取捨**:
- ✅ 對硬體拓撲（SIMT width、shared memory size、reg file）完全可適應，不需要手寫 kernel
- ✅ 比 heuristic 穩定：不需要針對每個 GPU arch 調參數
- ❌ 編譯時間長：每個候選都要編譯+執行的成本不可忽略
- ❌ 編譯時間不定：BEAM width 越大，搜索空間指數增長
- [`codegen/opt/search.py:57`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/codegen/opt/search.py#L57) 用 `_try_compile` 做 fast path（只編譯不執行）來控制成本

## 執行層: Device Abstraction

**位置**: [`tinygrad/device.py`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/device.py)

Device 是 singleton，透過掃描 `tinygrad/runtime/ops_*.py` 自動發現可用後端（[`device.py:17`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/device.py#L17)）。支援兩種模式：

| 模式 | 後端 | 特點 |
|------|------|------|
| **Direct**（`Compiled`） | CUDA, Metal, CL, HIP, WebGPU | 薄封裝廠商 API，使用原生 graph API（cuGraph, Metal ICB） |
| **HCQ**（`HCQCompiled`） | CPU, NV, AMD, QCOM | 硬體命令佇列 + timeline signal 同步，支援多介面選擇 |

**HCQ 模式的關鍵設計**（[`runtime/support/hcq.py:383`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/runtime/support/hcq.py#L383)）:
- 使用 timeline semaphore 進行非同步同步，而非 blocking sync
- 支援多介面（kernel driver / PCI MMIO / MOCK），由 `select_first_inited` 選擇
- Peer group 概念支援多裝置 graph（NV+AMD 混合）

**設計決策**: 為何要自製 NV/AMD runtime 而非使用 CUDA/HIP 原生 API？來自 `ops_nv.py` 的實作顯示 tinygrad 直接操作 NVIDIA 的 GPFIFO 硬體 ring buffer，繞過 CUDA driver 層。這讓 tinygrad 可以：
- 使用比 CUDA API 更底層的硬體控制（QMD submission）
- 避免 CUDA driver 的 overhead（context 切換、argument validation）
- 但綁定特定 GPU arch（需要追蹤 NVIDIA firmware 變更）

## JIT: TinyJit

**位置**: [`engine/jit.py:236`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/engine/jit.py#L236)

| Phase | Call次數 | 行為 |
|-------|---------|------|
| 0 (run) | cnt=0 | 正常執行，收集第一次的 kernel 序列 |
| 1 (capture) | cnt=1 | 再次執行，capture 所有 `create_linear_with_vars` 呼叫，合併為單一 `LINEAR` DAG，`jit_lower()` 執行完整的編譯管線 |
| 2+ (replay) | cnt>=2 | `run_linear(linear, var_vals, jit=True)` — 跳過編譯，直接重播已編譯的 kernel |

Graph batching（[`engine/jit.py:31`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/engine/jit.py#L31) `graph_split_rewrite`）: 將連續的 kernel call 打包成 `Ops.CUSTOM_FUNCTION("graph")`，由 `GraphRunner` 子類（CUDA/Metal/HCQ）處理 batch submission。

## 失敗模式與降級

tinygrad 的設計假定**所有 operation 都可能失敗且應有 fallback**：
- Device 初始化失敗 → 跳過該後端，嘗試下一個（[`device.py:39`](https://github.com/tinygrad/tinygrad/blob/149a87d/tinygrad/device.py#L39)）
- BEAM search 編譯失敗 → 靜默降級到 heuristic（無 crash）
- 不支援的 op → 在 codegen 的 `pm_decomp` 中分解為支援的 ops
- CUDA graph 不支援某些 kernel → `graph_split_rewrite` 自動只 batch 支援的 kernels

## 測試架構

- **Unit tests**: [`test/`](https://github.com/tinygrad/tinygrad/tree/149a87d/test) 目錄下，pytest + hypothesis
- **Ops tests**: [`test/backend/test_ops.py`](https://github.com/tinygrad/tinygrad/blob/149a87d/test/backend/test_ops.py) — 138+ kernel ops 的正確性驗證
- **Process replay**: 比較 PR 前後生成的 kernel 是否有偏差（[`test/external/process_replay/`](https://github.com/tinygrad/tinygrad/blob/149a87d/test/external/process_replay)）
- **Mock GPU**: [`test/mockgpu/`](https://github.com/tinygrad/tinygrad/tree/149a87d/test/mockgpu) — 模擬 GPU 行為進行 CI 測試
- **Fuzzing**: hypothesis-based property testing
