---
repo: pydantic/pydantic
file: 3-key-patterns
studied_at: 2026-05-23
commit_sha: 86f6bbf
---

# Pydantic · 值得偷學的設計

## Pattern 1: Hybrid Rust–Python 雙層架構

**是什麼**：把 library 的核心效驗邏輯用 Rust 實作（pydantic-core），Python 層只做 schema 產生與配置管理，兩層之間透過 PyO3 FFI 以 `CoreSchema`（一種 JSON-like IR）溝通。

**為什麼有效**：
- Python 的 type hint introspection 是 Python 擅長的事（動態反射），Rust 的類型安全驗證是 Rust 擅長的事（模式比對、錯誤收集、效能）
- 兩層的依賴方不同：`pydantic-core` 只依賴 Rust 標準庫 + PyO3，`pydantic` 只依賴 Python typing 工具。各自可以獨立開發與測試
- 建置階段（schema generation）的代價只發生在 class 定義時一次，後續的 instance 建立都只需跟 Rust 層互動，沒有 Python interpreter 的 overhead

**程式碼位置**：
- Python schema generator：[`_internal/_generate_schema.py`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic/_internal/_generate_schema.py)
- Rust validator engine：[`pydantic-core/src/validators/mod.rs`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic-core/src/validators/mod.rs)
- FFI 綁定（PyO3）：[`pydantic-core/src/lib.rs`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic-core/src/lib.rs)

**替代方案**：
- **Pure Python**：像 v1 或 marshmallow 全部在 Python 層做驗證，簡單但慢一個數量級
- **Pure Rust**：像 grex 或 serde 在 Rust 側寫 DSL macro，使用者體驗差，無法利用 Python typing
- **C extension**：寫 C Python extension，沒有 Rust 的模式比對和 memory safety

**何時可以借用**：當你的 library 有「建置期 → 執行期」的明顯分離，且執行期是效能瓶頸。Rust 層適合處理 pattern matching 密集的任務（像 validation 就是大量類型判斷）。

**注意事項**：
- **FFI 開銷**：每次驗證結果從 Rust 回傳到 Python 有 boxing 成本，雖低但不能忽略
- **雙版本管理**：pydantic 跟 pydantic-core 的版本必須同步（`pydantic-core==2.47.0` 是 hard pin），發布流程複雜
- **除錯困難**：當驗證邏輯有 bug，Python traceback 會穿過 FFI，堆疊資訊多半在 Rust 側丟失

---

## Pattern 2: Metaclass-Driven Schema Generation

**是什麼**：不是在 instance 建立時、而是在 **class 定義時**（metaclass `__new__`）就完成所有 schema 的生成與編譯。`BaseModel` 的每個 subclass 在宣告完 `id: int` 的瞬間就已經編譯好對應的 Rust validator。

**為什麼有效**：
- 跟 Pattern 1 互補 — 編譯成本被推到建置期，執行期零 overhead
- Python 的 metaclass 提供「類別定義生命週期鉤子」，剛好在 annotation 完整宣告完、instance 尚未建立的中間點介入
- 每次「建置」的結果（`__pydantic_core_schema__`、`__pydantic_validator__`、`__pydantic_serializer__`）被 cache 在 class 層級，所有 instance 共享

