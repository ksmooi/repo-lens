---
repo: fastapi/fastapi
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 0b98630
---

# FastAPI · 值得偷學的設計

## Pattern 1: 編譯期依賴分析 (Compile-time Dependency Tree)

**是什麼**: FastAPI 不在每次請求時動態掃描 endpoint 參數型別，而是在路由**註冊時**（`APIRoute.__init__`）就呼叫 `get_dependant()` 建立完整的依賴樹。請求到達時，`solve_dependencies()` 只需沿著已建立的樹結構遞迴求解。

```mermaid
flowchart LR
    subgraph "Compile-time (route registration)"
        A[APIRoute.__init__] --> B[get_dependant(endpoint)]
        B --> C[依賴樹 Dependant 節點]
    end
    subgraph "Runtime (per request)"
        D[請求到達] --> E[solve_dependencies]
        E --> F{子依賴?}
        F -->|有| G[遞迴 solve_dependencies]
        F -->|無| H[request_params_to_args]
        G --> F
    end
```

**為什麼有效**: 編譯期分析將型別反射、ForwardRef 解析、Depends 遞迴展開等昂貴計算移到啟動階段。對一個有 50 條路由的應用來說，這可能節省每次請求 1-5ms 的型別解析時間。在高吞吐場景（10k+ QPS），這個差異累積起來很可觀。

