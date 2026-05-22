---
repo: gin-gonic/gin
file: README
studied_at: 2026-05-22
commit_sha: 5f4f964
type: backend-framework
language: Go
framework: Gin
stars: 88.6k
status: active
---

# Gin · 概覽

Go 生態系中最普及的 HTTP 框架：以極簡 API + 自訂 Radix Tree 路由引擎，在 Martini 風格開發體驗與高吞吐量之間取得平衡。

## 解決什麼問題

Go 標準函式庫 `net/http` 的路由器只支援精確路徑匹配，沒有 URL 參數、middleware chain、請求綁定等現代 Web 框架該有的設施。在 Gin 出現之前（2014），Martini 是 Go 最受歡迎的框架，但它的反射-heavy 設計讓效能慘不忍睹（每請求 ~300 次反射 call）。Gin 保留了 Martini 的 `c.Get()/c.Set()` 和 middleware chain 開發體驗，但把自己搭建的 Radix Tree 路由器和 `sync.Pool` 式 context 重用作效能核心，打出「Martini-like API, up to 40x faster」的口號。

## 為什麼值得研究

- **Radix Tree 路由引擎**：自行實作的壓縮前綴樹，是 Gin 效能的關鍵。跟 `http.ServeMux` 的線性掃描或 `gorilla/mux` 的正則表達式比，設計取捨極具教育意義。
- **Middleware chain 的扁平化設計**：Gin 不在執行期嵌套 middleware，而是在路由註冊期就把 middleware + handler 拍平成單一 slice。這個「編譯期合併」的思路跟大多數框架不同。
- **介面分割的 border-line 範例**：binding 拆成 `Binding` / `BindingBody` / `BindingUri` 三組介面，render 統一成 `Render` 介面。分割得剛剛好——不多不少，同時保持了擴充性。
- **Go 框架 API 設計的歷史文件**：gin 的 API 演進（從 `gin.Default()` 到 `ginS` 單例、從 `binding.JSON` 全域變數到 `codec/json` 抽象層）記錄了 Go 生態怎樣從簡單粗暴走向模組化。

## 技術棧一句話

`Go` + (`net/http` standard library) + `go-playground/validator` + optional `json-iterator/go` / `sonic` + optional `goccy/go-yaml`

## 健康度信號

- ⭐ Stars: ~88.6k
- 📅 最後 commit: 2026-05-09
- 👥 主要維護者: thinkerou、appleboy、javierprovecho 等 5-7 人
- 🔄 commit 頻率: 每月 10-30 筆，活躍
- 📦 最新版本: v1.10+（2025-2026 持續發布）
- 🏢 贊助方: 無，社群驅動

## 我會在後續筆記中回答的問題

- Gin 的 Radix Tree 跟 `net/http` 的 ServeMux、`gorilla/mux`、`chi` 的 routing 差異在哪？
- Middleware chain 的扁平化設計有什麼 trade-off？
- Gin 的 Context 為什麼要 pool？Copy() 是做什麼用的？
- binding 跟 render 的介面設計為什麼是這樣拆？
- Gin 的「極簡」API 背後，哪些設計是歷史包袱？

## 競品比較

| 面向 | Gin | chi | gorilla/mux | net/http (1.22+) |
|------|-----|-----|-------------|------------------|
| 路由核心 | 自訂 Radix Tree | 自訂 Radix Tree (derived from httprouter) | 正則表達式 map | Trie (Go 1.22 新增) |
| Middleware 機制 | Handler 前綴合併 | 標準 `http.Handler` 嵌套 | 標準 `http.Handler` 嵌套 | 無原生支援 |
| 路由效能 (註冊 100 條) | 極高 (O(k) per match) | 高 | 中-高 (regex dep.) | 中 (Go 1.22+, 仍較慢) |
| Context 設計 | 自訂 `*gin.Context` + sync.Pool | 標準 `context.Context` | 自訂 `map[string]string` | 無 |
| 請求綁定 | 內建 (JSON/XML/Form/YAML/ProtoBuf/TOML/BSON) | 無內建 | 無內建 | 無 |
| 回應渲染 | 內建 6 種 JSON variant + XML/YAML/ProtoBuf/HTML | 無內建 | 無內建 | 無 |
| 套件依賴 | 少（validator + 可選 JSON lib） | 極少（無外部依賴） | 少 | 零依賴（stdlib） |
| 主要 trade-off | 自訂 Context 脫離標準介面，無法直接跟 `http.Handler` 互操作 | 完全相容 `http.Handler`，擴充性最好 | 支援 regex route，最靈活但也最慢 | 標準化但功能最簡 |
