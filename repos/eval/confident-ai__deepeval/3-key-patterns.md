---
repo: confident-ai/deepeval
file: 3-key-patterns
studied_at: 2026-05-23
commit_sha: 17e676f
---

# DeepEval · 值得偷學的設計

## Pattern 1: `__init_subclass__` 自動追蹤

**是什麼**:

DeepEval 利用 Python 的 `__init_subclass__` hook（[`base_metric.py:37-41`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/base_metric.py#L37-L41)）在每個 `BaseMetric` subclass 被定義時自動套用 tracing decorator。

```python
class BaseMetric:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        from deepeval.tracing.internal import observe_methods
        observe_methods(cls)
```

**為什麼有效**:

- **零侵入** — metric 作者不需要知道 tracing 的存在，只需繼承 `BaseMetric`
- **保證覆蓋** — 所有 metric（包括使用者自訂的）自動獲得 tracing 能力
- **編譯期註冊** — 在 class 定義時完成，非執行期，所以沒有遺漏的風險

**程式碼位置**: [`deepeval/metrics/base_metric.py:37-41`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/base_metric.py#L37-L41)

**替代方案**:
- **Decorator-based**（如 `@log_calls`）— 需要 metric 作者記得加裝飾器，可能遺漏
- **Metaclass-based** — Python 的 `__init_subclass__` 比 metaclass 更簡單、更少 magic
- **Aspect-oriented**（如 `wrapt`）— 在 import time 對所有 `BaseMetric` 子類做 monkey-patch，破壞性更大

**何時可以借用**: 當你有一個基礎 class 且**所有** subclass 都需要某個橫切關注點（tracing / logging / metrics）時。

**不適用**: 當只有部分 subclass 需要該行為時——你可能需要 opt-in 而非 opt-out。

---

## Pattern 2: LLM-as-Judge 的 G-Eval 實作

**是什麼**:

DeepEval 實作了 G-Eval（arXiv 2303.16634）——一種 chain-of-thought 驅動的 LLM-as-a-judge 評估方法。它不是直接問「這個輸出幾分」，而是：

1. 根據 `criteria` 自動產生評估步驟（`_generate_evaluation_steps()`）
2. 建構包含 criteria、steps、rubric、test case 的結構化 prompt
3. 要求 LLM 以 JSON 格式回傳逐步驟評分與理由
4. 透過 `trimAndLoadJson()` 解析回應，計算加權總分

```python
# [g_eval.py:42-79] GEval 初始化要求 criteria + evaluation_params
# [g_eval.py:127-134] 實際評估呼叫 _evaluate()
g_score, reason = self._evaluate(test_case, ...)
self.score = (float(g_score) - self.score_range[0]) / self.score_range_span
self.success = self.score >= self.threshold
```

**為什麼有效**:

- **Chain-of-thought 評分**比直接給分更一致、更可解釋
- **Rubric 設計**支援多個評分等級與對應描述，可自訂
- **Score normalization**將任意分數範圍映射到 0-1，讓 threshold 的語意一致
- **Evaluate steps**可靜態提供或動態產生，適應不同評估任務

**程式碼位置**: [`deepeval/metrics/g_eval/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/g_eval/)

**替代方案**:
- **單一 prompt 直接給分**（如 OpenAI Evals 的做法）— 更快但品質較差
- **Fine-tuned classifier**（如 CHILL 等）— 更準但需要訓練資料與維護
- **Human evaluation**— gold standard 但不可擴展

**何時可以借用**: 任何需要 LLM 作為評估者的場景，尤其是評估標準主觀或需要理由的任務。

**不適用**: 當你需要毫秒級延遲（G-Eval 一次呼叫需 1-3 秒）或極低成本時。

---

## Pattern 3: ErrorConfig 驅動的錯誤處理策略模式

**是什麼**:

DeepEval 將評估過程中的錯誤處理策略封裝為 `ErrorConfig`（[`evaluate/configs.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/configs.py)）物件，而非用 boolean flag 或環境變數。

```python
class ErrorConfig:
    ignore_errors: bool = False
    skip_on_missing_params: bool = False
```

在執行引擎中（[`_common.py:243-295`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/execute/_common.py#L243-L295)），錯誤處理的邏輯是：

```
MissingTestCaseParamsError → skip_on_missing_params? → skip / error
TypeError (signature mismatch) → fallback to old signature
Other Exception → ignore_errors? → swallow / raise
```

**為什麼有效**:

- **關注點分離** — metric 執行引擎不決定錯誤處理策略，呼叫端決定
- **組合性** — `AsyncConfig`、`DisplayConfig`、`CacheConfig`、`ErrorConfig` 各自獨立，可按需組合
- **單一職責** — 每個 config 只管理一個維度，易於理解與擴充

**程式碼位置**: [`deepeval/evaluate/configs.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/configs.py) + [`deepeval/evaluate/execute/_common.py:243-295`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/execute/_common.py#L243-L295)

**替代方案**:
- **Global flags**（如環境變數）— 無法在不同呼叫間有不同的行為
- **Exception hierarchy-based**（不同例外用不同 catch）— 適用於策略簡單時，但無法由呼叫端控制
- **Callback-based**（傳入 `on_error` callback）— 更靈活但也更複雜

**何時可以借用**: 任何需要讓呼叫端控制錯誤處理行為的 library，尤其是批次處理大量獨立任務的場景。

**不適用**: 單一任務、單一錯誤處理策略的簡單 library。

---

## Pattern 4: pytest 整合 — 不只是 plugin，而是斷言語意

**是什麼**:

DeepEval 不只是在 `pyproject.toml` 註冊 pytest plugin。它的核心 API 設計從語意上就對齊 pytest 的 assertion 哲學：評估就是「斷言正確」。

```python
# pyproject.toml:15 — pytest11 entry point
[tool.poetry.plugins."pytest11"]
deepeval = "deepeval.plugins.plugin"
```

但更重要的是 API 設計層面：

- `assert_test()` 回傳 `None`（成功時）或 `raise AssertionError`（失敗時）
- 失敗訊息包含所有 metric 的名稱、分數、threshold、理由——直接餵給 pytest 的 assertion error
- pytest plugin 同時提供 trace-scoped assert_test（[`evaluate/execute/trace_scope.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/execute/trace_scope.py)），讓 plugin 自動捕捉 trace context 作為 test case

**為什麼有效**:

- **零認知負擔** — 任何會寫 pytest 的開發者都知道如何用 `assert_test`
- **CI/CD 原生支援** — 無需特殊 runner，`pytest test_chatbot.py` 就能跑
- **Report 自動集成** — pytest 的 `--junitxml`、`--cov` 等全數相容

**程式碼位置**: [`pyproject.toml:15`](https://github.com/confident-ai/deepeval/blob/17e676f/pyproject.toml#L15) + [`deepeval/plugins/plugin.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/plugins/plugin.py)

**替代方案**:
- **獨立 CLI runner**（如 lm-evaluation-harness）— 彈性但無法嵌入現有測試流程
- **Custom test framework**（如 Inspect AI 的自主 runner）— 功能強大但學習成本高
- **Unittest integration**— pytest 生態遠大於 unittest

**何時可以借用**: 當你的工具需要開發者日常執行時，嵌入他們已有的工具鏈永遠比創造新工具鏈好。

**不適用**: 當評估邏輯複雜到需要自訂生命週期管理（如長時間 running agent evaluation）時。

---

## Pattern 5: Lazy Model Initialization via `initialize_model()`

**是什麼**:

DeepEval 的 metric 在建構時不會立即建立 model client。model 的初始化延遲到第一次 `measure()` 呼叫時，透過 `initialize_model()`（[`metrics/utils.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/utils.py)）完成。

```python
# GEval.__init__: 僅儲存 model 的 specification
self.model, self.using_native_model = initialize_model(model)
# 真正的 LLM client 建立發生在第一次 measure() 的內部呼叫
```

**為什麼有效**:

- **快速 import** — `from deepeval.metrics import GEval` 不需要等待 model client 初始化
- **避免不必要的 API 呼叫** — 如果 metric 從未被 `measure()` 呼叫（例如在 test discovery 階段），就不會建立 client
- **錯誤延遲** — API key 缺失等錯誤在執行時才暴露，而非 import time

**程式碼位置**: [`deepeval/metrics/utils.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/utils.py)（`initialize_model` 函式）

**替代方案**:
- **Eager initialization** — 建構時立即建立 client，簡單但浪費
- **Singleton model pool** — 所有 metric 共享 model instance，節省連線但增加耦合
- **Factory pattern** — 由外部注入 model instance，最靈活但呼叫端最複雜

**何時可以借用**: 當你的 library 使用昂貴的遠端資源（LLM API、資料庫連線）且無法保證使用者一定會用到它時。

**不適用**: 輕量級本地資源（如 regex matcher、計算函式）不需要延遲初始化。

---

## Pattern 6: Config 物件組合模式

**是什麼**:

DeepEval 將評估執行的所有可配置參數拆分為四個獨立的 config dataclasses：

- `AsyncConfig` — 併發控制（`max_concurrent`, `throttle_value`）
- `DisplayConfig` — 顯示行為（`verbose_mode`, `show_indicator`）
- `CacheConfig` — 快取策略（`use_cache`, `write_cache`）
- `ErrorConfig` — 錯誤處理（`ignore_errors`, `skip_on_missing_params`）

它們不是巢狀結構，而是平級參數物件，各自有預設值。

**為什麼有效**:

- **預設有用** — 每個 config 都有合理的預設值，新手可以完全忽略
- **選擇性覆蓋** — 進階使用者只覆蓋需要的 config
- **單一職責** — 每個 config 易於測試和修改
- **組合而非繼承** — 比繼承一個大 Config 類別更具彈性

**程式碼位置**: [`deepeval/evaluate/configs.py`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/configs.py)

**替代方案**:
- **單一大型 Settings 物件** — 簡單但職責混雜
- **Keyword arguments 散列** — 不設 config 物件直接用 keyword arg，但函式簽名會爆炸
- **Environment variable 為主** — 執行期靈活性差

**何時可以借用**: 當你的函式有 10+ 個可配置參數，且可自然分為幾組關注點時。

---

## Pattern 7: 框架整合的 Callback / Wrapper 雙模式

**是什麼**:

DeepEval 對不同 LLM framework 採用兩種不同的整合策略：

- **Callback 模式**（LangChain、LlamaIndex）— 實作 framework 的回呼介面，被動接收事件
- **Wrapper 模式**（OpenAI、Anthropic、OpenAI Agents）— 包裝原始 client，主動攔截呼叫

```python
# LangChain: callback handler
# deepeval/integrations/langchain/

# OpenAI: wrapper client
# deepeval/openai/
```

**為什麼有效**:

- **尊重 framework 生態** — callback-based framework 用 callback，client-based 用 wrapper
- **非侵入** — 兩種都不需要修改使用者現有的應用程式碼
- **自動 trace** — wrapper 模式下，所有 LLM 呼叫自動進入 DeepEval 的 tracing 系統

**程式碼位置**:
- LangChain callback: [`deepeval/integrations/langchain/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/integrations/langchain/)
- OpenAI wrapper: [`deepeval/openai/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/openai/)
- Pydantic AI integration: [`deepeval/integrations/pydantic_ai/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/integrations/pydantic_ai/)

**替代方案**:
- **Middleware 模式**（如 OpenTelemetry）— 更標準但需要框架支援
- **Monkey-patching** — 最簡單也最脆弱
- **Proxy pattern** — 所有請求經過一個代理服務，部署複雜

**何時可以借用**: 當你需要在多個不同的第三方系統中插入統一的監控/評估層時。

**不適用**: 當所有整合都使用同一種接入機制時（單一策略即可）。

## API 設計品味的觀察

DeepEval 的 API 有幾個值得注意的品味：

1. **`assert_test()` 而非 `run_eval()`** — 選擇「斷言」的語意而非「執行」的語意，讓評估融入 pytest 的 assertion 哲學。這是一個語言設計上的選擇：你是在「測試 output 是否正確」而非「跑一個評估任務」。

2. **Metric 的 threshold 預設 0.5** — 這是一個刻意保守的預設值。既不寬鬆到無感（如 0.1），也不嚴格到寸步難行（如 0.9）。使用者應該根據自己的應用調整。

3. **`SingleTurnParams` 而非字串列舉** — 用 Enum 而非字串來指定評估參數，獲得 type safety 與 IDE autocomplete。同時提供 `LLMTestCaseParams` 作為向下相容的別名（[`test_case/__init__.py:45`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/test_case/__init__.py#L45)）。

4. **Rich output** — 使用 `rich` 而非 `logging` 來呈現評估結果，讓 CLI 體驗接近 LLM 應用的「dashboard」而非傳統程式的「log」。

## 對相容性的態度

- **嚴格 SemVer** — 從 v1 到 v4 都有明確的 breaking change 標記
- **Deprecation 流程** — 使用 Python 的 `DeprecationWarning`，提供至少一個 minor version 的過渡期（如 [`test_case/__init__.py:45`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/test_case/__init__.py#L45) 的 `LLMTestCaseParams` → `SingleTurnParams`）
- **自動 changelog** — 透過 GitHub Actions + 自訂 script 維護
- **長期穩定** — 從 2023 年至今持續開發，v4 已經是非常成熟的產品
