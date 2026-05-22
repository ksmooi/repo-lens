---
repo: gin-gonic/gin
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 5f4f964
---

# Gin · 值得偷學的設計

## Pattern 1: 編譯期 Middleware 扁平化（Eager Handler Flattening）

**是什麼**: Middleware 和 handler 在路由註冊時就直接合併成一個扁平的 `HandlersChain`（`[]HandlerFunc`），而非在執行期透過嵌套 `http.Handler` 組合。

**為什麼有效**: 
- 執行時只有一個 for loop，沒有函式呼叫鏈，沒有 interface dispatch 開銷。
- 每個 middleware 的執行順序在註冊時就確定了，不會有執行期動態排序的不可預測性。
- 讓 `c.Next()` 的實作極簡——只是 index++ 和 slice iterate。

**程式碼位置**: 
- Middleware 註冊: [`routergroup.go:65-68`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/routergroup.go#L65) — `Use()`
- Middleware + handler 合併: [`routergroup.go:241-248`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/routergroup.go#L241) — `combineHandlers()`
- Chain 執行: [`context.go:188-196`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/context.go#L188) — `Next()`

**何時可以借用**:
- 在語言支援預先計算（pre-compute）完整 handler chain 的場景，例如 Go 或 Rust 這類編譯型語言。
- 當 middleware 的結構在請求到達前就完全已知（大多數 Web 框架都是如此）。
- 適合 middleware 數量和路由數量在可預測範圍內的專案。

**替代方案**:
- **Nested Handler 模式**（Go stdlib/chi）: 每個 middleware 包裝下一層，形成洋蔥圈。優點是每個 middleware 都是標準 `http.Handler`，可互操作；缺點是每層都有 func call overhead。程式碼: [`chi example`](https://github.com/go-chi/chi/blob/master/chain.go)
- **Decorator 模式**（Python/Java）: 使用裝飾器或 annotation 鏈。語法彈性高，但 middleware 間資料傳遞依賴 thread-local 或 request-scoped container。

**注意事項/何時不用**:
- 若 middleware 需要在執行期基於請求內容動態決定是否執行，扁平化的 chain 無法跳過已註冊的 middleware（只能在 chain 內部自己檢測條件後 `c.Next()` 或 return）。
- 每個路由都複製完整 chain（包含所有 middleware），大量路由（>10k）時記憶體開銷顯著。
- 對每個 middleware 做獨立單元測試變得困難（因為無法測試單個 middleware 而不觸發整個 chain）。

---

## Pattern 2: Radix Tree 優先級排序（Dynamic Priority Child Ordering）

**是什麼**: 樹節點的孩子陣列根據 `priority` 欄位動態排序，頻繁命中的路由被推到最前面，減少遍歷次數。

**為什麼有效**: 
- HTTP 路由並非均勻分佈——某些路由（`/`, `/health`, `/api/v1/users`）的訪問頻率遠高於其他。
- 透過在 `addRoute()` 對路過的 child 節點 increment priority，並用 bubble sort 把高優先級節點往前推，樹的自適應能力讓平均匹配時間逼近 O(1)。
- 不需要配置「熱路由清單」，也不用執行期的存取計數器——這是純粹的插入期優化。

**程式碼位置**: 
- Priority increment: [`tree.go:111-131`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/tree.go#L111) — `incrementChildPrio()`
- Wildcard 節點固定到末尾: [`tree.go:71-78`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/tree.go#L71) — `addChild()`

**替代方案**:
- **靜態 Trie**（標準 radix tree）: 不排序孩子，每次從頭開始遍歷以匹配字首。實作簡單但找路徑效能隨路由數量線性下降。
- **Hash Map**（gorilla/mux 的部分實作）: 對靜態 path 使用 `map[string]Handler`，O(1) 查詢。但對於包含 `:param` 的路由仍需要線性掃描或正則比對。

**注意事項/何時不用**:
- Priority 只在 `addRoute()` 時遞增，不會在執行期動態調整。若熱路由在冷路由之後註冊，插入期的優先級可能不一致。
- 對孩子較少（<5）的節點，排序的效益可忽略。
- Bubble sort 在極端情況下（單節點數十個孩子）是 O(n²)，但現實中的路由樹很少發生。

---

## Pattern 3: Context sync.Pool 搭配零次 allocation 寫入

**是什麼**: `*gin.Context` 透過 `sync.Pool` 管理，內部的 `responseWriter` 在 struct 中 inline 嵌入而非指標，`Params` 和 `skippedNodes` 透過指標共享 pool 的 backing array。

**為什麼有效**:
- 每個請求的生命期很短（通常 <100ms），大量 short-lived 物件分配會對 GC 產生壓力。
- `reset()` 方法將所有 slice truncate 到長度 0（而非 nil），保留 backing array 給下個請求重用。
- `writermem` 是 `responseWriter` struct 的 inline 字段，不是指標——這避免了 response writer 的獨立 heap allocation。
- `params` 和 `skippedNodes` 是 `*Params` 和 `*[]skippedNode` 指標，讓 radial tree 可直接修改呼叫者的 buffer，不用 copy。

**程式碼位置**: 
- Pool 建立: [`gin.go:229-231`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/gin.go#L229)
- Context reset: [`context.go:103-118`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/context.go#L103)
- Pool Put: [`gin.go:674`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/gin.go#L674)

**替代方案**:
- **Per-request allocation**（多數 Web 框架預設）: 每個請求 `new` Context 和 ResponseWriter。開發簡單，但無法避免 GC 壓力。
- **Arena 分配器**（Uber's `go.uber.org/fx` 和某些 proxy 專案）: 使用自訂記憶體池管理和手動釋放。效能更高但顯著增加複雜度和潛在的安全漏洞。

**注意事項/何時不用**:
- `sync.Pool` 的物件在 GC 後可能被回收，不能用來保證物件一直存在。
- `reset()` 遺漏任一欄位都可能造成夾帶前個請求的資料——這是使用物件池最常見的 bug。
- 若 handler 在 goroutine 中使用 `*Context`（如 `go func() { c.JSON(...) }()`），會造成 race condition。Gin 提供 `c.Copy()` 但極少被正確使用。
- `maxParams` 在 `pool.New` closure 中捕獲的值可能與實際註冊後的值不一致（雖然 Go 的 closure 是動態讀取，所以沒問題）。

---

## Pattern 4: 三層 Binding 介面分割（Interface Segregation for Binding）

**是什麼**: 請求綁定不使用單一統一的 `Bind()` 介面，而是拆成 `Binding`（讀 `req.Body`）、`BindingBody`（附加預讀位元組的 `BindBody()`）、`BindingUri`（完全獨立的 URI 參數綁定）三個介面。

**為什麼有效**:
- `BindingUri` 的輸入不是 `*http.Request`，而是 `map[string][]string`——若是單一介面，URI binder 必須忽略 `*http.Request` 參數。
- `BindingBody` 的存在允許框架層先讀取 body（如用在 Logger 中介層記錄請求內容），再由 binder 從 bytes 而非 `req.Body` 綁定，避免 `req.Body` 被消費一次後無法再次讀取。
- 呼叫端（`c.ShouldBindWith()`）可根據情境選擇正確的介面，編譯器在型別層面卡位。

**程式碼位置**:
- 介面定義: [`binding/binding.go:32-49`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/binding/binding.go#L32)
- Binder 工廠（Content-Type 選擇）: [`binding/binding.go:93-120`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/binding/binding.go#L93) — `Default()`

**替代方案**:
- **單一萬用介面**（大多數 Web 框架）: `Bind(req, target)` 一個方法簽名處理所有類型。簡單但 URI binder 需忽略 `req`，body binder 需忽略 map。
- **Tag-based routing**（Spring Boot）: 用 annotation 參數 `@RequestBody`, `@RequestParam`, `@PathVariable` 分別指定來源。語法清晰但依賴反射。

**注意事項/何時不用**:
- `BindingUri` 不嵌入 `Binding`，導致 `BindingUri` 無法被 `Default()` 工廠回傳——這意味著 URI 綁定無法透過 Content-Type 自動選擇。
- 三組介面的區分對新手框架使用者不直覺——許多人第一次接觸 `c.ShouldBindUri()` 時不知道為什麼不能直接用 `c.ShouldBind()`。
- Form 綁定的邊界模糊：`formBinding` 讀 `req.Form`（包含 query + body），`formPostBinding` 只讀 `req.PostForm`（僅 body）。這個分別容易讓使用者困惑。

---

## Pattern 5: ReturnObj 模式隱藏介面實作差異

**是什麼**: `RouterGroup.returnObj()` 方法根據 `group.root` 旗標回傳不同物件：root group 回傳 `*Engine`，subgroup 回傳 `*RouterGroup`。因為兩者都實作 `IRoutes` 介面，呼叫端無需知道差異。

**為什麼有效**:
- 讓 method chaining 在不同層級一致運作: `engine.GET("/a", h).GET("/b", h)` 和 `group.GET("/a", h).GET("/b", h)` 都合法。
- Engine Embedding RouterGroup 已經讓 Engine 繼承了 RouterGroup 的所有方法，但因為 return type 不同（`*RouterGroup` vs `*Engine`），若沒有 `returnObj()`，從 Engine 呼叫 `GET()` 會回傳 `*RouterGroup`，破壞 chaining。

**程式碼位置**: [`routergroup.go:254-259`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/routergroup.go#L254)

**替代方案**:
- **Type assertion 模式**（其他 Go 框架）: 不回傳介面，而是每個方法回傳具體型別，由使用者手動型別斷言來續接 chain。
- **不使用介面，直接回傳 `*RouterGroup`**：但 Engine 的 `GET()` 回傳 `*RouterGroup` 會讓鏈接 `engine.GET("...", h).Use(mw)` 失敗（因為 `Use()` 最終回傳 `IRoutes`，但 `*Engine` 型別變成了 `*RouterGroup`）。

**注意事項/何時不用**:
- 多了 `interface{}`（`IRoutes`）的 dispatch overhead（雖然可忽略）。
- 對 IDE 和靜態分析工具不友善——工具無法確定 chain 的最終型別是 `*Engine` 還是 `*RouterGroup`。
- 這個模式對框架作者有用，但對框架使用者透明。

---

## Pattern 6: 渲染器（Render）用獨立 struct 而非設定開關

**是什麼**: 每種 JSON 變體（JSON, IndentedJSON, SecureJSON, JsonpJSON, AsciiJSON, PureJSON）都是獨立的 `Render` struct，而不是在同一個 renderer 上加 boolean flag。

**為什麼有效**:
- 避免 flag explosion：若將 6 種 JSON 變體合成一個 renderer，至少需要 5 個 bool（`indent`, `secure`, `jsonp`, `ascii`, `pureHTML`）——它們的交互組合（`indent && secure`？）會讓程式邏輯難以理解。
- 每個 struct 只需要實作 `Render()` 和 `WriteContentType()`，極簡。
- 新增一種 JSON 變體不需要修改既有程式碼（開閉原則）。

**程式碼位置**: 
- JSON 變體定義: [`render/json.go:57-194`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/render/json.go#L57) — 6 個 struct
- Render 介面: [`render/render.go:10-15`](https://github.com/gin-gonic/gin/blob/5f4f9643258dc2a65e684b63f12c8d543c936c67/render/render.go#L10)

**替代方案**:
- **Builder pattern**（其他框架）: `response.JSON().Secure().WithCallback("cb").Marshal(data)`。靈活但需要 builder 實作所有組合。
- **Options struct**（gRPC 的典型做法）: `JSONRenderer{Security: true, Prefix: "while(1);"}`。簡單但無法在編譯期約束互斥選項（如 `Secure && Indent`）。

**注意事項/何時不用**:
- 6 個 struct 的程式碼重複度較高（每個都包含類似的 marhsal + write 邏輯）。但這個重複是可接受的，因為每個 struct < 30 行，總共不到 200 行。
- 若未來 JSON 變體擴展到 15 種，可能應考慮使用策略模式（Strategy Pattern）而非獨立 struct。

---

## 整體設計品味的觀察

Gin 的設計風格可總結為「**效能優先，但不在 API 層面犧牲開發體驗**」。它的每個核心取捨都在強調這一點：

- 能用 slice 不用 linked list，能用 for loop 不用 recursion（`tree.go` 的 `getValue()` 是 iterative 的，但 `findCaseInsensitivePath()` 是 recursion——觀察到作者只在必要時才用遞迴）。
- 不追求 100% 通用性（不支援 regex route、不支援 async middleware），以換取簡單和速度。
- 全域 mutable state 的普遍存在（`Validator`、`EnableDecoderUseNumber`、`consoleColorMode`）反映框架誕生時期的實用主義——先能用，再談潔癖。
- Middleware chain 的設計明顯受 Express.js/Node.js 影響，但用 Go 的 goroutine 取代了 Node.js 的回呼（callback）模式。
- 「6 種 JSON renderer」與「不支援 regex route」形成鮮明對比——Gin 對常見需求提供豐富選項，但對複雜需求選擇不提供，引導使用者用自訂 middleware/handler 解決。
