---
repo: confident-ai/deepeval
file: 9-questions
studied_at: 2026-05-23
commit_sha: 17e676f
---

# DeepEval · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 `BaseMetric` 和 `BaseConversationalMetric` 是兩個完全分離的 class 而非一個泛型 class？**
  - 兩者有大量重複的屬性與方法（`threshold`, `score`, `async_mode`, `measure()` 等）。
  - 我目前的推測：[UNVERIFIED] 因為 `ConversationalTestCase` 的評估邏輯與 `LLMTestCase` 有本質差異（需要逐 turn 評估），而 Python 的型別系統讓泛型化增加不少複雜度。但也可能是歷史原因——conversational metric 是後來才加入的。
  - 相關程式碼: [`deepeval/metrics/base_metric.py:17-161`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/base_metric.py#L17-L161)

- [ ] **`evaluate()` vs `assert_test()` 的行為重疊有多大？**
  - 兩個函式都接受 test cases + metrics 並執行評估，但 `assert_test()` 拋出 `AssertionError`，`evaluate()` 回傳 `EvaluationResult`。除了 assertion 行為外，底層執行引擎是同一套（`a_execute_test_cases()` vs `execute_test_cases()`）。
  - 我目前的推測：[UNVERIFIED] `assert_test()` 是 pytest 專用的薄 wrapper，`evaluate()` 是給 SDK 用戶的 general API。但從程式碼看來，`evaluate()` 多了一些平台同步邏輯。
  - 相關程式碼: [`deepeval/evaluate/evaluate.py:66-154`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/evaluate.py#L66-L154) vs [`deepeval/evaluate/evaluate.py:157-333`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/evaluate/evaluate.py#L157-L333)

- [ ] **為什麼 GEval 的 sync `measure()` 在 `async_mode=True` 時自己啟動 event loop 跑 async，而不是由執行引擎統一處理？**
  - 這導致執行引擎無法控制 metric 層的併發——每個 metric 自己決定要不要開 event loop。
  - 我目前的推測：[UNVERIFIED] 歷史原因——sync `measure()` 是先存在的，async 是後來加的。改由引擎統一排程需要大重構。
  - 相關程式碼: [`deepeval/metrics/g_eval/g_eval.py:104-125`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/g_eval/g_eval.py#L104-L125)

- [ ] **DAG metric 的實際使用率如何？**
  - DAG 是 DeepEval 特別自豪的功能——用圖形組合多個評估條件。但引入的複雜度不低（序列化、圖遍歷、節點定義）。
  - 相關程式碼: [`deepeval/metrics/dag/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/metrics/dag/)

- [ ] **`deepeval/models/` 目錄下的 `DeepEvalBaseLLM` 抽象層的使用範圍？**
  - 看起來不是所有 metric 都透過這個抽象層來呼叫 LLM——部分 metric 直接使用 `openai` / `anthropic` SDK。
  - 我目前的推測：[UNVERIFIED] 各 metric 在早期是自己接 LLM，後來才逐步統一抽象，但尚未全面遷移。

- [ ] **為什麼 `deepeval/openai/` 和 `deepeval/integrations/langchain/` 的整合模式不同？**
  - OpenAI 是 wrapper（包裝 `openai` client），LangChain 是 callback handler。為什麼不統一？
  - 相關程式碼: [`deepeval/openai/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/openai/) vs [`deepeval/integrations/langchain/`](https://github.com/confident-ai/deepeval/blob/17e676f/deepeval/integrations/langchain/)

## 想問維護者的問題

- DeepEval 的演進路線是「從 metric 擴充到完整評估平台」還是「從平台收斂到核心評估引擎」？v4.0 加入了更多 tracing 和 agentic 能力，感覺是在擴張。
- 「Confident AI」平台和開源 DeepEval 的邊界在哪？哪些功能不會開源？
- MCP metric 和 MCP 協定本身的關係如何——是評估 MCP-based agent 還是評估 MCP server 本身的品質？

## 下次再看時的待辦

- [ ] 探索 `deepeval/optimizer/` 的 prompt 優化演算法（CoPro、GEPA、SIMBA、MiPro v2）— 這些演算法是如何根據評估結果反饋修改 prompt 的？
- [ ] 對照 lm-evaluation-harness 的 task 抽象與 DeepEval 的 metric 抽象——不同的評估方式導致怎樣的系統設計差異？
- [ ] 研究 `deepeval/tracing/` 的 OpenTelemetry 整合，特別是 span 的生命週期如何與評估流程對齊
- [ ] 了解 `deepeval/synthesizer/` 的 golden data 生成邏輯——用 LLM 產生評估資料的品質控制機制

## 跨專案對照備忘

- **Pattern: ErrorConfig 策略模式** — DeepEval 的 `ErrorConfig`（四個獨立的 config dataclasses 組合）在 repo_lens 的 `agentic` 類別中觀察相似的執行期策略注入模式（如 microsoft/autogen 的 component 系統）。已觀察到 2 個 repo，尚不滿足 3 個的跨 repo pattern 門檻。
- **Pattern: `__init_subclass__` 自動橫切關注點** — DeepEval 用 `__init_subclass__` 自動加 tracing，ruff（astral-sh/ruff）用類似的機制註冊規則。這可能是 Python 開發工具的常見模式——候選 pattern 暫名「subclass-hook-auto-registration」。