**程式碼位置**:
- 編譯期: [`routing.py:948-965`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/routing.py#L948-L965)
- 執行期: [`dependencies/utils.py:598`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/dependencies/utils.py#L598)

**何時可以借用**: 任何需要在每個 request 做參數解析的框架或服務。關鍵前提是「參數的 type annotation 是靜態的」——不會因請求內容而變。若是動態 schema（如 GraphQL），這個 pattern 就不適用。

**注意事項**: 編譯期分析的依賴結構是靜態的。無法在請求層級動態改變依賴選擇（如根據請求 header 決定注入哪個 implementation）。若需要 runtime conditional injection，需搭配 factory pattern 或 `@functools.lru_cache` 等方式。

**替代方案**: Spring 使用 runtime proxy + reflection（啟動時掃描 bean，請求時動態注入）。Flask 沒有內建 DI，由第三方套件實作（如 Flask-Injector）。

---

## Pattern 2: Type Hints 作為 Single Source of Truth

**是什麼**: FastAPI 讓使用者**只寫一次型別**，框架從中推導出 validation、serialization、OpenAPI schema。參數在端點函式的簽名中定義一次，所有下游行為自動推導。

**為什麼有效**: 傳統上寫一個 REST endpoint 要寫三遍參數定義：(1) 函式簽名的型別，(2) 框架的 validation rules，(3) API documentation 的 schema specification。FastAPI 透過 Python 的 annotation 系統將這三者合一。這消除了「文件不更新」、「validation 沒改」等最常見的 API bug 來源。

**程式碼位置**: `param_functions.py` 定義了 `Query()`, `Path()`, `Body()`, `Header()`, `Cookie()`, `Form()`, `File()`, `Depends()`, `Security()` 等函式。每個參數函式都透過 `Annotated` 或 default value 攜帶 metadata，最終在 `analyze_param()` 中解析。

**何時可以借用**: 任何需要「從一種宣告產生多種產出」的系統。例如 CLI framework 可以從 function signature 推導 argument parser（Click 的 type hints branch）；設定管理可以從 dataclass 推導環境變數映射。

**注意事項**: Single Source of Truth 的前提是「truth 夠用」。如果型別系統無法表達的約束（如「創建時必填，更新時可選」）就需要額外宣告（如 `response_model_exclude_unset=True`）。當約束無法用型別表達時，這個 pattern 會開始出現 leaky abstraction。

**替代方案**: 
- **OpenAPI-first**: 先寫 OpenAPI yaml，再從中產生程式碼（如 Connexion、Fastify 的某些 plugin）。優點是 API 規格獨立於語言，缺點是型別安全性較低。
- **手動三遍**: Flask + marshmallow + flasgger 的經典組合。各自獨立但需要維護一致性。

---

## Pattern 3: Generator-as-Dependency 的 Resource Lifecycle

**是什麼**: FastAPI 允許 dependency function 使用 `yield` 而非 `return`，框架會自動管理 resource 的 acquire/release。Generator 的 cleanup 透過雙層 `AsyncExitStack` 在正確的生命週期時機執行。

```python
def get_db():
    db = Session()      # acquire
    try:
        yield db        # inject into endpoint
    finally:
        db.close()      # release (at request scope end)
```

**為什麼有效**: 這是 Python 語言層級 context manager 的模式（`__enter__`/`__exit__`）在 DI 系統中的應用。比起傳統的 `before_request`/`after_request` hook，它讓 resource lifecycle 跟 dependency 綁定，而不是跟 request lifecycle 綁定。一個 dependency 可以是「只活到 function 結束」（scope=function），也可以是「活到 request 結束」（scope=request）。

**程式碼位置**: [`dependencies/utils.py:666-676`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/dependencies/utils.py#L666-L676) — `_solve_generator()` 處理 generator 注入。

**何時可以借用**: 任何需要 resource 自動管理的框架層設計：資料庫 session、HTTP client connection pool、檔案 handle、lock 等。Generator 是 Python 表達 acquire/release 最自然的方式。

**注意事項**:
- Sync generator 透過 `run_in_threadpool()` 執行 — 不會在主 event loop 執行 cleanup
- Generator 內的 `try/finally` 必須存在，否則異常時 resource 不會釋放
- 巢狀 generator 的 cleanup 順序是反向的（inner stack 先 exit）
- 若 generator 中的 `yield` 被 skip（如 exception 發生在 endpoint 呼叫前），`finally` 仍會執行（Python generator 的 `close()` 機制保證）

**替代方案**: 
- **Flask's `teardown_request`**: 鬆耦合註冊，需要配對註冊與 cleanup
- **Django's `connection.on_commit`**: 僅處理 commit 後的 callback
- **ContextVar + middleware**: 手動管理 context 的 lifecycle

---

## Pattern 4: 委託式 Schema 生成 (OpenAPI Delegation)

**是什麼**: FastAPI 不做自己的 Python type → JSON Schema 轉換。它將所有 type hints 包裝成 Pydantic 的 `ModelField`，然後呼叫 Pydantic 的 `GenerateJsonSchema.generate_definitions()` 進行實際的 schema 生成。FastAPI 的工作是**組裝 OpenAPI 結構**（paths、parameters、responses），而不是**生成 JSON Schema**。

**為什麼有效**: 型別→schema 的轉換是一個極其複雜的問題：需要處理 Union、Optional、Annotated、Literal、ForwardRef、recursive models、generics、discriminated unions……Pydantic 在 v2 花了大量工程實作 `GenerateJsonSchema`，FastAPI 直接利用這個成果。這也讓 FastAPI 的使用者可以直接受益於 Pydantic 社群對 JSON Schema 支援的持續改進。

**程式碼位置**:
- [`openapi/utils.py:514-606`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/openapi/utils.py#L514-L606) — `get_openapi()` 編排整個流程
- [`_compat/v2.py`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/_compat/v2.py) — Pydantic v2 的 `GenerateJsonSchema` 調用封裝

**何時可以借用**: 當你的系統需要從某種 metadata（型別、宣告、schema）產生另一種輸出時，考慮找出該領域的「事實標準」工具，委託給它而非自己實作轉換。

**注意事項**: 委託的缺點是依賴關係更緊密。當 Pydantic v1→v2 時，FastAPI 需要透過 `_compat/` 層進行大量適應。此外，Pydantic 的 schema 行為變化會直接影響 FastAPI 的 OpenAPI 輸出。這是一個從「穩定遷移」到「跟上上游」的長期維護成本。

**替代方案**: 
- **自己實作 JSON Schema 生成**: 如 pydantic-core 出現前 Pydantic v1 的做法
- **使用 Marshmallow schema**: Flask 社群的常見選擇，schema 定義獨立於型別系統

---

## Pattern 5: 拍平依賴樹供 OpenAPI 使用

**是什麼**: FastAPI 維護一個完整的巢狀依賴樹（`get_dependant`），但也維護一個拍平版本（`get_flat_dependant`），遞迴將所有子節點的 path_params、query_params、body_params 等收集到單一 Dependant 物件。

```python
# dependencies/utils.py:138-189
def get_flat_dependant(
    dependant: Dependant,
    skip_repeats: bool = False,
) -> Dependant:
    # 遞迴收集所有子節點的參數，拍平到一個 Dependant 中
```

**為什麼有效**: 依賴樹是「執行用的結構」——巢狀遞迴，每個節點獨立求解。但 OpenAPI schema 需要「宣告用的結構」——所有參數在一個平面上，不關心誰依賴誰。拍平後，OpenAPI 生成器只需對一層參數做迭代，不需要理解依賴關係。

**程式碼位置**: [`dependencies/utils.py:138-189`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/dependencies/utils.py#L138-L189)

**何時可以借用**: 當你維護兩種不同用途的資料結構視角時（一個用於執行，一個用於展現/分析），考慮從核心結構衍生展開結構，而不是維護兩份獨立資料。

**注意事項**: 拍平時會遇到重複問題——兩層 dependency 都依賴同一個 `current_user`。`skip_repeats=True` 透過 `cache_key` 去重。

**替代方案**: 
- **執行時再取得**: 每次需要 schema 時重新掃描依賴樹，不做快取
- **維護兩個獨立清單**: 容易不同步

---

## Pattern 6: Dependency Override 測試模式

**是什麼**: FastAPI 提供 `app.dependency_overrides` dict，測試時可以注入 mock 取代 production dependency：

```python
def mock_get_db():
    yield test_session

app.dependency_overrides[get_db] = mock_get_db
```

這個機制不必使用 monkey-patch，也不必改動 production 程式碼。

**程式碼位置**: 在 `solve_dependencies()` 中檢查 override：

```python
# dependencies/utils.py:632-647
dependency_overrides_provider = getattr(
    request.scope, "dependency_overrides_provider", None
)
if dependency_overrides_provider:
    dependency_overrides = dependency_overrides_provider.dependency_overrides
    if sub_dependant.call in dependency_overrides:
        # 用 override 取代原 callable
```

**為什麼有效**: 比起 monkey-patch（改動 `module.function` 直到測試結束），dependency override 是框架層級的機制：type-safe、scope-clear（只影響當前 `app` 實例）、不依賴於 import 順序或模組引用。

**何時可以借用**: 任何提供 DI 的框架都應該考慮這個機制。Django 的 `settings override` 類似但僅限於設定，無法 override function。

**注意事項**: Override 是 callable-based，不是 type-based。不同模組的同名函式不會衝突，但**同一個函式物件只能有一個 override**。

---

## 整體設計品味的觀察

FastAPI 的設計風格可以總結為「**薄膠合層 + 委託給強者**」：

- HTTP 處理委託給 Starlette（ASGI、routing、request/response）
- Schema validation 委託給 Pydantic（type→schema、序列化、驗證）
- JSON encoding 委託給 Pydantic + 自訂有限例外
- CLI 委託給 `fastapi-cli`（獨立套件）

FastAPI 本身只做**兩件事**：把這些元件串起來（glue）和在串的過程中加入開發者體驗的糖（type hints-driven DI）。

這種設計在維護上有明顯取捨：每個 major dependency 的 breaking change 都需要 FastAPI 反應（`_compat/` 層的存在就是證據）。但對使用者來說，這套設計的好處遠大於代價——他們學的 Pydantic、Starlette 知識在 FastAPI 以外的 context 也能直接用。
