---
repo: HKUDS/CLI-Anything
file: README
studied_at: 2026-05-21
commit_sha: 436a4f5
type: devtool
language: Python
framework: Click
stars: 38.7k
status: active
---

# CLI-Anything · 概覽

## 解決什麼問題

GUI 軟體是為人類設計的——有滑鼠、有螢幕、有圖形介面。但 AI agent 沒有這些東西。CLI-Anything 解決的是「**如何讓 AI agent 能操控任何 GUI 軟體**」這個根本問題。

方案很直接：為每個 GUI 軟體**自動生成一個完整、有狀態的 CLI**，讓 agent 可以用子命令、REPL 和 JSON 輸出來操控軟體。支援 Claude Code、Pi、OpenCode、Codex、OpenClaw、Qodercli、Goose、GitHub Copilot CLI 等主流 AI coding agent。

核心主張：**Today's Software Serves Humans. Tomorrow's Users will be Agents.**

## 為什麼值得研究

- **38.7k stars**、687 commits、2 個月從零爆發到 50+ 個 community harnesses——開源史上成長最快的 CLI 專案之一
- 本質上是**一套生產 CLI 的工廠**（meta-CLI tool），不是只做一個 CLI
- 內建完整的 CLI 生態系統：方法論（HARNESS.md）+ 產生器（7-phase pipeline）+ 套件管理器（CLI-Hub）+ AI skill 抽象層
- 統一 REPL Skin 設計讓每個產出的 CLI 都有相同的 UX 品味
- 是「**AI agent 與既有軟體生態的橋樑**」這個命題的 open-source reference implementation

## API 風格一句話

Click 子命令群組 + 可選 REPL（有狀態互動模式）+ `--json` 雙模輸出（human/machine readable）。

```bash
# 單次操作
cli-anything-blender scene new --name "MyScene" --json

# 或進入 REPL 進行多步驟操作
cli-anything-blender
◆ cli-anything · Blender   v1.0.0
  ◇ Install:   npx skills add HKUDS/CLI-Anything --skill cli-anything-blender -g -y
blender ❯ scene new --name "MyScene"
```

## 技術棧一句話

`Python` + `Click` + `prompt_toolkit` + `PEP 420 namespace packages`

## 要跟誰比

| 面向 | CLI-Anything | 直接寫 Python script | MCP (Model Context Protocol) | Shell script wrapper |
|---|---|---|---|---|
| 抽象層 | 完整的 7-phase 方法論 | 無 | 協定層標準 | 無 |
| 狀態管理 | 內建（REPL + session JSON） | 手動 | 無狀態 | 無狀態 |
| AI agent 發現 | SKILL.md + 統一格式 | 無 | tool listing 協定 | 無 |
| 套件管理 | CLI-Hub registry | 手動 pip | 無 | 手動 |
| 產出品質 | 統一 REPL Skin + 測試 強制 | 看作者 | 看實作 | 看作者 |
| 支援 agent 平台 | 8+ 平台 | 任何可裝 Python 的 | 僅支援 MCP client | 任何支援 shell 的 |

## 健康度信號

- ⭐ Stars: ~38.7k
- 📅 最後 commit: 2026-05-20
- 📦 687 commits, 50+ harnesses
- 🔄 Release 頻率: 幾乎每日（news section 每日更新）
- 👥 主要維護者: `yuh-yang`（HKUDS 團隊），社群貢獻者 40+
- 🏢 贊助方: 香港大學數據科學實驗室（HKUDS）

## 我會在後續筆記中回答的問題

- **7-phase pipeline** 各階段的具體產出與品質控制是怎麼做的？
- **PEP 420 namespace packages** 為什麼是這個架構的關鍵？
- CLI-Hub 的 registry 與 install 機制如何做到多來源（pip / npm / brew / bundled）？
- 統一 REPL Skin 的設計取捨：為什麼每個 CLI 共用同一份 skin，而不是各自實作？
- SKILL.md 抽象層如何讓 AI agent 發現並使用這些 CLI？
