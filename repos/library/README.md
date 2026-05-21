# library/ — 函式庫與 SDK

**上層目錄**: [repos/](../README.md)

通用函式庫、SDK、Python / Rust / TypeScript 套件。
學習重點在於「如何設計讓使用者願意依賴的 API」——簡潔、穩定、可擴充。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **公開 API 設計** | 薄抽象還是厚抽象?流暢 API 還是顯式配置?型別系統怎麼運用? |
| **擴充機制** | plugin / hook / subclass?擴充點的 API 穩定性承諾?third-party 生態? |
| **公開 vs 內部界線** | `__all__` / explicit exports?`_xxx` 命名規則?internal module 怎麼隔離? |
| **錯誤設計** | 錯誤類型層級?exception vs result type?錯誤訊息品質? |
| **型別標注** | PEP 561 / stub file?泛型設計?runtime 型別檢查 vs static only? |
| **相容性策略** | semver 遵循?deprecation warning 機制?minimum Python / runtime 版本? |
| **測試策略** | property-based testing?fuzz testing?cross-version CI matrix? |

套用模板:`_templates/library.md`

---

## 典型專案

| 專案 | 語言 | 功能 |
|------|------|------|
| Pydantic | Python | 資料驗證、型別強制、settings 管理 |
| Tenacity | Python | retry 策略、backoff、stop condition |
| Tokio | Rust | async runtime、task scheduler、IO driver |
| Zod | TypeScript | schema validation、型別推導、parse > validate |
| Instructor | Python | LLM structured output、Pydantic 整合 |
| Loguru | Python | Python logging 替代、sink 抽象、結構化 log |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 通用函式庫、SDK(被 import 使用) | ✅ `library` | — |
| 給開發者執行的 CLI / build 工具 | — | ✅ [`devtool`](../devtool/) |
| 後端框架本體(路由 / ORM) | — | ✅ [`backend-framework`](../backend-framework/) |
| 分散式系統 / 資料庫引擎 | — | ✅ [`infra`](../infra/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
