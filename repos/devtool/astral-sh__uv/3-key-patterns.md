---
repo: astral-sh/uv
file: 3-key-patterns
---

# uv · 值得偷學的設計

## Pattern 1: Decoupling Crate（循環依賴破壞者）

**是什麼**：

把一個跨多個 core crates 的介面（`BuildContext` trait）提取到一個獨立的 `uv-types` crate 中，然後用一個專門的 `uv-dispatch` crate 來實作這個 trait。這樣 resolver、installer、build 三者只需要依賴 `uv-types`，不需要互相依賴。

**為什麼有效**：

Rust 的 crate 系統不允許 cyclic dependencies——這是編譯時的強制檢查。`uv-dispatch` 的出現讓圖論上的循環依賴被一個「知道所有東西」的中樞 crate 吸收。這個 pattern 等效於 OOP 中的 DI container，但透過 Rust 的 trait system 在編譯期完成。

**程式碼位置**：
- Trait 定義：[`crates/uv-types/src/build.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-types/src/build.rs) — `BuildContext` trait
- 實作：[`crates/uv-dispatch/src/lib.rs:109`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-dispatch/src/lib.rs#L109) — `BuildDispatch` struct
- 使用範例：`uv-resolver` 的 `ResolverProvider` trait 使用 `BuildContext` 來取得 package metadata

**替代方案**：
- **Python 生態的 monkeypatching**（pip 的做法）：在執行期把需要的東西塞進 module-level variable。靈活但沒有型別安全。
- **Cargo workspace 內部的 path dependency**：直接依賴另一個 crate。簡單但無法打破循環。
- **Service locator pattern**：全域註冊表。型別不安全且測試困難。

**何時可以借用**：
當你有一個大型 Rust workspace 且發現 resolver 和 builder 需要互相參照時。不要試圖把兩者合併——分離但用一個「連接器 crate」來協調，比混在一起更乾淨。

**注意事項**：
這個 pattern 的前提是你願意接受一個「知道太多」的 crate。`BuildDispatch::new()` 有 ~20 個參數——每個新需求都增加一個參數，沒有 DI framework 的注入機制。

## Pattern 2: Combine Trait for 多層配置合併

**是什麼**：

用一個 `Combine` trait 定義配置來源的合併行為，讓多層配置（CLI > project > user > system）能優雅地疊加。

```rust
pub trait Combine {
    fn combine(self, other: Self) -> Self;
}
```

**為什麼有效**：

Python 工具（如 pip）用 dict merge 或 configparser 的 section overlay 來處理配置合併。uv 用 trait 把合併邏輯型別化——每個配置欄位都知道自己的合併規則：

- `Option<T>`：`Some` 覆蓋 `None`，兩者都 `Some` 時保持優先級較高者
- `Vec<T>`：串接（但不是所有欄位都這樣）
- `enum`：高優先級覆蓋低優先級（但有例外）

**程式碼位置**：[`crates/uv-settings/src/combine.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-settings/src/combine.rs)

**替代方案**：
- **serde merge**：把多個設定檔 deserialize 到同一個 struct，後者覆蓋前者。簡單但無法自訂合併邏輯（例如 append vs replace）。
- **一個 dict 疊一個 dict**（pip 的方式）：靈活但 type safety 差。
- **Typed config builder**（如 `config` crate）：功能強大但開發複雜度高。

**何時可以借用**：
任何需要處理多層配置來源的 CLI 工具或 library。特別是當某些欄位的合併行為需要 `append`（如 extra index URLs）而某些需要 `replace`（如 cache dir）時。

## Pattern 3: PubGrub + Fork-Based Universal Resolution

**是什麼**：

用 `pubgrub` 演算法作為核心，但對有 marker expression 差異的套件進行「fork」——每個 fork 獨立解析，最後合併進單一 lockfile。

**為什麼有效**：

傳統 resolver（如 Poetry 的 SAT solver）只能處理一個「環境」的解析。Python 生態的 marker expressions 讓同一套依賴在不同平台可能有不同版本需求。Fork-based 方法在概念上等同於把 N 個環境的解析平行化：

```
Input:  requests; sys_platform == "win32"
Fork 1 (win32=true):  requests==2.28
Fork 2 (win32=false): requests==2.31
Merge:  {requests: {2.28 if win32, 2.31 otherwise}}
```

**程式碼位置**：
- Fork 可能性判斷：[`crates/uv-resolver/src/resolver/environment.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-resolver/src/resolver/environment.rs#L68) — `ForkingPossibility`
- Fork index 管理：[`crates/uv-resolver/src/fork_indexes.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-resolver/src/fork_indexes.rs)
- Universal marker：[`crates/uv-resolver/src/universal_marker.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-resolver/src/universal_marker.rs)

**替代方案**：
- **Single-platform lock**（Poetry）：團隊每人跑 `poetry lock` 後 push，CI 可能跟開發者環境不符合。
- **Multiple lockfiles**（pip-tools）：每種平台產一個 `requirements-win.txt`、`requirements-linux.txt`。工作量大且容易遺漏。
- **Defer to runtime**（pip）：不安裝 platform-specific 依賴，讓使用者執行時報錯再補。最常見也最痛苦。

**何時可以借用**：
當你的依賴系統有多環境（不同 OS / Python 版本 / 硬體架構 / feature flags）需求，且你想用單一 lockfile 管理全部時。

## Pattern 4: uv.lock — 冗餘但完整的序列化格式

**是什麼**：

uv.lock 是一種 **TOML-based 的 lockfile**，格式是 `[[distribution]]` array of tables，每條記錄一個 package 的完整資訊（source URL、hash、dependencies、marker 等）。格式設計明顯受 `Cargo.lock` 啟發，但比它更冗餘。

**為什麼有效**：

一個 lockfile 包含所有版本資訊，使得 `uv sync` 不需要重新解析依賴——它只是讀取 lockfile 並下載對應的 wheel。這讓 install 操作 O(1) 於依賴圖的複雜度。

關鍵設計選擇：
- `[[distribution]]` 有 `index` 記錄來源索引（`[uv.tool.uv.sources]`），這讓 lockfile 能獨立於任何配置檔案
- 每個記錄都有 `sdist` 和 `wheel` 的 hash，支援強驗證
- `marker` 欄位記錄該記錄的適用環境（universal lockfile 的關鍵）

**程式碼位置**：[`crates/uv-resolver/src/lock/export.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-resolver/src/lock/export.rs) — `PylockToml` 序列化

