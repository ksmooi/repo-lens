---
repo: sqlalchemy/sqlalchemy
file: 9-questions
studied_at: 2026-05-22
commit_sha: 873f877
---

# SQLAlchemy · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 Connection 和 Session 各自實作了自己的 event 分派，而不是統一由 Event Manager 管理？**
  - 我目前的推測: [UNVERIFIED] 可能是因為 Connection 和 Session 的生命週期與 event 類型差異太大——Connection 的 event 是 SQL 執行層級（`before_execute`），Session 是 ORM 生命週期層級（`before_flush`）。但也許可以用更統一的方式管理。
  - 相關程式碼: [`engine/base.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/engine/base.py) 的 event dispatcher / [`orm/session.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/session.py) 的 event dispatcher

- [ ] **為什麼 Unit of Work 的 topological sort 是用自製的實作而非 networkx 等套件？**
  - 我目前的推測: [UNVERIFIED] 避免外部依賴？但也許是因為 topo sort 的需求簡單（dag，節點數通常 < 100），用自製的實作更輕量。
  - 相關程式碼: [`util/topological.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/util/topological.py)

- [ ] **2.0 改版選擇保留舊版 Query API 這麼多年（1.4→2.0→2.1 beta 目前 still 有 Query class），遷移策略是什麼？**
  - 我目前的推測: [UNVERIFIED] 2.0 的改版策略很聰明——1.4 先讓 `select()` 可用（但同時保留 Query），2.0 讓 `select()` 成為預設推薦，但 Query 仍然可用。SQLAlchemy 一直對 backward compatibility 非常重視（即使 ORM 是出了名的複雜），這可能是刻意放慢 migration 步調。
  - 相關程式碼: [`orm/query.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/query.py) 依然存在且可使用

- [ ] **Pool 的 `_ConnectionFairy` proxy 機制是否影響 GC 行為？**
  - 我目前的推測: [UNVERIFIED] Pool 回傳的 `_ConnectionFairy` 是弱引用管理，connection 物件不會被意外 GC 掉。但這個 proxy layer 在極端高併發場景下可能增加 overhead。
  - 相關程式碼: [`pool/base.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/pool/base.py) `_ConnectionFairy` 類別

- [ ] **Async 支援為什麼選擇 greenlet 而非直接 async-ify 整個 codebase？**
  - 我目前的推測: [UNIVERSIFIED] 因為 SQLAlchemy 的核心（engine、pool、dialects）跟 DB-API（同步）綁定太深。全部改 async 意味著重寫 engine/base.py、pool/base.py、所有 dialects、所有 compiler。這是不可能的任務。greenlet 方案是務實的 trade-off——只加一層薄薄的 async wrapper，讓同步的核心在 greenlet 內跑。

## 想問維護者的問題

- **如果把 SQLAlchemy 從頭來過，你會維持 Core/ORM 分離還是做單層？是什麼讓你決定維持這個取捨？**
- **2.0 style 的 `select()` 統一 Core 和 ORM 的 query 介面，但很多地方還是有 Core/ORM 的型別分裂（例如 `Column` vs `Mapped`）。有沒有計劃進一步統一？**
- **為何不用 Python 原生 `__init_subclass__` 取代 metaclass (`DeclarativeMeta`)？兩者在這情境下的優缺點是什麼？**

## 下次再看時的待辦

- [ ] 深入研究 `sql/compiler.py` 的 visitor 機制——特別是在 `visit_select()` 與 `visit_whereclause()` 之間如何傳遞 context
- [ ] 分析 `orm/strategies.py` 中的 lazy/eager/joined/subquery loading 策略，比較它們的 SQL 生成與效能特徵
- [ ] 深入 `orm/dependency.py` 與 `orm/persistence.py` 的 flush pipeline——特別是如何處理 circular dependency
- [ ] 比較幾個主要 dialect（sqlite、postgresql、mysql）的 compiler override 差異
- [ ] 看 `util/preloaded.py` 的 lazy import 機制——SQLAlchemy 的 import time 很長，這是怎麼管理的

## 跨專案對照備忘

- **Visitor-based SQL compilation**: 跟 `llvm-project` 中的 `Visitor` pattern 類似（`clang/AST/RecursiveASTVisitor.h`）——都是透過 visitor 將內部的 AST/expression tree 轉換成目標語言或底層表示。值得跟其他 Python SQL toolkit（如 `pypika`、`sqlglot`）做比較。
  - **候選 pattern**: 如果另外兩個 repo 也有類似的「expression tree -> dialect-aware output」系統，可以抽成 `expression-tree-compilation.md`

- **Identity Map + Unit of Work**: 跟 JPA/Hibernate 的實作非常相似。Hibernate 的 Session 也有 Identity Map、dirty checking、flush on commit。SQLAlchemy 在 Python 生態中的對應 pattern 值得記錄。
  - **候選 pattern**: 如果另外兩個 Python ORM（如 `ponyorm`、`peewee`）也有 Identity Map，可以寫 `identity-map-unit-of-work.md`

- **Event dispatch over subclassing**: SQLAlchemy 的 event system 策略跟 Microsoft Entity Framework 的 `DbContext.OnModelCreating` 思路相反——Entity Framework 選擇留 override hooks（subclassing），SQLAlchemy 選擇 event（pub-sub）。兩者都是常見的框架擴充策略，值得比較。
