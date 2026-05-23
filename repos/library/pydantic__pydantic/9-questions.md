---
repo: pydantic/pydantic
file: 9-questions
studied_at: 2026-05-23
commit_sha: 86f6bbf
---

# Pydantic · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 CoreSchema 不用 protobuf / flatbuffers 之類的序列化格式，而是自訂 JSON-like dict？**
  - 我目前的推測：[UNVERIFIED] CoreSchema 需要在 Python 層產出（GenerateSchema）後傳給 Rust 層。如果強制序列化為 protobuf，Python 側的開發者需要維護 proto 定義，增加複雜度。自訂 dict 格式讓 Python 側可以直接除錯（`print(core_schema)`），開發者體驗更好
  - 相關程式碼：`_internal/_generate_schema.py`、`pydantic-core/src/validators/mod.rs`

- [ ] **Plugin system 為何選擇 protocol-based 而非更常見的 middleware chain 模式？**
  - 我目前的推測：[UNVERIFIED] Protocol-based 讓 plugin 可以獨立在驗證過程中插 hook，而不需要了解整個 validator 的 dispatch chain。Middleware chain 如果 order 不正確，容易造成功能性 bug（例如先驗證還是先 logging）。Pydantic 的做法將 hook 點限制在「schema validator 創建時」的一個明確時間點
  - 相關程式碼：`plugin/__init__.py`、`plugin/_schema_validator.py`

- [ ] **pydantic-core 中 `chain.rs` 的 lazy/strict 切換策略是怎麼決定的？**
  - 我目前的推測：[UNVERIFIED] 可能基於「lax 模式先試，失敗後 strict 作為 fallback」的貪婪策略。但 `lax_or_strict` 的切換成本視型別而異 — 對 `int` 來說 lax=strict 幾乎相同（都是 int 或字串），對 `datetime` 來說 lax 接受的字串格式遠多於 strict
  - 相關程式碼：`pydantic-core/src/validators/chain.rs`、`pydantic-core/src/validators/lax_or_strict.rs`

- [ ] **TypeAdapter 跟 BaseModel 的 schema pipeline 之間的差異在哪？**
  - 我目前的推測：[UNVERIFIED] TypeAdapter 不走 metaclass，所以沒有 `ModelMetaclass.__new__` 的流程，而是直接在 `TypeAdapter.__init__` 中呼叫 `GenerateSchema`。這代表 schema generation 發生的時機不同（lazy，在 instance 創建時而非 class 定義時）
  - 相關程式碼：`type_adapter.py`、`main.py`（BaseModel）

## 想問維護者的問題

- 從 v1 的 decorator-based validator 到 v2 的 `Annotated[T, AfterValidator]`，這個 API 轉變的**最大驅動力**是什麼？效能、正確性、還是維護性？
- `pydantic-core` 的開發是怎麼測試的？Rust 側的型別轉換行為有沒有 formal specification 或 property-based test？
- 對 pydantic 的 plugin 開發者最常踩的坑是什麼？
- Python 3.14 `annotationlib` 的改動對 Pydantic 的影響有多大？有沒有考慮過直接 fork `annotationlib` 來相容舊版？

## 下次再看時的待辦

- [ ] 實際跑一次 benchmark：比較 v1 跟 v2 的 validation throughput（尤其在大 nested model 場景）
- [ ] 讀 `pydantic-core/src/validators/chain.rs` 的 lax/strict 切換實作
- [ ] 追蹤 `_generate_schema.py` 中最複雜的部分：`Annotated` metadata 疊加的處理順序（`AfterValidator` vs `Field` vs `__get_pydantic_core_schema__` 的優先級）
- [ ] 探索 pydantic plugin 的真實案例：Logfire 的 plugin 是怎樣包裝 validator 的
- [ ] 看 `tests/benchmarks/` 的 benchmark 結構

## 跨專案對照備忘

- **Hybrid Rust–Python 雙層架構** → 跟 [ruff](https://github.com/astral-sh/ruff) （astral-sh）的做法類似：Python 寫 CLI 和規則註冊，Rust 做核心檢查。Pydantic 跟 ruff 都選擇了「Python 層做 schema/rule 定義，Rust 層做執行」。**候選 pattern**：目前只在 pydantic 和 ruff（可能還有 uv）觀察到這種 hybrid 模式，值得追蹤是否在其他 Python library 重現

- **Metaclass-driven schema generation** → 跟 SQLAlchemy 的 `declarative_base` 類似：在 class 定義時收集 metadata 並編譯，後續 instance 使用預先編譯好的產物。差異在於 SQLAlchemy 用 descriptor（`Column`）而非 type hints。SQLAlchemy 的 declarative model class 也是用 metaclass 攔截定義過程。**候選 pattern**：Pydantic + SQLAlchemy 都用了 metaclass（或者 SQLAlchemy 用了類似 `__init_subclass__` 機制），可以抽成一個「class-definition-time compilation」pattern

- **`Annotated` 作為 metadata 載體** → 跟 FastAPI 的 `Annotated` 用法（`Annotated[str, Query(max_length=50)]`）同源。FastAPI 是 Pydantic 的使用者，但 FastAPI 對 `Annotated` 的使用已超出 Pydantic 的範圍 — 它不只是驗證，還用來注入路由資訊
