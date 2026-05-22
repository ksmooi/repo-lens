---
repo: fastapi/fastapi
file: 9-questions
studied_at: 2026-05-22
commit_sha: 0b98630
---

# FastAPI · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 `APIRouter` 不覆寫 Starlette 的 `add_route()` 方法來禁止使用？**
  - 目前使用者可以呼叫 `app.add_route("/", handler)`（Starlette 原生），繞過 FastAPI 的 DI 和 OpenAPI 系統。文件標 `deprecated`，但無法在編譯期阻止。
  - 相關程式碼: [`applications.py:84-93`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/applications.py#L84-L93)

- [ ] **為什麼 `jsonable_encoder()` 要自己維護一個完整的 `ENCODERS_BY_TYPE` dict，而不是完全依賴 Pydantic 的序列化？**
  - Pydantic 的 `model_dump(mode="json")` 已經處理了很多類型轉換，但 `jsonable_encoder` 仍保留自己的 encoder registry。是否為了歷史相容（Pydantic v1 時代遺留）？
  - 相關程式碼: [`encoders.py:84-112`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/encoders.py#L84-L112)

- [ ] **`sse.py` 的 KEEPALIVE 機制為何選擇 own implementation 而非 ASGI spec 的辦法？**
  - `EventSourceResponse` 自行實作 keepalive ping，但 ASGI 協定的 `http.response.body` 已經支援 chunked transfer encoding。
  - 相關程式碼: [`fastapi/sse.py`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/sse.py)

- [ ] **為什麼 `response_model` 的處理模式始終是 `"serialization"`（序列化），但在 schema 生成時卻用 `"validation"`（驗證）模式來建立 schema？**
  - 相關程式碼: [`routing.py:904-915`](https://github.com/fastapi/fastapi/blob/0b9863020dd9b3a237e971fa1e10afa0da280c9a/fastapi/routing.py#L904-L915)

## 想問維護者的問題

- 雙層 `AsyncExitStack` 在部分場景會被簡化為單層（如 routing 的 `self.dependant` 中 generator 很少會同時需要兩種 scope）。有考慮過 `scope` 參數的預設值從 `"request"` 改為 `"function"` 嗎？後者效能更好（內層 stack exit 更早），而且更符合直覺？
- `param_functions.py` 長達 2460 行，其中每個參數函式的 `Doc` 字串洋洋灑灑。這些文件字串在 IDE 的 hover popup 上看起來確實很棒，但維護成本很高。有考慮過用文件 generator 從測試文件同步這些字串嗎？
- `fastapi-slim` 套件的存在原因是為了減少雲端函式的部署大小嗎？能否詳細說明它跟 `fastapi[standard]` 在實務上的使用場景差異？

## 下次再看時的待辦

- [ ] 深入研究 `fastapi/_compat/v2.py` 中 Pydantic v2 的 `GenerateJsonSchema` 調用封裝，特別是 `get_definitions()` 怎麼處理 recursive models 和 discriminated unions。
- [ ] 理解 `fastapi-slim` 套件的 `pyproject.toml` 跟主套件的差異（位於 `fastapi-slim/` 目錄下，是一個獨立的 Python package）。
- [ ] 讀取 Starlette 的 `Router.__init__()` 跟 `ASGIApp` 介面，更精確理解 FastAPI 在哪些地方自訂了 Starlette 的行為。
- [ ] 跟其他 ASGI 框架（Litestar、Starlite）比較 middleware stack 建置方式。

## 跨專案對照備忘

- **編譯期依賴樹 + 執行期求解** 這個雙階段設計，在 microsoft/autogen 的 agent runtime 中也有類似的「註冊期建立 DAG，執行期遞迴求解」的模式。兩個 repo 都選擇了 compile-time/runtime 分離，雖然應用的領域完全不同（API framework vs agent framework）。→ **候選 pattern: 雙階段依賴解析 (Compile-time / Runtime Dependency Resolution)**
- **Generator-as-resource-lifecycle**（`yield db` → `finally: db.close()`）其實就是 context manager 作為 DI 的模式。在 `_patterns/` 中還沒有這個 pattern，但它出現在 FastAPI、Dependency Injector、和部分 Django plugin 中。需要至少觀察第三個獨立 repo 才能升級。
- **薄膠合層架構**（framework 本身只做 glue + DX，核心功能委託給專業套件）在 Python 生態中常見（如 Click 之於 Argparse、HTTPX 之於 requests），但在 framework 領域最顯著的就是 FastAPI + Starlette + Pydantic 的組合。Litestar 走了另一條路（自帶 type→schema 轉換），值得寫成比較。