**替代方案**：
- **Poetry lockfile**：類似但較簡化，不走 universal。
- **pip-tools requirements.txt**：純文字 list，沒有 hash、沒有 marker、沒有 dependency tree。
- **Cargo.lock**：格式相似但更輕量（沒有 marker 和 multi-index 的概念）。

**何時可以借用**：
當你的工具需要一個「凍結所有依賴版本」的檔案，且需要支援多平台或多環境時。

## Pattern 5: CLI Namespace 的組合設計

**是什麼**：

uv 的 CLI 分成五個主要 namespace（`pip`、`project`、`tool`、`python`、`workspace`），每個 namespace 有獨立的子命令。但這些 namespace 並非全部獨立——`pip` namespace 提供了傳統 pip 相容的介面，而 `project` namespace 是 uv 原生的工作流。

這個雙軌設計讓使用者可以逐步遷移：先用 `uv pip install`（熟悉），再用 `uv add` / `uv sync`（高效）。

**為什麼有效**：

這解決了工具採用的經典「冷啟動問題」：使用者需要立即的生產力，而不是先花半小時學習新哲學。`uv pip` 是拐杖，`uv project` 是終點。

**程式碼位置**：
- CLI 定義：[`crates/uv-cli/src/lib.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-cli/src/lib.rs) — 用 `clap::Subcommand` derive macro 定義
- pip 子命令：[`crates/uv/src/commands/pip/`](https://github.com/astral-sh/uv/blob/135a363/crates/uv/src/commands/pip/)
- project 子命令：[`crates/uv/src/commands/project/`](https://github.com/astral-sh/uv/blob/135a363/crates/uv/src/commands/project/)

**替代方案**：
- **Poetry**：只有 `poetry add`、`poetry install`，不提供 pip 相容介面——導致某些使用者無法完全採用。
- **PDM**：有 `pdm add` 和 `pdm install`，也有 `pdm use pip` 但比較是附加功能。

**何時可以借用**：
當你設計一個「取代既有工具」的新工具時。提供一條「老 API 相容 → 原生 API」的漸進路徑，遠比強迫使用者立即跳船有效。

## Pattern 6: Batch Prefetch（解析時的並行預取）

**是什麼**：

`BatchPrefetcher` 在 resolver 解析依賴時，會往前看需要哪些 package versions，並預先 fetch 它們的 metadata。這讓網路 I/O 跟 CPU 密集的 PubGrub 解析重疊進行。

**為什麼有效**：

一旦 PubGrub 決定要測試一個 package version，fetch metadata 的需求是確定的。與其等解析到那一步才 fetch，不如在解析還在算目前節點時，就把「很快會需要」的 metadata 全部發 request。

**程式碼位置**：[`crates/uv-resolver/src/resolver/batch_prefetch.rs`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-resolver/src/resolver/batch_prefetch.rs)

**替代方案**：
- **Lazy fetch**（pip）：只有在決策需要時才 fetch，慢但簡單。
- **Aggressive prefetch**：一次 fetch 所有已知版本的 metadata。快但浪費頻寬。

**何時可以借用**：
任何有「決策需要 I/O 結果」的系統。關鍵是「可預測性」——當你能預測接下來需要什麼資料時，prefetch 是最便宜的加速手段。

## API 設計品味的觀察

uv 的 CLI 設計偏好「顯式但簡潔」：

- **子命令清晰**：`uv add`、`uv lock`、`uv sync` vs `uv pip install`、`uv pip compile`——前者是使用者友好、後者是 pip 相容
- **錯誤訊息**：大量使用 `miette` crate 產出 rich diagnostic（有顏色、有 hints、有 code frame）
- **--help 品質**：每個參數都有說明，子命令有「aliases」的標記。`clap` 的 `wrap_help` feature 確保長說明不被截斷
- **環境變數**：`UV_` prefix 的一致性（[`uv_static::EnvVars`](https://github.com/astral-sh/uv/blob/135a363/crates/uv-static/src/lib.rs) 定義所有支援的 env vars）

一個有趣的設計選擇是 `--preview` 機制：新功能先以 `--preview` 旗標進入，經過幾個版本後才變成預設行為。這讓開發者可以在正式 release 前收集 feedback。

## 對相容性的態度

從 CHANGELOG.md 的長度（51K+ 字）可以看出 uv 對 breaking change 的管理態度：

- **Deprecation cycle**：舊參數先標 `warn_user!()` 警告，數個版本後才移除。例如 `--isolated` 被 `--no-config` 取代時，先用 warning 通知了至少 3 個 minor release
- **Preview feature gate**：所有可能改變既有行為的新功能都先以 --preview 進入，經過 1-2 個 minor release 後才變成預設
- **Backward compat 優先**：`uv pip` 子命令故意保留跟 pip 99% 相同的 CLI 選項（甚至包括 pip 的 deprecated 選項），讓遷移成本趨近於零
