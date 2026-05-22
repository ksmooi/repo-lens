---
repo: sqlalchemy/sqlalchemy
file: README
type: backend-framework
studied_at: 2026-05-22
commit_sha: 873f877
language: Python
framework: SQLAlchemy
stars: 11.9k
status: active
---

# SQLAlchemy · 概覽

Python 生態系中最成熟的 ORM 與 SQL 工具組，以「Core + ORM 雙層分離」的架構，同時滿足低階 SQL 操作與高階物件持久化的需求。

## 解決什麼問題

Python 應用需要與關聯式資料庫互動，但原生 DB-API 層級太低（直接寫 SQL 字串、手動管理連線、手動 mapping 結果），而 Django ORM 等框架又太綁定於特定 web 框架。SQLAlchemy 在中間建立了一個**框架無關的 SQL 抽象層**——Core 提供 SQL expression 與 engine 抽象，ORM 在此之上提供 Data Mapper 風格的物件持久化——讓開發者可以在不同抽象層級之間無痛切換。

## 為什麼值得研究

- **Core-ORM 雙層分離是 Python 生態獨有的設計**：多數 ORM（Django ORM、ActiveRecord）是 monolothic 設計，SQLAlchemy 的 Core/ORM 分離讓開發者可只取所需。這個取捨在 framework 設計中有很高的參考價值。
- **Data Mapper vs Active Record**：SQLAlchemy 採用 Data Mapper 模式（與 Hibernate/NHibernate 同族），而非 Django 的 Active Record。這個選擇影響 session 管理、transaction boundary、物件生命週期等一切設計。
- **Identity Map + Unit of Work**：Session 內實作了完整的 Identity Map 與 Unit of Work 模式，包括 dirty tracking、relationship cascade、flush order 的 topologial sort。
- **版本演進典範**：從 0.x→1.x→2.x（2025），SQLAlchemy 經歷了從「功能堆疊」到「API 收斂」的成熟路線，2.0 甚至做了 breaking change（移除 Query API 改為純 select()）。

## 技術棧一句話

`Python` + `DB-API` + `SQL Expression` + `Connection Pool`

## 健康度信號

- ⭐ Stars: ~11.9k
- 📅 最後 commit: 2026-05-20
- 👥 主要維護者: Mike Bayer（zzzeek）為核心，約 10+ 長期貢獻者
- 🔄 commit 頻率: 幾乎每日活躍
- 📦 最新版本: 2.1.0b3（2026）

## 入口檔案

- Core 入口: [`engine/create.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/engine/create.py) — `create_engine()` 工廠函式
- ORM 入口: [`orm/session.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/session.py) — `Session` 類別
- SQL Expression 入口: [`sql/expression.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/sql/expression.py) — `Select`、`ClauseElement` 等類別

## 我會在後續筆記中回答的問題

- SQLAlchemy 的 Core 與 ORM 之間的精確邊界在哪？為什麼這樣切？
- Session.flush() 的背後經過多少層抽象才到一條 INSERT？
- Declarative mapping 如何用元程式將 class 轉換為 Mapper 組態？
- Connection Pool 的 QueuePool 實作與常見坑是什麼？
- SQLAlchemy 的 Event 系統如何做到跨模組的低耦合？
- 2.0 style 為什麼用 `select()` 取代 `Query`？

## 比較：Python ORM 生態

| 面向 | SQLAlchemy | Django ORM | Peewee | SQLAlchemy 的 trade-off |
|---|---|---|---|---|
| 主要抽象 | Core + ORM 雙層 | 單層 ORM | 單層 ORM | 學習曲線陡，但靈活度最高 |
| 架構模式 | Data Mapper | Active Record | Active Record | Session 管理需要更小心，但可以處理複雜 mapping |
| 框架相依性 | 無（獨立） | 綁定 Django | 無 | 可與 Flask/FastAPI/純腳本混用 |
| Async 支援 | 自有的 async extension（ext/asyncio） | 3rd-party（channels）不完整 | aio-peewee | 非同步支援較原生，但需要理解 `sync_runner` 機制 |
| Lazy loading 機制 | 每屬性可配置 lazy/eager | select_related/prefetch_related | 簡易 | 精細控制但可能 N+1 |
| Schema 管理 | Alembic（獨立專案） | 內建 migration | 內建 migration | 分離設計但需額外學習 |
