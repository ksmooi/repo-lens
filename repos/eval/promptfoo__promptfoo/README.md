---
repo: promptfoo/promptfoo
file: README
studied_at: 2026-05-23
commit_sha: 1205a2a
type: eval
language: TypeScript
framework: Commander.js + React 19 + Drizzle ORM
stars: 21.5k
status: active
---

# promptfoo · 概覽

## 解決什麼問題

LLM 應用的開發目前仍高度依賴「直覺」— prompt 改了不知道是變好還是變差、換模型後行為會不會偏、production 會不會出安全問題。promptfoo 把軟體工程的測試文化帶到 LLM 開發：**宣告式 config + assertion 驅動的評估 + 自動化紅隊測試**，讓你像跑 unit test 一樣驗證 prompt 的品質與安全性。

## 為什麼值得研究

promptfoo 有三個設計面向值得關注：

1. **架構本身即 pattern** — 一個 CLI 工具如何做到「eval + red team + compare + share + view」多種功能而保持模組化，是很好的 monorepo design 案例
2. **Provider 抽象層** — 50+ provider 整合（OpenAI、Anthropic、Bedrock、自訂 HTTP、Python 腳本），用 chain-of-responsibility 而非 map lookup 的設計選擇很少見
3. **Plugin × Strategy × Grader** — redteam 子系統把「測試生成 → 攻擊變形 → 結果評分」三個階段完全解耦，是 security testing framework 的典範架構

## 技術棧一句話

`TypeScript` + `Commander.js` + `React 19 (MUI v7)` + `Drizzle ORM (SQLite)` + `Zod` + `Nunjucks`

## 健康度信號

- ⭐ Stars: ~21.5k
- 📅 最後 commit: 2026-05-23（今天仍有更新）
- 📦 最新版本: v0.121.12
- 📈 Release 頻率: 高頻（release-please 自動化，幾乎每天有 release）
- 🔄 SemVer 遵循狀況: 寬鬆（0.x 階段，breaking change 常見）
- 🏢 組織狀況: 已加入 OpenAI（2025年），但保持 MIT 開源

## 我會在後續筆記中回答的問題

- 整個 eval pipeline 從 config 到結果輸出的完整流程？
- 50+ provider 的解析機制是怎麼設計的？
- Redteam 的 plugin × strategy × grader 如何分工？
- 這個工具的競爭定位 vs DeepEval / LangFuse / Arize 的差異？

## 競品比較

| 面向 | promptfoo | DeepEval | LangFuse / LangSmith | Arize Phoenix |
|---|---|---|---|---|
| 主要抽象 | 宣告式 YAML + CLI | pytest 風格的斷言 | 追蹤 + 評估平台 | 可觀測性平台 |
| 部署方式 | 100% 本地 CLI + Node SDK | Python library | SaaS / 自託管 | SaaS / 自託管 |
| Red teaming | 內建完整系統（100+ plugins） | 基礎支援 | 無內建 | 無內建 |
| Provider 覆蓋 | 50+（含 HTTP/generic） | 主要透過 LLM-as-Judge | 主要追蹤現有呼叫 | 主要追蹤現有呼叫 |
| 主要 trade-off | CLI-first，需要 YAML 設定 | Python-first，測試整合自然 | 平台依賴，資料送外部 | 平台依賴 |
| CI/CD 整合 | CLI 指令，exit code 驅動 | pytest 原生 | SDK callback | SDK callback |
