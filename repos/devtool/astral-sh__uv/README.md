---
repo: astral-sh/uv
file: README
studied_at: 2026-05-22
commit_sha: 135a363
type: devtool
language: Rust
category: cli-tool
api_style: imperative
stars: 85.3k
status: active
---

# uv · 概覽

## 解決什麼問題

把 Python 套件管理的七把瑞士刀（`pip`、`pip-tools`、`pipx`、`poetry`、`pyenv`、`twine`、`virtualenv`）全部整合進一支單一二進位。速度比等價的 `pip` 操作快 **10–100x**（以 warm cache 衡量）。

更關鍵的是它解決了 Python 生態的核心矛盾：**pip 沒 lockfile → 依賴不確定；Poetry 有 lockfile 但只鎖單一平台 → 團隊協作容易崩。** uv 的 universal lockfile 一次鎖定所有平台、所有 Python 版本、所有 marker expression 分支，這是它跟所有既有工具的關鍵差異。

## 為什麼值得研究

1. **multi-crate Rust workspace 的模組化藝術** — ~58 個 crate，每個 crate 的責任極度單一（`uv-resolver` 只管解版本、`uv-client` 只管 HTTP、`uv-git` 只管 git dependency、`uv-cache` 只管快取），透過 `uv-dispatch` 這個「循環依賴破壞者」來協調。這是大型 Rust monorepo 的典範級實作。

2. **Universal resolver 的工程取捨** — 用 PubGrub 演算法做 universal resolution，支援 marker expression 的 fork-based 版本選擇。這是 PyPI 生態中第一個生產級 universal resolver。

3. **Cargo 風格的 workspace 系統** — 把 Cargo workspace 的概念搬進 Python 生態，包含 member discovery、依賴繼承、共享 lockfile。值得想設計「多專案協作工具」的人參考。

4. **配置層疊系統** — CLI args > project `uv.toml` > user config > system config，用 `Combine` trait 來 merge，設計乾淨。

## API 風格一句話

Imperative CLI + 子命令分類（`pip`、`project`、`tool`、`python`、`workspace` 五個 namespace），配 `--help` 友善的詳盡提示。不強迫使用者用 declarative config（不像 Poetry），但也支援完整的 declarative config。

## 技術棧一句話

`Rust` + `tokio`（async runtime）+ `pubgrub`（依賴解析）+ `clap`（CLI）+ `reqwest`（HTTP）+ `tracing`（日誌）

## 健康度信號

- ⭐ Stars: ~85.3k
- 📅 最後 commit: 2026-05-21
- 📦 最新版本: 0.11.16
- 📈 Release 頻率: 幾乎每日或每兩日
- 🔄 SemVer 遵循狀況: 嚴格（0.xx 階段但已趨近 1.0，版本號從 0.1.x 一路到 0.11.x，變更管理嚴謹）

## 競品比較

| 面向 | uv | Poetry | pip + pip-tools | PDM |
|------|-----|--------|----------------|-----|
| 實現語言 | Rust | Python | Python | Python |
| Relative speed | 1x (基準) | ~10–50x slower | ~10–100x slower | ~10–50x slower |
| Lockfile 類型 | universal (跨平台) | single-platform | 無 (pip-tools 只產 requirements.txt) | universal |
| 安裝速度 | 極快 (並行 wheel install) | 中等 | 慢 (序列化) | 中等 |
| Python 版本管理 | 內建 | 無 (需 pyenv) | 無 | 內建 |
| Workspace 支援 | 有 (Cargo 風格) | 有限 | 無 | 有 |
| PEP 723 scripts | 支援 | 不支援 | 不支援 | 支援 |
| pip 相容介面 | `uv pip` | 無 | 原生 | 有 |
| 維護者 | Astral (Ruff 團隊) | 社群 (Sébastien Eustace) | PyPA 社群 | Frost Ming |
| License | Apache-2.0 / MIT | MIT | MIT | MIT |

uv 做得最極致的是「**速度 + 整合度**」的雙重取捨：為了快用 Rust，為了整合把所有功能塞進一支二進位。代價是 binary 較大（~30–50MB），以及功能邊界變得模糊（例如 `uv python install` 下載 CPython 原始碼後編譯，這跟 pip 的職責距離很遠）。
