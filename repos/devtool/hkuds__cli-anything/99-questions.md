---
repo: HKUDS/CLI-Anything
file: 99-questions
studied_at: 2026-05-21
commit_sha: 436a4f5
---

# CLI-Anything · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼不把 HARNESS.md 寫成可程式執行的 pipeline？**
  - 目前 HARNESS.md 的 7 個 phase 是自由文字，沒有強制 agent 必須依序執行。如果一個 phase 被跳過或做錯，沒有 runtime 檢查
  - 我的推測：寫成可執行的 pipeline 需要一套 DSL 或 script，這會大幅增加專案複雜度。目前 method-driven 的方式在「彈性」和「可控性」之間選擇了前者。[UNVERIFIED]
  - 相關程式碼：[`cli-anything-plugin/HARNESS.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/HARNESS.md#L1-L8)

- [ ] **PEP 420 namespace package 的設計在什麼 edge case 下會失效？**
  - 如果兩個不同的 harness 定義了相同名稱的子模組（例如 `cli_anything/shared/utils.py`），行為是 undefine 的
  - 目前的 harness 都是 `cli_anything/<software>/...`，軟體名作為 namespace 分隔，不太可能衝突
  - 但如果有套件試圖貢獻到 `cli_anything/core/` 這樣的共享 namespace，就會有問題
  - 相關程式碼：[`cli-anything-plugin/commands/cli-anything.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/commands/cli-anything.md#L80-L85)

- [ ] **repl_skin.py 的複製貼上模式如何長期維護？**
  - 每個 harness 從 plugin 複製一份 repl_skin.py。如果 repl_skin.py 有 bug 或需要更新，所有已存在的 harness 必須手動同步
  - 目前約 50 個 harness 都 run 過一遍 `pip install -e .`——更新 repl_skin.py 是否需要全部 rebuild？
  - 我的推測：因為每個 harness 是獨立 pip 套件，沒有中央更新機制。但如果 repl_skin.py 的 API 穩定不變，這可能不是實際問題。[UNVERIFIED]
  - 相關程式碼：[`cli-anything-plugin/repl_skin.py`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/repl_skin.py)

- [ ] **LLM 的 context window 限制了 HARNESS.md 的複雜度天花板？**
  - 目前 HARNESS.md 約 747 行（37KB），對主流 LLM 的 context window 來說還算輕量
  - 但如果 HARNESS.md 持續擴增（更多的 phase、更多的規則、更多的 edge case 處理），總有一天會超過 agent 的有效 context
  - CLI-Anything 的解法是把深入指南移到 `guides/` 目錄，HARNESS.md 只保留核心 phase。這是個合理的設計
  - 相關程式碼：[`cli-anything-plugin/HARNESS.md`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/HARNESS.md) 對照 [`cli-anything-plugin/guides/`](https://github.com/HKUDS/CLI-Anything/blob/436a4f5c42452b86b64fe0373e1ed67a4347a18a/cli-anything-plugin/guides/)

## 想問維護者的問題

- CLI-Anything 的成長曲線非常陡峭（2 個月 38k stars、50+ harnesses）。這個爆發性的增長是預期的，還是 organic 的？
- 怎麼確保社群貢獻的 harness 品質？除了 CI 測試，有 code review 流程嗎？
- 有沒有計畫讓 CLI-Anything 支援「非 Python」的後端語言？例如直接產生 Rust 或 Go 的 CLI？
- Preview 機制（`preview_bundle.py`）的設計理由是什麼？為什麼不直接用 --output 參數？

## 下次再看時的待辦

- [ ] 深入閱讀 `docs/PREVIEW_PROTOCOL.md` 和 `preview_bundle.py`，理解 preview 架構的完整設計
- [ ] 對照 `agent-state-machine` pattern 分析 CLI-Anything 的 Session state 管理設計
- [ ] 實際跑一次 `/cli-anything ./blender` 的完整流程，記錄 LLM token 消耗和時間

## 跨專案對照備忘

- **Method-Driven Code Generation pattern**: CLI-Anything 的 HARNESS.md approach 跟其他 agent-driven code generation 工具類似。如果這個 pattern 在其他 2+ repo 出現，可以抽到 _patterns/
- **PEP 420 Namespace-as-Plugin-System pattern**: 用 namespace packages 做 plugin 生態系的做法值得追蹤。這跟 `pluggy`/`entry_points` 為主的 plugin 系統是不同的設計哲學
- **Output Verification pattern**: 「不信任 exit code，強制驗證輸出內容」——這在 AI agent 產生的程式碼中特別重要，若在更多 repo 看到可寫成 pattern
