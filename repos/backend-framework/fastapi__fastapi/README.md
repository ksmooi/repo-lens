---
repo: fastapi/fastapi
file: README
studied_at: 2026-05-22
commit_sha: 0b98630
type: backend-framework
language: Python
framework: Starlette + Pydantic
stars: ~98.4k
status: active
---

# FastAPI · 概覽

FastAPI 是一個現代的 Python Web 框架，以 typing hints + Pydantic 為核心，自動產生 OpenAPI 文件。它的主要賣點不是功能本身——Flask 和 Django 也都做得到——而是**開發體驗**：寫一次型別，自動獲得 request validation、response serialization、OpenAPI schema、互動式 API 文件。

## 解決什麼問題

在 FastAPI 出現之前撰寫 Python HTTP API 主要有三條路徑：

- **Flask** — 靈活但手動。request parsing、validation、serialization 全都得自己來，或靠第三方套件拼裝。型別停留在裝飾，沒有編譯期或編輯器支援。
- **Django REST Framework** — 功能完整但厚重。ViewSet、Serializer、Permission、Router 等抽象層層疊疊，對一個簡單的 CRUD 需要學習大量概念。
- **Starlette** — 高效且 ASGI-native，但不提供 schema validation 或 API documentation。

FastAPI 把這三者的痛點一次解決：Flask 般的簡潔語法、DRF 般的完整功能、Starlette 般的效能——而且全部透過 Python typing hints 串起來。

## 為什麼值得研究

- **Type hints 作為「真相來源」** — FastAPI 是最早（也是最成功）把 Python typing hints 當作 API contract 的框架。參數 validation、序列化、OpenAPI schema 都從同一套型別推導，解決了「寫三遍」的痛苦。
- **編譯期依賴注入** — FastAPI 的 `Depends()` 機制在不同作中少見。它在路由註冊時就建立了完整的依賴樹（`get_dependant`），請求到達時只需遞迴求解（`solve_dependencies`），而非像 Flask 的 `before_request` 或 Spring 的 runtime proxy 那樣動態掃描。
- **雙層 AsyncExitStack** — Generator-based dependencies（如資料庫 session）的生命週期管理使用內外兩層 `AsyncExitStack`，分別對應 request scope 和 function scope，這在避免 resource leak 的同時保持了精確的清理時機。
- **純 Python 的 OpenAPI 生成** — FastAPI 不做自己的型別→schema 轉換，而是把工作委託給 Pydantic 的 `GenerateJsonSchema`，再組裝成 OpenAPI 結構。這是「站在巨人的肩膀上」的典範。

## 技術棧一句話

`Python` + `Starlette(ASGI)` + `Pydantic(schema)` + `uvicorn/server`

## 健康度信號

- ⭐ Stars: ~98.4k
- 📅 最後 commit: 2026-05-21
- 👥 主要維護者: Sebastián Ramírez (tiangolo) 為主，社群貢獻者數百人
- 🔄 commit 頻率: 每日活躍
- 📦 最新版本: 0.136.1

## 比較表格

| 面向 | FastAPI | Flask | Django REST Framework | Starlette |
|---|---|---|---|---|
| 主要抽象 | `FastAPI` app + `APIRouter` + `Depends` | `Flask` app + Blueprint + decorators | `ViewSet` + `Serializer` + `Router` | `Starlette` app + `Route` + `Middleware` |
| Request parsing | Type hints → Pydantic | 手動 (`request.form` / `request.json`) | 宣告式 (Serializer) | 不提供 |
| Response serialization | `response_model` + Pydantic | 手動 (`jsonify`) | `Serializer.data` | 不提供 |
| OpenAPI 產生 | 自動 | 第三方 (flasgger) | 自動 (coreapi / OpenAPI) | 不提供 |
| 非同步支援 | Native async (ASGI) | WSGI (可搭 gevent) | WSGI (Django 3.1+ ASGI) | Native async (ASGI) |
| DI 機制 | `Depends()` 編譯期分析 | 無內建 | 無內建 | 無內建 |
| 學習曲線 | 低—中 | 低 | 高 | 中 |
| **主要 trade-off** | 強依賴 Pydantic 生態，型別系統即 contract | 靈活但瑣事多，缺乏統一 validation | 功能完整但抽象層厚重 | 極簡但需自行補 validation / doc |

## 我會在後續筆記中回答的問題

- 為什麼 FastAPI 選擇繼承 Starlette 而不是直接依賴？這對架構有什麼影響？
- 編譯期依賴分析（`get_dependant`）跟 Spring 那種 runtime proxy 比，有什麼 trade-off？
- 雙層 AsyncExitStack 在 generator dependency 的生命週期管理上是怎麼設計的？
- FastAPI 的 type hints → OpenAPI schema 的完整轉換路徑是什麼？
- `response_model` 的序列化路徑跟手動 `jsonable_encoder` 差在哪？
