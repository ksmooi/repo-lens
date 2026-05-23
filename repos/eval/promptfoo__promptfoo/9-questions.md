---
repo: promptfoo/promptfoo
file: 9-questions
studied_at: 2026-05-23
commit_sha: 1205a2a
---

# promptfoo · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 evaluator 是 4700 行的單一檔案？**
  這是歷史累積還是刻意選擇？從架構文件看到有 `layers.json` 定義了模組邊界，但 evaluator 本身沒有拆開。我猜測是因為 eval 流程中 state 共享太多（progress bar、token usage accumulator、rateLimitRegistry 等），拆開後反而需要大量 context passing。[UNVERIFIED]

- [ ] **Remote plugin generation 的具體伺服器端邏輯是什麼？**
  50+ plugins 只能透過 remote API 執行（`createRemotePlugin`），這些 plugin 在 promptfoo 伺服器端是怎麼實現的？是同樣的模板 + LLM 生成，還是有專用模型？原始碼中只看到 `getRemoteGenerationUrl()` 和 `shouldGenerateRemote()` 的控制邏輯，沒有伺服器端實作。

- [ ] **Assertion type 的 dispatch 為什麼不用 decorator/annotation？**
  47 種 assertion type 的 handler 註冊用一個靜態的 `Record<string, Function>`（[`src/assertions/index.ts:224-313`](https://github.com/promptfoo/promptfoo/blob/1205a2a2be77beb8505731515d0af1ee893cacb0/src/assertions/index.ts#L224-L313)），每次新增 type 必須修改這個 map。為什麼不用裝飾器或自動註冊？推測是為了保持明確的可讀性，避免 magic。[UNVERIFIED]

- [ ] **Redteam 的 LLM-as-Judge rubric 品質如何保證？**
  每個 grader 的 rubric 是手寫的 Nunjucks 模板。100+ graders 意味著 100+ 份手寫評分標準。這些 rubric 有沒有測試覆蓋？有沒有自動化評估 rubric 本身品質的機制？從原始碼中沒有看到 rubric 的測試或驗證。

- [ ] **被 OpenAI 收購後，專案走向會變嗎？**
  2025 年 promptfoo 被 OpenAI 收購時公開承諾保持 MIT 開源。從 2026-05 的開發活動來看（每天 release），發展仍活躍。但長期來看：
  - 是否會優先支援 OpenAI 的模型和功能？
  - Remote redteam generation 是否最終只對 OpenAI 用戶開放？
  目前沒有任何證據顯示偏離，但這類收購的長期影響通常需要 1-2 年才會顯現。[UNVERIFIED]

## 想問維護者的問題

- 為什麼選擇 prefix-based chain-of-responsibility 而非 decorator/annotation-based 的 provider 註冊？
- Evaluator 的 4700 行單檔是刻意保留還是重構尚未完成？
- 你認為 promptfoo 跟 DeepEval 的核心差異是什麼？是 YAML vs Python 的取捨，還是更深層的設計哲學？

## 下次再看時的待辦

- [ ] 探索 scheduler 的 AdaptiveConcurrency 實作細節 — 具體怎麼從 header 學習 rate limit
- [ ] 對照 DeepEval 的 metric 系統 — 看 assert-driven 和 metric-driven 兩種 eval 哲學的差異
- [ ] 瀏覽 `src/server/` 的 API routes — 了解 view server 的 REST API 設計
- [ ] 看 `src/redteam/commands/report.ts` — 了解 redteam 報告的生成方式

## 跨專案對照備忘

- **Provider chain-of-responsibility 設計**跟 Hermes Agent 的 provider 系統（`providers/` 目錄、CLI 切換模型）屬於同一類問題的不同解法。Hermes 用 CLI 參數 + env 切換，promptfoo 用 prefix 字串解析。兩者都在解決「讓使用者用簡單字串指定 LLM provider」的問題 → **候選 pattern: LLM Provider 的 Registry 設計**
- **Plugin × Strategy × Grader 三層 pipeline** 跟 LangChain 的 Runnable 介面、AutoGen 的 Agent 註冊機制有相似的解耦意圖。不過 promptfoo 的 pipeline 是離線（安全測試生成 → 執行 → 評分），而非執行期的請求路由。這個觀察尚未滿足 pattern 收錄門檻（目前只在此 repo 看到完整的 Pipeline 架構）。
