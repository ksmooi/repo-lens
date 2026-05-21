# devtool/ — 開發工具

**上層目錄**: [repos/](../README.md)

CLI 工具、build tool、linter、formatter、開發輔助工具。
學習重點在於「如何設計讓開發者每天都想用的工具」的 UX 與工程取捨。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **CLI 設計** | subcommand 組織方式?flag 命名慣例?help text 品質?exit code 規範? |
| **Build Graph** | task 的 dependency 怎麼宣告?incremental build 怎麼判斷(hash / mtime)?cache 策略? |
| **Plugin / Extension** | plugin 怎麼發現(entry_points / 目錄掃描)?API 穩定性?沙盒執行? |
| **Config 載入** | config 格式(TOML / YAML / JSON)?優先級(env > file > default)?驗證機制? |
| **效能** | 啟動時間?並行執行策略?Rust / Go 實作的效能優勢在哪? |
| **錯誤訊息設計** | 錯誤訊息清不清楚?有沒有建議修復?span / source location 標示? |
| **Cross-platform** | Windows / macOS / Linux 的差異怎麼處理?path separator?shell 相容性? |

套用模板:`_templates/library.md`

---

## 典型專案

| 專案 | 語言 | 功能 |
|------|------|------|
| Ruff | Rust | Python linter + formatter、極快、Flake8 / Black 替代 |
| uv | Rust | Python 套件管理、pip + venv 替代、pip 100x 速度 |
| Turborepo | Go / Rust | monorepo build system、remote cache、task graph |
| Just | Rust | command runner(Makefile 替代)、跨平台、簡潔語法 |
| Mise | Rust | 多語言版本管理(asdf 替代)、hooks、tasks |
| Biome | Rust | JS / TS linter + formatter、Prettier / ESLint 替代 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 給開發者日常使用的 CLI / build 工具 | ✅ `devtool` | — |
| 通用函式庫(被 import 使用而非執行) | — | ✅ [`library`](../library/) |
| 容器 / 部署 / 監控相關工具 | — | ✅ [`infra`](../infra/) |
| 程式碼分析 / 測試框架 | — | ✅ [`eval`](../eval/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
