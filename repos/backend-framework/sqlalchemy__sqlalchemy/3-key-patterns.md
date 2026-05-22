---
repo: sqlalchemy/sqlalchemy
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 873f877
---

# SQLAlchemy · 值得偷學的設計

## Pattern 1: Visitor-based SQL Compilation

**是什麼**:
SQLAlchemy 用 Visitor Pattern 將 ClauseElement 物件樹編譯為 SQL 字串。每個 ClauseElement 子類別（`Select`、`WhereClause`、`BinaryExpression` 等）實作 `_compiler_dispatch()`，Compiler 透過 `visit_xxx()` 方法遍歷整棵樹。

**為什麼有效**:
傳統的字串拼接方式難以處理條件組合、參數綁定、dialect 差異。Visitor Pattern 把「資料結構」和「操作」分離——ClauseElement 樹只描述「想做什麼查詢」，Compiler 決定「在目標資料庫上怎麼實現這個查詢」。

**程式碼位置**:
- [`sql/compiler.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/sql/compiler.py) — 核心 compiler，含 `visit_select()` 等 visitor 方法
- [`sql/visitors.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/sql/visitors.py) — Visitor 基礎架構
- [`sql/expression.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/sql/expression.py) — ClauseElement 類別定義

**何時可以借用**:
當你有需要把「結構化資料（AST / DSL / expression tree）」轉換成「另一種格式（SQL / GraphQL / 其他字串語言）」時，Visitor Pattern 是標準解法。常見的應用場景：SQL query builder、form validation tree、rule engine。

**替代方案**:
- **String interpolation**: 簡單但無法支援 dialect 切換與參數 injection 防護
- **Template-based**: Jinja SQL templates 更直覺但缺乏型別安全與 cache 支援
- **Code generation**: 如 Prisma 的 generator，透過 codegen 產生型別安全的 query。靈活度較低但編譯期檢查更強

**注意事項**:
- Visitor 遍歷的效能開銷在 query 數量少時不明顯，但 batch 場景下 compiler cache 至關重要
- 新增新的 ClauseElement 類型需要同時新增 visitor 方法，否則會報 `NotImplementedError`

---

## Pattern 2: Identity Map + Unit of Work

**是什麼**:
Session 同時實作了 Identity Map 與 Unit of Work 兩個企業級 persistence pattern：
- **Identity Map**: 每個 PK 在同一 Session 中只對應一個 Python 物件實例 (`orm/identity.py:37`)
- **Unit of Work**: Session 追蹤物件的 dirty/new/deleted 狀態，在 flush() 時一次性寫入 (`orm/unitofwork.py`)

**為什麼有效**:
Identity Map 避免了同一個 row 存在兩個不一致的 Python 物件。Unit of Work 確保所有變更改在一個 transaction 邊界內寫入，避免 partial write 問題。

**程式碼位置**:
- [`orm/identity.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/identity.py) — `IdentityMap` 類別（37 行開始）
- [`orm/unitofwork.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/unitofwork.py) — `UOWTransaction` 類別
- [`orm/session.py:4488`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/session.py#L4488) — `_flush()` 方法，調用 UOWTransaction

**何時可以借用**:
任何需要 ORM 支援的應用。Identity Map 模式對 Entity Framework、Hibernate 使用者很熟悉。如果你在設計自己的 persistence 層，這兩個 pattern 幾乎是必備的。

**替代方案**:
- **Active Record (Django ORM, Rails ActiveRecord)**: 每個 Model 物件自己負責 CRUD，沒有 Identity Map。優點是簡單直覺，缺點是 session scope 不明確、難以做 batch update。
- **簡單的 data mapper without identity map**: 每次都 new 新物件。簡單但無法做 dirty tracking，且無法保證同一 row 的一致性。

**注意事項**:
- Identity Map 的 scope 就是 Session 的生命週期。如果你讓 Session 活太久，Identity Map 會累積大量物件導致記憶體問題
- Unit of Work 的 dependency sorting 在 deep relationship chain 時可能很慢（topological sort 的 worst-case 階層）

---

## Pattern 3: Cache Key for Query Compilation

**是什麼**:
SQLAlchemy 為每個編譯過的 ClauseElement 樹計算 cache key — 一個 hashable 的結構（tuple），代表 query 的「靜態結構」。如果相同 cache key 的 query 再次出現，compiler 直接回傳快取的 Compiled object，跳過 compilation。

**為什麼有效**:
Query compilation 需要遍歷 ClauseElement 樹、格式化 SQL、處理參數綁定。對高吞吐量的應用來說，這個 overhead 可觀。Cache key 機制讓「相同結構但不同參數」的 query 只 compilation 一次。

**程式碼位置**:
- [`sql/cache_key.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/sql/cache_key.py) — cache key 計算核心（`CacheKey` class）
- [`sql/compiler.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/sql/compiler.py) — 使用 cache key 的地方

**何時可以借用**:
當你處理「結構重複但參數不同」的工作負載時——例如在同一個 request 週期內多次查詢不同 ID 的資料、或 template 渲染引擎的 compilation cache。

**替代方案**:
- **LRU cache on compiled SQL strings**: 更簡單，但無法區分「結構相同、參數不同」的情況（因為參數值已嵌入 SQL）
- **No cache**: 永遠重新編譯。簡單但浪費 CPU
- **Prepared statements (DB layer)**: DB 端也有 cache，但無法處理 dialect 切換

**注意事項**:
- Cache key 必須忽略參數值。如果參數值也被編入 cache key，那就失去 cache 的意義了
- 當 Statement 使用 lambda expression（`sql/lambdas.py`）時，cache key 的計算更複雜，因為 lambda 的 implementation may change

---

## Pattern 4: Declarative Mapping via Metaclass

**是什麼**:
SQLAlchemy 的 declarative extension 使用 metaclass（`DeclarativeMeta`）攔截 class 建立過程，將 declarative class 的屬性宣告（`Column`、`relationship`、`mapped_column`）自動轉化為 Mapper 配置。

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

# 背後實質上等同於:
# mapper_registry.mapped(User)
# Mapper(User, Table("users", ...))
```

**為什麼有效**:
Metaclass 讓使用者的 class 宣告既是 Python class 定義，也是 Mapper 配置描述。不需要額外的 DSL 或 config 檔案——這是 Pythonic 風格的最佳示範。

**程式碼位置**:
- [`orm/decl_base.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/decl_base.py) — `DeclarativeMeta` metaclass 與 `Mapper` 生成邏輯
- [`orm/decl_api.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/orm/decl_api.py) — 高層 API（`declarative_base()`、`registry.mapped()`）
- [`ext/declarative/__init__.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/ext/declarative/__init__.py) — 舊版 declarative extension（2.0 後推薦用 `orm.declarative_base`）

**何時可以借用**:
任何需要在 class 建立時捕捉組態的場景——ORM mapping、serialization schema、DI container、設定驗證。

**替代方案**:
- **Decorator approach (Pydantic, attrs)**: 用 `@dataclass`/`@define` 裝飾器，class 定義更純淨，但無法在 class body 執行期間捕捉屬性宣告
- **Explicit DSL**: 如 SQLAlchemy 的 `Mapper` 指令式 mapping，更顯式但更冗長
- **Code generation**: 從 schema 文件產生 class，型別安全但工作流程複雜

**注意事項**:
- Metaclass 的副作用是隱式的。class 一被定義，Mapper 就註冊了。這在某些場景（如 test isolation）可能造成問題，需要 `clear_mappers()`
- 從舊版 `ext/declarative` 到新版 `orm.declarative_base` 的遷移需要小心

---

## Pattern 5: Event Dispatch as Cross-Cutting Concern

**是什麼**:
SQLAlchemy 的 Event 系統 (`event/`) 是 framework 內建的 pub-sub 機制，讓 Core 和 ORM 的內部生命週期可以對第三方 observer 開放。從 Engine 的 `before_execute`、`after_execute` 到 Session 的 `before_flush`、`after_flush`，幾乎所有重要節點都有對應的 event hook。

**為什麼有效**:
Event system 讓 SQLAlchemy 在不解耦的前提下提供擴充點。例如 ORM 的 `before_flush` event 讓開發者在 flush 前攔截 dirty objects——這比繼承 Session 或 monkey-patch 都乾淨。

**程式碼位置**:
- [`event/api.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/event/api.py) — `listen()`、`listens_for()` API
- [`event/base.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/event/base.py) — `dispatcher` 與 `Events` 類別
- [`event/registry.py`](https://github.com/sqlalchemy/sqlalchemy/blob/873f877/lib/sqlalchemy/event/registry.py) — listener 註冊管理

**何時可以借用**:
當你的 framework 需要提供多樣化的擴充點，但又不想因此暴露出複雜的 subclass 介面時。

**替代方案**:
- **Subclassing**: 使用者繼承 `Session` 並覆蓋方法。優點是型別安全，缺點是每個功能都需要開一個 hook。SQLAlchemy 也有 subclass hook（如 `Session.get_bind()` 可覆蓋）
- **Mixins / Adapters**: 用 composition 模式。更彈性但使用較不便
- **Middleware / Pipeline**: Flask 與 Starlette 等 web framework 採用的 pipeline 模型。適用於 HTTP request/response 類別的情境，但不適用於 ORM session 的 object-level lifecycle

**注意事項**:
- Event listener 的執行順序是註冊順序，不是依賴順序。如果 A 事件 listener 依賴 B 事件 listener 先執行，需要自行管理
- 過多 listener 影響效能。每個 event dispatch 都遍歷 listener 清單
- Event listener 內的 exception 可能導致難以預測的狀態

---

## 整體設計品味的觀察

SQLAlchemy 的作者 Mike Bayer 偏好「**顯式但不強制**」的設計風格。

**顯式**: Session 管理、transaction boundary、flush timing 都是顯式的。不像 Django ORM 會默默 flush，SQLAlchemy 讓開發者知道每個 DB 操作何時發生。

**但不強制**: 幾乎所有東西都可客製化——你可以覆蓋 `Session.get_bind()`、自訂 `TypeDecorator`、寫自己的 `Dialect`、用 Event 攔截任何內部操作。框架不綁住你的手，但要求你知道自己在做什麼。

程式碼風格上，SQLAlchemy 有幾個值得注意的傾向：

1. **超大的單檔**：`engine/base.py` 3355 行、`orm/session.py` 5416 行、`sql/compiler.py` 不知多大。這打破了「檔案不要超過 500 行」的普遍建議，但在 SQLAlchemy 中，`Connection` 與 `Session` 這類核心類別的所有行為在同一檔案內讓 traceability 更好。

2. **大量的 type annotations**：2.0 版本大幅增加了 typing 支援（`TypedReturnsRows`、`TypeVarTuple`、`Unpack`），讓 IDE 可以推導 `session.execute(select(...))` 的回傳型別。

3. **`_` 前綴的私有方法層層遞迴**：`execute()` -> `_execute_internal()` -> `_execute_on_connection()`，每一層都嚴格區分 public API 與 internal implementation。