**程式碼位置**：[`_internal/_model_construction.py:88-200`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic/_internal/_model_construction.py#L88-L200)

**替代方案**：
- **Decorator-based**：像 `@dataclass` 用 decorator 包起來，但缺少 metaclass 的自動攔截能力
- **`__init_subclass__`**：Python 3.6+ 的替代方案，但無法接管 `__init__` 的簽名合成
- **Class decorator**：需要使用者明確包裹，忘記就沒有效果

**何時可以借用**：當你的框架需要在「使用者定義 class」時做大量靜態分析或預編譯。適用的訊號是：你的 library 永遠需要在 instance 建立前知道 class 的完整結構。

**注意事項**：
- Metaclass 衝突：如果你的使用者同時用了另一個 metaclass（如 ABCMeta），需要使用 `ABCMeta` 作為基底繼承（Pydantic 就是這麼做的）
- IDE 相容性：metaclass 合成的 `__init__` 簽名在 IDE 中可能不正確，需要 mypy/pyright plugin 輔助
- 除錯複雜度：metaclass __new__ 中的錯誤會讓 class 定義本身失敗，堆疊資訊不直觀

---

## Pattern 3: `Annotated` as Extensible Validation Metadata

**是什麼**：利用 Python 的 `Annotated[T, ...]` 作為驗證 metadata 的載體。使用者在型別宣告時用 `Annotated[int, AfterValidator(...), Field(ge=0)]` 附加驗證與序列化行為，不需要定義新的 subclass 或 decorator。

**為什麼有效**：
- `Annotated` 是 Python 3.9+ 標準庫的一部分，IDE 和 type checker 都支援
- 與 type hint 的自然疊加：`int` 是核心型別，`AfterValidator(...)` 是行為修飾，兩者透過 `Annotated` 保持在一起
- Pydantic 的 `__get_pydantic_core_schema__` protocol 讓第三方型別也可以用同樣機制擴充

**程式碼位置**：
- 內建 validators：[`functional_validators.py:30`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic/functional_validators.py#L30) (AfterValidator, BeforeValidator, WrapValidator, PlainValidator)
- Annotated 解析邏輯：[`_generate_schema.py`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic/_internal/_generate_schema.py) (遍歷 `get_args(type_)` 找 Annotated metadata)

**替代方案**：
- **Decorator 模式**（v1 的做法）：`@validator('field')` 跟 field 宣告分離，跟讀困難
- **Field() 參數**：`Field(gt=0)` 是另一種方式，但只能用在 model field 上，不能用在 `TypeAdapter` 或任意型別
- **Subclass 模式**：每個被驗證的型別都定義新 class，組合爆炸

**何時可以借用**：當你設計一個 type-based library，使用者的型別需要附加行為（驗證、序列化、標記），而且這些行為跟型別本身邏輯相關。

**注意事項**：
- `Annotated` 的 metadata 順序重要：Pydantic 的 processing order 是 metadata 被反覆運算的順序（左到右），使用者必須理解這個順序才能正確使用
- IDE 在 `Annotated` 中的型別推導有時會報 `Any`，需要額外 type stub
- Python 3.12 後 `Annotated` 的行為有微調（支援直接 unpack）

---

## Pattern 4: Lazy Dynamic Import System

**是什麼**：Pydantic 的 `__init__.py` 不直接 import 所有公開 API，而是用一個 `_dynamic_imports` dict + `__getattr__` hook 實現**延遲載入**（lazy loading）：只有當使用者真的存取某個名稱時，才載入對應的模組。

```python
_dynamic_imports = {
    'BaseModel': (__spec__.parent, '.main'),
    'Field': (__spec__.parent, '.fields'),
    'TypeAdapter': (__spec__.parent, '.type_adapter'),
    ...
}

def __getattr__(attr_name):
    dynamic_attr = _dynamic_imports.get(attr_name)
    module = import_module(module_name, package=package)
    result = getattr(module, attr_name)
    # Cache result in globals for subsequent access
    globals()[attr_name] = result
    return result
```

**為什麼有效**：
- 減少 import 時間：Pydantic 有大量模組（30+），其中有些只在邊緣情況用到（如 `color.py`、`networks.py`）。使用者 import `pydantic` 時只載入觸及到的 API
- 循環 import 避免：`BaseModel` import `_internal` 模組，`_internal` 模組也可能需要 `BaseModel`。Lazy loading 把實際載入推到所有模組都建立完之後
- 與 TYPE_CHECKING 搭配：在 `TYPE_CHECKING` 區塊預先 import 以維持 IDE 支援，runtime 則不執行

**程式碼位置**：[`pydantic/__init__.py:248-452`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic/__init__.py#L248-L452)

**替代方案**：
- **Explicit `__all__` + direct import**：import tree 簡單時更清晰
- **`importlib.import_module` 自行管理**：沒有 cache 機制的 lazy import 多次存取會重複 import
- **__import__() 魔術**：Python 內建但不可靠

**何時可以借用**：當你的 library 有大量公開 API（100+）、import time 顯著、或存在循環 import。

**注意事項**：
- `__getattr__` 對 `from pydantic import X` 完全支援，但 **`import pydantic.X`** 不支援
- cache 機制：Pydantic 的實作在第一次存取後將結果寫入 `globals()`，後續存取跳過 import
- deprecation 整合：被 deprecated 的 import 在 `__getattr__` 中發出 `PydanticDeprecatedSince*` warning

---

## Pattern 5: Versioned Deprecation Warnings

**是什麼**：Pydantic 定義了多個 deprecation warning 類別（`PydanticDeprecatedSince20`、`PydanticDeprecatedSince26`、...、`PydanticDeprecatedSince212`），對應不同版本引入的棄用。使用者可以根據自己所需的最小版本抑制警告。

```python
class PydanticDeprecatedSince20(PydanticDeprecationWarning):
    ...

class PydanticDeprecatedSince26(PydanticDeprecationWarning):
    ...

class PydanticDeprecatedSince210(PydanticDeprecationWarning):
    ...
```

**為什麼有效**：
- 使用者可以精確控制：如果你還在用 pydantic 2.9，你可以 `filterwarnings('ignore', category=PydanticDeprecatedSince210)`，不需要抑制所有 deprecated
- Library 維護者可以追蹤版本：一個 deprecated item 在哪些版本後應該刪除一目瞭然
- 跟 Python 的 `warnings.filters` 相容，無需自訂過濾機制

**程式碼位置**：[`pydantic/warnings.py`](https://github.com/pydantic/pydantic/blob/86f6bbf/pydantic/warnings.py)

**替代方案**：
- **單一 DeprecationWarning**：簡單但無法精細控制
- **changelog-only deprecation**：只在 Release Notes 寫「X 被棄用」，不在 runtime 提示，使用者很容易忽略
- **SemVer 強硬派**：breaking change 只發生在 major version，不提供 deprecation window

**何時可以借用**：任何有長期維護計畫的 library（尤其是 2+ 年的 minor releases），且使用者的版本分佈很廣。

**注意事項**：
- 版本類別名稱會隨時間累積。如果持續到 v2.20、v2.30，類別清單會很長。Pydantic 目前已經到 `PydanticDeprecatedSince212`
- 使用者仍然可能因為混合抑制而錯過重要警告
- 對大量 deprecated items 的 library，單一版本類別不夠精細 — Pydantic 的做法是**只按版本劃分**，不按功能

---

## API 設計品味的觀察

Pydantic 的 API 設計展現了幾種品味偏好：

1. **偏好 Annotation 而非 Decorator**：v2 中 `Annotated[int, AfterValidator(...)]` 是推薦方式，`@field_validator` decorator 是 v1 的遺留。Annotation 更貼近 type hints 的聲明式精神
2. **單一 ConfigDict 取代多個 class attribute**：v1 用 `class Config:` 子類別定義多個 attribute；v2 用單一個 `model_config = ConfigDict(...)` dict，更一致、更容易動態設定
3. **Protocol 而非 ABC**：Pydantic 的 plugin 系統和 `__get_pydantic_core_schema__` 都基於 `Protocol`，避免繼承強耦合
4. **對 Python 版本的誠實**：在 `__init__.py` 和 `_model_construction.py` 中多次出現 `if sys.version_info >= (3, 14)` 這類版本分支。不試圖用相容層掩蓋版本差異，而是直接處理

## 對相容性的態度

從 v1 → v2 的 migration 策略可以看出 Pydantic 團隊的相容性哲學：

- **內建 v1 相容層**：將完整的 v1 原始碼 snapshot 到 `pydantic/v1/` 子套件，使用者只需 `from pydantic import v1` 即可逐步遷移。這比維護兩條平行的 pip package 版本（如 `pydantic-v1`）更簡單，但增加了 ~30 個 extra 模組
- **Deprecation Windows**：引入 `PydanticDeprecatedSince*` 版本階梯式警告，使用者知道「這個在 2.x 尚可用，2.y 後移除」
- **對「醜但相容」的容忍**：v2 保留了 `validator`、`root_validator` 這些 v1 的 deprecated API（需從 `pydantic.deprecated` 導入），代價是維護 v1 風格的 `_decorators_v1.py` 與 `deprecated/` 資料夾
