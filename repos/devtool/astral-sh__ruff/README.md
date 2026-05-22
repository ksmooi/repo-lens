---
repo: astral-sh/ruff
file: README
studied_at: 2026-05-22
commit_sha: 8c04080
type: devtool
language: Rust
framework: None
stars: ~47.6k
status: active
---

# Ruff · 概覽

## 解決什麼問題

Ruff 是一個用 Rust 寫的 **Python linter 與 code formatter**，目標是成為 Python 生態中最快的靜態分析工具。它能取代 Flake8（及其數十個 plugin）、isort、Black 和 Pylint 的部分功能，而且**快幾個數量級**——官方聲稱比 Flake8 快 10-100 倍、比 Black 快 20 倍以上。

更重要的是，Ruff 正在擴展為 Python **type checker**。2024 年下半年起 Ruff 開始內建型別檢查功能（以 `ruff check --preview` 或獨立的 `ruff ty` 指令可用），目標是逐步覆蓋 mypy 和 Pyright 的領域。

## 為什麼值得研究

Ruff 是非 Python 語言實作 Python 工具鏈的典範。它展示了：

- **如何從零開始寫一個 Python 解析器**（`ruff_python_parser`），支援完整的 Python 3.7–3.14 語法
- **如何組織數百條 lint rule**（80+ 個規則模組，跨 Flake8 生態），採用多 pass checker 架構
- **如何同時實現 formatter 的等價性與可讀性**（以 Black 的 output 為 gold standard）
- **如何用 Rust 的類型系統來做 Python 的型別推論**（`ty_python_semantic` 達 13 萬行）
- **一個工具如何逐步吞噬相鄰領域**（linter → formatter → type checker → LSP server）

## API 風格一句話

CLI-first + 設定檔驅動（`pyproject.toml` / `ruff.toml` / `.ruff.toml`），零配置即可使用。

## 技術棧一句話

`Rust` (edition 2024) + `clap` (CLI) + `mimalloc`/`jemalloc` (分配器) + `rayon` (並行)

## 健康度信號

- ⭐ Stars: ~47.6k
- 📅 最後 commit: 2026-05-21（幾乎每日 update）
- 📦 最新版本: 0.15.14（2026-05-22 附近）
- 📈 Release 頻率: 每月 2-3 個 minor release
- 🔄 SemVer 遵循狀況: 寬鬆（minor 版本常含破壞性變更，major 尚未發布）

## 三句話總結架構

| 層次 | 核心 crate | 職責 |
|---|---|---|
| CLI 層 | `crates/ruff` | 參數解析、指令調度、結果輸出 |
| Linter 核心 | `crates/ruff_linter` | 多 pass checker（token → AST → import → physical lines） |
| 型別檢查 | `crates/ty*` | 全 Python 型別推論引擎（`ty_python_semantic` 達 13 萬行） |

## 跟競品的比較

| 面向 | Ruff | Flake8 + plugins | mypy | Black |
|---|---|---|---|---|
| 主要抽象 | 多 pass checker | 單 pass AST visitor | 型別圖 | CST/comment-preserving |
| 語言 | Rust | Python | Python | Python |
| 執行速度 | 10-100x faster | baseline | ~5x slower | ~20x slower |
| 規則數量 | 800+ 條（含 plugin 等價實作） | ~200（含 plugin） | 型別檢查為主 | N/A（formatter only） |
| 重要 trade-off | Rust 開發成本高，但換來極致速度 | Python plugin 容易寫，但慢 | 型別系統最完備，但效能差 | 輸出品質業界標準，但不可配置 |

