---
repo: pydantic/pydantic
type: library
category: library
api_style: declarative
studied_at: 2026-05-23
commit_sha: 86f6bbf
language: Python (Rust core)
stars: ~27.8k
status: active
---

# Pydantic · 概覽

## 解決什麼問題

讓 Python 的 type hints 不只是靜態型別檢查的工具，還能**在 runtime 執行資料驗證**。使用者用純 Python class 語法定義 schema，Pydantic 在實例化時自動完成型別轉換、格式校驗、自訂 validator 串聯。

## 為什麼值得研究

- **「type hints → runtime validation」的橋樑設計**：Pydantic 把 Python 的 annotation 系統從靜態檢查延伸到 runtime，這個突破性概念催生了 FastAPI、LangChain 生態
- **Python ↔ Rust 的雙層架構**：v2 用 Rust (PyO3) 實作效驗核心，Python 層只做 schema generation，是 hybrid 架構的經典案例
- **Plugin 系統與 extension protocol**：`__get_pydantic_core_schema__` 讓第三方型別無縫整合
- **破壞性相容策略**：v2 完全重寫，但內建完整的 v1 相容層 (`pydantic.v1`)，是 library migration 策略的教科書

## API 風格一句話

**Declarative + decorator-driven**。用 class annotation 定義 schema，用 `@field_validator` / `@model_validator` / `Annotated[type, AfterValidator(...)]` 擴充驗證行為。

## 技術棧一句話

`Python 3.10+` + `pydantic-core` (Rust / PyO3) + `typing-extensions` + `annotated-types` + `typing-inspection`

## 健康度信號

| 面向 | Pydantic | pydantic-core (Rust) | attrs / cattrs | marshmallow |
|---|---|---|---|---|
| 核心抽象 | `BaseModel` + type hints | — | `@attrs.define` | `Schema` class |
| 效能 | Python schema gen + Rust validation | Native Rust | Pure Python | Pure Python |
| 型別檢查支援 | mypy/pyright plugin | — | mypy plugin | 無 |
| 生態整合 | FastAPI、LangChain、SQLModel | — | 有限 | Flask-Marshmallow |
| 學習曲線 | 低（純 Python 語法） | 高（需懂 Rust） | 中 | 中高 |
| 主要 trade-off | Python←→Rust FFI 開銷 | 需獨立編譯 | 無 extra dep | 序列化較強但驗證較弱 |

## 我會在後續筆記中回答的問題

- Python type hints 到 Rust 驗證引擎的 schema pipeline 是怎麼走的？
- `BaseModel.__init__` 為什麼不用傳統的 `__init__`，而用 metaclass 合成的？
- Pydantic 的 plugin 系統比起 middleware / hook 型 library 有什麼不一樣的設計選擇？
- 從 v1 到 v2 的破壞性變更怎麼管理？`pydantic.v1` 相容層的代價？
