## 學習對象

- Repo: [astral-sh/ruff](https://github.com/astral-sh/ruff)
- Commit: `8c04080` (2026-05-21)
- 語言 / 主要技術: Rust
- 專案類型 / 類別: `devtool`（套用模板: `library.md`）
- 學習深度: standard
- 筆記路徑: `repos/devtool/astral-sh__ruff/`

## 關鍵入口檔案

| 檔案 | 選擇理由 | 深度精讀指標 |
|---|---|---|
| `crates/ruff/src/lib.rs` | CLI 調度中樞，`run()` 函數分派所有指令 | 674 行，含 `check()`、`format()`、`server()` 跳板函數 |
| `crates/ruff_linter/src/linter.rs` | `check_path()` — 五 pass checker 的入口與調度 | 1256 行，定義 token → AST → import → physical lines 管線 |
| `crates/ruff_linter/src/checkers/ast/mod.rs` | AST visitor 核心，四階段（binding/traversal/cleanup/analysis） | 3758 行，SemanticModel 的狀態管理核心 |
| `crates/ruff/src/commands/check.rs` | `ruff check` 指令的完整 orchestrator（檔案發現→平行→彙整） | 302 行，rayon 平行化 + cache 載入邏輯 |
| `crates/ruff_linter/src/registry.rs` | 規則命名空間系統（Linter enum + RuleNamespace） | 522 行，proc macro 驅動的前綴規則註冊 |

## 三個最重要的發現

1. **五 pass checker 架構** — Ruff 的 linter 不是單一巨大 AST visitor，而是分為 token → filesystem → logical line → AST → import → physical line 六個獨立 pass，每個 pass 只處理其所需資訊的規則。這讓每個 pass 的程式碼保持專注，且未啟用的 pass 在 runtime 被零成本跳過。
2. **從 linter 走向 type checker** — Ruff 已經開始內建完整的 Python 型別推論引擎（`ty_python_semantic` crate 約 13 萬行），是 Ruff 中最大的單一 crate。這個方向顯示 Ruff 的最終目標是取代 mypy + Pyright，而不只是 Flake8 + Black。
3. **RuleNamespace + proc macro 的規則註冊系統** — 透過自訂 derive macro 讓 80+ 個規則模組無需手動註冊。新增一條規則只需在對應模組加一個 enum variant，prefix 和 code 的對應由 `ruff_macros` 自動生成。

## 候選 Pattern

- **attribute-driven plugin registration**: `#[prefix = "ANN"]` 屬性 + `RuleNamespace` derive macro 的組合，讓規則註冊完全自動化。已在 Ruff 和 biome 中觀察到類似的 proc macro 使用模式，值得追蹤是否成為 Rust CLI 工具的 common pattern。

## 品質檢查摘要

- 各檔字數（bytes）: README=2,963 / 1-arch=15,127 / 2-walk=10,703 / 3-pat=11,345 / 9-q=5,380
- Mermaid 圖數量: 1-arch=2 張、2-walk=1 張、3-pat=1 張 = 總計 4 張
- path:line 引用總數: 36 個
- 量化資訊出現次數（版本、規模、效能等具體數字）: 15+ 處（版本號、LOC、stars、速度倍數等）
- 比較表格出現次數: README 中 1 個（Ruff vs Flake8 vs mypy vs Black）

## 執行決策日誌

- REPO_SLUG 決定與衝突檢查結果: `astral-sh__ruff` — 無衝突，repos/devtool/ 下無既有筆記
- 類別確認: CATEGORY=`devtool` / TEMPLATE=`library.md`
- 類別決策說明: Ruff 是 Python linter + formatter，屬於 CLI 開發工具，歸類 devtool。對應 `library.md` 模板（與 `devtool` 類別匹配）
- 新增類別時的附帶動作: 無，使用既有類別
- 關鍵入口檔案選擇與排除理由: 選擇了 lib.rs（CLI 調度）、linter.rs（核心 linter）、checkers/ast/mod.rs（AST visitor）、commands/check.rs（指令 orchestrator）、registry.rs（規則註冊）。排除了 formatter crate（與 linter 共享 parser 但獨立，架構上相對直觀）、ty* crates（規模太大，standard depth 無法完整覆蓋）、ruff_server（LSP protocol 實作較獨立）
- 競品識別: Flake8 + plugins（傳統 lint）、mypy（type checking）、Black（formatting）
- subagent 使用情況: 未啟用（LEARNING_DEPTH=standard，直接分析）
- 工具替換: pygount timeout（超過 60s），改用 `find + wc -l` 替代
- 模板章節未明確規則的處理: 將 `library.md` 的 `## API 風格一句話` 調整為 CLI-first 描述（devtool 特性）
- 任務 prompt 覆寫 AGENTS.md 的項目: 無衝突
- [UNVERIFIED] 標註的部分清單:
  1. 1-architecture.md — 自建 parser 的切換時間點
  2. 9-questions.md — token/AST pass 平行化可能性
  3. 9-questions.md — ty* crates 獨立性
  4. 9-questions.md — formatter comment-preserving 策略
  5. 9-questions.md — RuleNamespace proc macro 實作細節

## Git 身份驗證

- Commit author: ksmooi <kaishianmooi@gmail.com>
- Git config 確認指令輸出:
  ```
  ksmooi
  kaishianmooi@gmail.com
  ```

## 驗證版本說明

此 PR 為 Hermes Agent repo 學習功能，學習對象: astral-sh/ruff。
