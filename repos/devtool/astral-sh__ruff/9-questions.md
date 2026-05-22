---
repo: astral-sh/ruff
file: 9-questions
studied_at: 2026-05-22
commit_sha: 8c04080
---

# Ruff · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 token pass 和 AST pass 之間不做平行化？**
  - 我目前的推測: [UNVERIFIED] 雖然 check_path 內的五個 pass 是序列執行，但不同檔案是平行處理的。檔案內 pass 序列化可能為了避免 `SemanticModel` 的 race condition——AST pass 建立 semantic model，後續的 import pass 依賴於它。如果 token pass 不需要 semantic model，理論上可以跟 AST pass 平行？但程式碼中 token pass 可能也需要存取 `context`（設定了 `noqa_line_for` 等），這個共享狀態讓平行化在 file 層級比在 pass 層級更容易。
  - 相關程式碼: [`crates/ruff_linter/src/linter.rs:120-255`](https://github.com/astral-sh/ruff/blob/8c04080b5e449b077500fff1cf1d83c2a69af4c9/crates/ruff_linter/src/linter.rs#L120-L255)

- [ ] **type checker（ty* crates）的加入對 crate 結構的影響有多大？**
  - 我目前的推測: [UNVERIFIED] `ty_python_semantic` 是單一 crate 超過 13 萬行，很可能有一天會需要再拆分（類似於 `ruff_linter` 與其依賴的分層）。目前的架構中，`ruff_db` 作為共享的 symbol/file system layer，是 type checker 和 linter 之間的主要橋樑。不清楚 ty* crates 是否可以完全獨立出來作為一個 standalone type checker binary。
  - 相關程式碼: `crates/ty_python_semantic/`（131K LOC，但檔名未知）

- [ ] **Formatter 在 comment-preserving 上的具體策略是什麼？**
  - 我目前的推測: [UNVERIFIED] Ruff 的 formatter 採用類似 Black 的 approach：parse 時保留 comment positions，formatting 時根據 comment 的 AST 關聯節點決定 placement。但從程式碼來看，`ruff_formatter` 和 `ruff_python_formatter` 是兩個不同的 crate，前者是 generic formatting infra（類似 Prettier 的 `doc` builder pattern），後者是 Python 的具體 formatting 規則。這個分層是否與 linter 的架構平行？Formatter 跟 linter 共用 parser 但有不同的 IR，具體的 shared code 在哪？
  - 相關程式碼: [`crates/ruff_formatter`](https://github.com/astral-sh/ruff/blob/8c04080b5e449b077500fff1cf1d83c2a69af4c9/crates/ruff_formatter), [`crates/ruff_python_formatter`](https://github.com/astral-sh/ruff/blob/8c04080b5e449b077500fff1cf1d83c2a69af4c9/crates/ruff_python_formatter)

- [ ] **規則註冊中的 `#[derive(RuleNamespace)]` 實際生成的程式碼是什麼？**
  - 我目前的推測: [UNVERIFIED] `ruff_macros` crate 是一個 proc macro crate，提供 `RuleNamespace` derive macro。從 registry.rs 的用法來看，它應該會為 `Linter` enum 生成 `prefix()` 方法、`from_code()` 方法、以及 `all_rules()` 方法。但具體實現（例如 `#[prefix = "ANN"]` 屬性如何跟各個 rule mod 內部的 `Rule` enum variant 關聯？）需要看 proc macro 的實作，我沒時間深入讀完。
  - 相關程式碼: [`crates/ruff_macros`](https://github.com/astral-sh/ruff/blob/8c04080b5e449b077500fff1cf1d83c2a69af4c9/crates/ruff_macros)

- [ ] **`ruff_workspace` 如何處理多個 pyproject.toml 的疊加邏輯？**
  - 我目前的推測: 每個檔案的設定是「最近目錄的設定檔疊加在上層設定上」，但我不清楚疊加的計算是發生在 resolve 階段還是每個檔案的 lint 階段。如果每個檔案 lint 時都算一次，是否有效能消耗？
  - 相關程式碼: [`crates/ruff_workspace/src/resolver.rs`](https://github.com/astral-sh/ruff/blob/8c04080b5e449b077500fff1cf1d83c2a69af4c9/crates/ruff_workspace/src/resolver.rs)

## 想問維護者的問題

- Ruff 走向 type checker 的 roadmap 是什麼？完成度到哪裡了？目前 `ruff ty` 支援的 Python typing 功能 subsets 是什麼？
- 為什麼選擇在 monorepo 內放一個 `ty` crate 而不是獨立的 repo？是因為跟 linter 共享 `ruff_db` 太方便嗎？還是有其他原因？
- 自建 parser 的維護成本 vs 使用 `rustpython-parser` 的比較？這是一個公開評估過後獨立做的決定嗎？
- 對於 formatter 的可配置性，Ruff 的立場是什麼？Black 的「uncompromising」被廣泛承認但也有人抱怨。Ruff 的 `ruff format` 是走同樣路線還是未來會開放更多選項？

## 下次再看時的待辦

- [ ] 讀 `ruff_macros` crate，理解 `RuleNamespace` derive macro 如何自動生成規則代碼的 prefix 對應
- [ ] 對照 `ruff_python_parser` 的實作與 `rustpython-parser` 的不同，理解 Ruff 的 parser 設計選擇
- [ ] 了解 `ruff_db` crate 的實作——它是 linter 和 type checker 共享的底層基礎設施
- [ ] 研究 `ruff check` 的 fix 機制：`Fix` struct 的 `Applicability`（safe/unsafe）和 `IsolationLevel` 是如何影響修復行為的
- [ ] 探索 `ty_python_semantic` 的型別推論演算法——它能處理 generics、overloads、TypedDict 到什麼程度？

## 跨專案對照備忘

- **RuleNamespace + proc macro 註冊機制** → 跟 ratatui 的 `Style` widget registration pattern 相似 → 候選 pattern：**attribute-driven plugin registration**（使用 proc macro 自動將 enum variants 對應到 plugin/rule 的註冊）
- **多 pass checker 架構** → 與 compilers 的 multiple IR pass pipeline 概念類似，但在 linting context 中的實作方式值得抽象
