---
repo: promptfoo/promptfoo
file: 3-key-patterns
studied_at: 2026-05-23
commit_sha: 1205a2a
---

# promptfoo · 值得偷學的設計

## Pattern 1: Chain-of-Responsibility 的 Provider 註冊

**是什麼**: Provider 解析不用常見的 `Map<string, Factory>`，而用一個陣列儲存 `{ test: (path) => boolean, create: (path) => Provider }` pairs，依序匹配。

**為什麼有效**: 讓 provider 字串如 `openai:chat:gpt-4o` 可以經過多層次解析—先 match `openai:` prefix，內部再根據 sub-type 判斷要用 chat、completion、responses、embedding 還是其他類別。這種設計讓單一 prefix 可以承載複雜的路由邏輯，而不需要一個巨大的 switch statement。

**程式碼位置**: [`src/providers/registry.ts:125-1704`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/providers/registry.ts#L125-L1704)（providerMap 陣列）

**何時可以借用**: 當你的系統需要讓外部輸入（URL、provider path、plugin ID）經過多層次路由到不同實作時，且各層次的匹配邏輯不同。

**替代方案**: 標準 Map lookup 更快（O(1) vs O(n)），但犧牲了 chain 的靈活性。LiteLLM 用 mapping table 做法—模型名稱直接對應到 provider 和參數。

## Pattern 2: Plugin × Strategy × Grader 的 Pipeline 解耦

**是什麼**: Redteam 子系統把測試流程拆成三個完全獨立、可組合的階段：Plugin（生成測試輸入）→ Strategy（轉換成攻擊變體）→ Grader（評分攻擊是否成功）。

```
Plugin.generateTests() → TestCase[] → applyStrategies() → TestCase[] → eval.grade() → Result[]
```

**為什麼有效**: 每個階段獨立演化—Plugin 只要會生成測試輸入，不需關心攻擊手法（strategy）和評分邏輯（grader）。Strategy 可以套用到任何 plugin 產生的 test case 上。Graders 專注於評估 LLM 的回應是否「出問題」。三個階段的組合產量很大：100+ plugins × 30+ strategies × 每個 plugin 對應 grader。

**程式碼位置**: Plugin registry [`src/redteam/plugins/index.ts`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/redteam/plugins/index.ts) × Strategy registry [`src/redteam/strategies/index.ts`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/redteam/strategies/index.ts) × Grader registry [`src/redteam/graders.ts:134-313`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/redteam/graders.ts#L134-L313)

**注意事項**: 實務上並非所有 plugin × strategy 組合都有意義（編碼策略會破壞 canary token），所以需要 `excludeStrategies` 和 `CANARY_BREAKING_STRATEGY_IDS` 來管理不合法組合。

## Pattern 3: 自適應 Rate Limiting Scheduler

**是什麼**: 不是用固定的 max-concurrency，而是從 provider 的 response header 學習 rate limit 資訊，動態調整併發數。

**為什麼有效**: LLM provider 的 rate limit 很複雜—不同 tier、不同 model、不同 endpoint 都有不同限制。固定 concurrency 要不就是太保守（浪費速度）要不就是太激進（被 429）。自適應系統從 response 中學習：收到 429 → 降低 concurrency（指數退避）；成功請求 → 逐步增加 concurrency（加性增加）。

**程式碼位置**: [`src/scheduler/rateLimitRegistry.ts:22-149`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/scheduler/rateLimitRegistry.ts#L22-L149) + [`src/scheduler/adaptiveConcurrency.ts`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/scheduler/adaptiveConcurrency.ts)

**替代方案**: 固定 concurrency + 手動設定 retry delay。簡單但無法適應不同 provider 的限制差異。Ollama 等本地模型不需要 rate limiting，但遠端 API 需要。promptfoo 的設計讓同一套 eval code 可以同時處理這兩種場景。

**何時可以借用**: 任何需要呼叫外部 API 且 rate limit 不可預測的場景。

## Pattern 4: Flat-Cartesian 的 Eval Step 展開

**是什麼**: 不採用巢狀迴圈 `for test in tests { for provider in providers { for prompt in prompts { ... } } }`，而是先計算所有組合的笛卡兒積，展開成一個扁平陣列。

**為什麼有效**: 扁平化後的每個 step 互相獨立，可以任意排列執行順序 — 支援 resume（跳過已完成的 pair）、retry（只重試失敗的 step）、watch mode（只執行變更影響的 step）。扁平陣列也讓併發控制更單純：`async.forEachOfLimit(flatArray, concurrency)`。

**程式碼位置**: [`src/evaluator.ts:4292-4556`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/evaluator.ts#L4292-L4556)（_runEvaluation 中的 buildRunEvalOptions）

**注意事項**: 組合爆炸是主要風險 — 多個 prompts × providers × tests × var_combos × repeats 可能在特定情境下膨脹到幾萬個步驟。實務上透過預設的 max-concurrency 和 rate limiting 來控制。

## Pattern 5: LLM-as-Judge 的 Grader 系統

**是什麼**: Redteam 的 grader 不使用規則比對，而是把攻擊和回應送給 LLM，讓 LLM 判斷攻擊是否成功。

**程式碼位置**: [`src/redteam/plugins/base.ts:380-559`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/redteam/plugins/base.ts#L380-L559)（RedteamGraderBase）

**為什麼有效**: 安全測試的「成功」定義很模糊 — 同樣的攻擊回應，有人覺得是拒絕，有人覺得是洩漏。LLM-as-Judge 可以用自然語言描述評分標準（rubric），靈活性遠高於關鍵字比對。每個 grader 的 rubric 可以根據攻擊類型客製化評分標準。[UNVERIFIED] 但這個做法是否比傳統分類器更準確，取決於 rubric 的品質和評分 LLM 的能力。

**替代方案**: 
- 關鍵字/正則比對: 簡單但容易被繞過（模型說「我不能提供幫助」不等於拒絕）
- 專用分類器（LlamaGuard、Beavertails）: promptfoo 也支援，作為 grader 的補充而非替代
- 向量相似度: 適合判斷「是否偏離預期」，但不適合「是否安全」

**何時可以借用**: 任何需要對 LLM 輸出做 subjective assessment 的場景（安全性、品質、風格）。

## API 設計品味的觀察

- **Config 是 YAML 而非程式碼**: promptfoo 選擇宣告式 YAML config 而非 Python/TS API 作為主要介面。這讓非工程師也能撰寫測試，但犧牲了動態生成測試的能力（需要透過 `scenarios` 或 `file://` 引用外部腳本來補償）
- **Assertion 的 weight 系統**: assertion 可以用 `weight: 0` 標記為僅記錄（不影響通過率），這讓 metric-only assertion 和 blocking assertion 可以在同一個 test case 中共存
- **ProviderResponse 的 fat response 模式**: 一個 response 結構涵蓋 text, json, image, video, audio, guardrail。這讓 evaluator 只需看懂一種 response shape

## 對相容性的態度

此專案處於 0.x 階段（目前 0.121.12），不保證 API 穩定。從 CHANGELOG 觀察：
- 幾乎每天有 release（release-please 自動化），顯示快速迭代
- 少量的 deprecation notice — 許多 breaking change 直接換掉而不是慢慢淘汰
- 2025 年被 OpenAI 收購後繼續 MIT 開源，開發速度未放緩
