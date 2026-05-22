---
repo: astral-sh/uv
file: 9-questions
---

# uv · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 uv 堅持一次 lock 所有平台，而不是讓 CI 在每個平台上各自 lock？**
  - 我目前的推測：Universal lockfile 的主要好處是「commit 一次，到處可用」，避免 CI 跟開發者環境的 lockfile 不同步。但代價是 resolution 時間增加（因為同時解 N 個環境維度）。對於只有 Linux 的專案，universal lockfile 是不是 overkill？[UNVERIFIED]
  - 相關程式碼：[`crates/uv-resolver/src/resolver/environment.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-resolver/src/resolver/environment.rs)

- [ ] **uv.toml vs pyproject.toml 的配置優先級為何不是單一的？**
  - 目前使用者需要同時知道 `uv.toml` 的格式和 `pyproject.toml` 中 `[tool.uv]` 的格式。為何不統一成一個？
  - 我目前的推測：`uv.toml` 是 standalone config file，`[tool.uv]` 是嵌入在 pyproject.toml 中的配置。兩者語意一致但來源不同，使用者可以選擇自己偏好的方式。但這增加了學習成本。[UNVERIFIED]
  - 相關程式碼：[`crates/uv-settings/src/lib.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-settings/src/lib.rs)

- [ ] **`Combine` trait 的實作是手動撰寫的——為何不自動 derive？**
  - 每個配置 struct 都需要手動實作 `Combine`。對於有 20+ 個欄位的 struct，這很容易遺漏邊界情況。
  - 我目前的推測：Rust 的 derive macro 對於「欄位級別合併策略不同」的情況（有的 append、有的 replace、有的 merge）很難通用化。但一個 proc macro 或許可以接受 `#[combine(append)]` 之類的 attribute。[UNVERIFIED]
  - 相關程式碼：[`crates/uv-settings/src/combine.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-settings/src/combine.rs)

- [ ] **uv 的 Python version management 跟 pyenv 比起來真正的優勢是什麼？**
  - uv 可以下載並編譯 CPython 原始碼，這是一個非常重的操作。pyenv 用 prebuilt binary 更快更輕。為何 uv 選擇從 source build？（或許因為對應多平台 prebuilt binary 的維護成本太高？）
  - 相關程式碼：[`crates/uv-python/src/downloads.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-python/src/downloads.rs)

- [ ] **PEP 723 script 的 inline metadata 解析是如何跟 workspace system 互動的？**
  - 一個 script 有自己的依賴（寫在 `// requirements: ...` comment 中），但執行時是否也能看到 workspace 的依賴？
  - 我目前的推測：`uv run script.py` 會先讀 PEP 723 metadata，若存在則建立獨立的 virtualenv 安裝 script 的依賴，不跟 workspace 共用。[UNVERIFIED]

## 想問維護者的問題

- uv 團隊在決定「這個功能放進 uv 還是不放」時的標準是什麼？例如 `uv python install` 下載 CPython 原始碼，這明顯超出「套件管理器」的邊界，但你們還是做了。
- Workspace 的 `MemberDiscovery` 支援 glob pattern（`member = ["packages/*"]`），但實作方式是自己掃描檔案系統而非讓使用者明確列出——為何偏向自動發現而不是顯式列舉？
- `uv.lock` 的格式設計參考了 Cargo.lock，但為何選擇 TOML 而非更輕量的 JSON Line 或 YAML？

## 下次再看時的待辦

- [ ] 探索 `uv-resolver` 的 batch prefetch 實作細節——它如何預測接下來需要哪些 package versions？
- [ ] 對照 Poetry 的 SAT solver 跟 uv 的 PubGrub resolver 的效能差異
- [ ] 深入閱讀 `uv-install-wheel` 的 link mode（hardlink / copy / clone / reflink）實作
- [ ] 對照 `uv-python/src/discovery.rs` 跟 pyenv 的 Python discovery 策略

## 跨專案對照備忘

- **Decoupling crate pattern**（uv-dispatch）跟 Ruff 的 `ruff_linter` / `ruff_formatter` 分離類似——Astral 團隊偏好這種「獨立中介 crate」的模組化風格。已在 Ruff (astral-sh/ruff) 和 uv 兩個 repo 觀察到，尚未滿足 _patterns/ 的 3+ repo 收錄門檻。→ **候選 pattern**（名稱：`async-pipeline-crate` 或 `decoupling-dispatch-crate`）
- **Universal lockfile + fork marker** 的設計，跟 Cargo 的 `cfg` expression 處理類似——都是在單一 lockfile 中編碼多平台資訊。但 Cargo 的 lockfile 不記錄 marker，由 Cargo.toml 的 `[target.'cfg(...)'.dependencies]` 處理。
- **Combine trait for multi-layer config** — 此模式與 Ruff 的 `ruff.toml` + `pyproject.toml` 配置載入相似。Astral 團隊在兩個工具中都使用了類似的「層疊配置合併」設計，但具體實作不同（uv 用 `Combine` trait，Ruff 用不同的 config loading 方式）。→ **候選 pattern**（名稱：`layered-config-combine`）
