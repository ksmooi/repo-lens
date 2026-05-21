---
repo: HKUDS/RAG-Anything
file: 3-key-patterns
studied_at: 2026-05-21
commit_sha: e8c0081
---

# RAG-Anything · 值得偷學的設計

## Pattern 1: Mixin-based Dataclass 組合

**是什麼**: RAG-Anything 的主類別使用 `@dataclass` + 三個 mixin 的組合方式，而非傳統繼承。

```python
@dataclass
class RAGAnything(QueryMixin, ProcessorMixin, BatchMixin):
    lightrag: Optional[LightRAG] = field(default=None)
    llm_model_func: Optional[Callable] = field(default=None)
    ...
```

每個 mixin（`ProcessorMixin`、`QueryMixin`、`BatchMixin`）在自己的檔案中獨立定義，只透過 type hints 宣告它需要的屬性（`config`、`logger`、`lightrag` 等），而不繼承這些屬性。

**為什麼有效**:
- 三個職責（文件處理、查詢、批次處理）的程式碼各自集中在一個檔案，而不是散落在一個大類中
- 新增一個 mixin 不需要修改既有 mixin 的程式碼
- 每個 mixin 可以獨立測試

**程式碼位置**:
- 主類別：[`raganything.py:50-643`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/raganything.py#L50-L643)
- ProcessorMixin：[`processor.py:30-2252`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/processor.py#L30)
- QueryMixin：[`query.py:23-868`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/query.py#L23)
- BatchMixin：[`batch.py:19-428`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/batch.py#L19)

**何時可以借用**:
- 你的類別有 2-4 個職責，每個職責需要 200+ 行程式碼
- 這些職責之間共用少量屬性（config、logger、lightrag），沒有複雜的相依關係
- 你希望每個職責可以獨立維護

**替代方案**:
- **傳統繼承**：`RAGAnythingBase → ProcessorQueryBatch`。Python 的 MRO 會讓多層繼承的除錯與重構成本隨著層數上升
- **Composition（組合）**：把 processor、query、batch 做成獨立物件傳入。更乾淨的 API，但需要明確的「組合」API 設計，對這個專案的規模來說可能過度設計
- **Protocol/Interface**：如果 mixin 之間有複雜的屬性依賴，用 `typing.Protocol` 可以讓 type checker 更嚴格

**注意事項**: Python 的 MRO（C3 linearization）在某些 edge case（菱形繼承）下會產生非直覺的行為。如果 mixin 數量超過 4 個，建議改為 composition。

---

## Pattern 2: LRU/KV 雙層 Parse Cache

**是什麼**: 文件解析結果的快取採用 LightRAG 的 key-string-value JSON storage 作為 persistence layer，搭配 MD5 hash key 和雙因子失效檢測（mtime + config）。

```python
def _generate_cache_key(self, file_path, parse_method=None, **kwargs):
    config_dict = {
        "file_path": str(file_path.absolute()),
        "mtime": file_path.stat().st_mtime,
        "parser": self.config.parser,
        "parse_method": parse_method or self.config.parse_method,
    }
    config_dict.update(relevant_kwargs)
    config_str = json.dumps(config_dict, sort_keys=True)
    return hashlib.md5(config_str.encode()).hexdigest()
```

**為什麼有效**: 文件解析是 RAG pipeline 中最耗時的步驟（一個複雜的 PDF 可能需要數分鐘）。快取讓同文件只解析一次，後續處理直接跳過。mtime + config 雙因子檢測確保檔案內容或解析參數變更時自動失效。

**程式碼位置**: [`processor.py:48-96`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/processor.py#L48-L96)

**何時可以借用**:
- 你的 pipeline 有**成本高且輸出穩定的步驟**（文件解析、embedding、程式碼編譯）
- 輸入可以透過 mtime/checksum 唯一識別
- 你已經有 KV storage 基礎設施

**替代方案**:
- **單純的檔案系統快取**：將 parsed result 寫成 JSON 檔案，mtime 做為 key。更簡單但缺少統一的 storage 管理
- **Redis/Memcached**：更快的效能但引入額外基礎設施
- **LRU in-memory cache**（`functools.lru_cache`）：適合 session 內快取，但跨 session 重啟後失效

**注意事項**: MD5（而非 SHA256）在這個場景是可接受的——cache key 的碰撞風險遠低於安全性風險。但若你的 key 包含使用者上傳的惡意內容，建議改用 SHA256。

---

## Pattern 3: Context-Aware 的 Multimodal 分析

**是什麼**: 在分析文件中的圖片/表格時，不只是傳入該區塊的內容，還從前後文中提取「上下文」一併餵給 VLM/LLM。

```
┌─────────────────────────────────────┐
│ Page N-1 text: "As shown in Figure  │
│ 3, the model achieves..."           │ ← Context window
├─────────────────────────────────────┤
│ [Figure 3: accuracy chart]          │ ← Current item
├─────────────────────────────────────┤
│ Page N+1 text: "This improvement is │
│ attributed to..."                   │ ← Context window
└─────────────────────────────────────┘
```

**為什麼有效**: 孤立的圖片描述（「一張折線圖」）對 RAG 檢索的幫助有限。有上下文的描述（「圖 3 顯示本論文提出的模型在 accuracy 指標上超越 baseline 約 5%）才能跟使用者的問題產生語義關聯。

**程式碼位置**:
- ContextConfig: [`modalprocessors.py:39-52`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/modalprocessors.py#L39-L52)
- ContextExtractor: [`modalprocessors.py:55-101`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/modalprocessors.py#L55-L101)

**何時可以借用**:
- 要處理的文件結構是**線性內容中穿插跨模態元素**（論文、報告、教科書）
- 跨模態元素的語義依賴前後文的解釋

**替代方案**:
- **不做 context**: 只餵圖片/表格的 raw content。更快更簡單，但 VLM 的輸出會缺乏文件的脈絡
- **整頁/整文件作為 context**: 直接將整個頁面或文件的文字都餵給 VLM。token 成本高，且 VLM 可能被不相關內容干擾
- **Hybrid**: 不只取前後文字，還取所在章節的標題、caption、footnote — RAG-Anything 的 ContextConfig 支援 `include_headers` 和 `include_captions` 參數，可以視為 hybrid 的一種

**注意事項**: Context window 的參數（window size、max context tokens）對結果品質有直接影響。預設值（window=1, max_tokens=2000）在短文件上合適，但對於長文件可能需要調大。

---

## Pattern 4: Feature-gated Import with Graceful Degradation

**是什麼**: `__init__.py` 對所有可選功能使用 `try/except ImportError` 進行防禦性導入，讓沒有安裝特定依賴的使用者仍然可以 import 主功能。

```python
try:
    from .resilience import retry, async_retry, CircuitBreaker
except ModuleNotFoundError:
    pass
except ImportError:
    pass
```

**為什麼有效**: RAG-Anything 有許多 optional dependencies（PaddleOCR、WeasyPrint、Pandoc），每個都可能不在使用者的環境中。防禦性導入確保 `import raganything` 在任何情況下成功，不會因為某個可選功能的 import 錯誤而全面失敗。

**程式碼位置**: [`__init__.py:1-114`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/__init__.py)

**何時可以借用**:
- 你的函式庫有可選的額外功能，每個功能有獨立的依賴
- 你希望 `import your_library` 永遠成功

**替代方案**:
- **Lazy import**: 在使用時才 import，而不是在 module level。更乾淨，但使用者會在呼叫時才發現 ImportError
- **All-in-one dependencies**: 把所有依賴寫進 `install_requires`。簡單但會強迫使用者安裝不需要的東西
- **Submodule 分隔**: 把可選功能放到 `your_library.extra.*`，使用者需要 `pip install your-lib[extra]` 才能使用。這是 RAG-Anything 在 `pyproject.toml` 中已經做的事（`all`、`image`、`text` 等 extras）

**注意事項**: 同時使用 `ModuleNotFoundError` 和 `ImportError` 的 double-try 有些冗餘——`ModuleNotFoundError` 是 `ImportError` 的子類。但這可能來自於不同 Python 版本的相容性需求。

---

## Pattern 5: PromptRegistry + Atomic Swap 的多語言支援

**是什麼**: `PromptRegistry` 是一個封裝了 prompt dict 的類別，支援 `swap()` 方法讓語言切換時一次替換所有 prompt，而不是逐筆更新。

```python
class PromptRegistry:
    def swap(self, prompts: dict[str, Any]) -> None:
        self._data = dict(prompts)
```

**為什麼有效**: 多語言 RAG 系統中，prompt 切換需要是原子的——如果在替換一半的時候有其他 thread/coroutine 讀取，會讀到混合語言的 prompt。`swap()` 透過一次性替換整個 dict 避免 race condition（Python 的 dict reference assignment 是 atomic 的）。

**程式碼位置**: [`prompt.py:13-65`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/prompt.py#L13-L65)

**何時可以借用**:
- 你的系統需要動態切換 prompt 語言或版本
- 多個 coroutine 可能同時讀取 prompt

**替代方案**:
- **單一 dict + lock**: `threading.Lock` 保護寫入。更明確但引入鎖的複雜度
- **每次使用時從檔案讀取**: 無 race condition 但 I/O 開銷大
- **Singleton 模式**: 類似 Registry 但通常只有一個實例

**注意事項**: Python dict reference assignment 的 atomicity 是 CPython 的實作細節（GIL），不是語言保證。但對這個使用場景已經足夠。若需要跨 interpreter 的 thread safety，需要使用 `threading.Lock` 或 `copy-on-write`。

---

## Pattern 6: Async-to-Sync Wrapper 用 `asyncio.to_thread`

**是什麼**: MinerU 的文件解析是同步的 blocking 操作，但 RAG-Anything 的 pipeline 是 async 的。它使用 `asyncio.to_thread` 將同步操作包裝為非同步，而不修改 MinerU 本身的程式碼：

```python
content_list = await asyncio.to_thread(
    doc_parser.parse_pdf,
    pdf_path=file_path,
    output_dir=output_dir,
    method=parse_method,
)
```

**為什麼有效**: 文件解析是 CPU-bound 操作，跑在 default executor（ThreadPoolExecutor）中，不會阻塞 async event loop。這讓 RAG-Anything 的 api 層可以保持 async 的一致體驗，同時相容同步的第三方程式庫。

**程式碼位置**: [`processor.py:472-478`](https://github.com/HKUDS/RAG-Anything/blob/e8c0081b7c2d3ff0a2e643fb10ea0f582b8f1dcf/raganything/processor.py#L472-L478)

**何時可以借用**:
- 你的 async code 需要呼叫一個 blocking 的第三方函式庫
- 你不想改寫該函式庫（或無法改寫）

**替代方案**:
- **`loop.run_in_executor`**: 更接近 asyncio 底層，但 API 較複雜
- **將整個 pipeline 改為同步**: 放棄 async 的效能優勢
- **使用 `asyncio.to_thread`**（Python 3.9+）: 最簡潔的方式

**注意事項**: `asyncio.to_thread` 使用 Python 的 ThreadPoolExecutor（default max_workers=min(32, os.cpu_count()+4)）。大量並行文件解析時可能會耗盡執行緒池，RAG-Anything 用 `max_concurrent_files` 參數控制並行度。

## RAG Pipeline 設計的哲學觀察

RAG-Anything 的設計反映了對 RAG pipeline 的幾個看法：

1. **文件理解是關鍵瓶頸**: 這個專案花最多力氣的地方不是 RAG 檢索（那部分委託給了 LightRAG），而是文件解析和跨模態理解。作者群顯然認為文件處理的品質決定了 RAG 的上限。

2. **LLM/VLM 是文件理解的必要元件，不是可選**: 對於圖表、表格、公式，RAG-Anything 堅持用 LLM/VLM 產生結構化描述，而不是試圖用純 rule-based 方法。這意味著接受更高的 latency 和成本換取更好的語義理解。

3. **Pipeline 而非 Framework**: 跟 LlamaIndex、LangChain 等「框架」不同，RAG-Anything 更像一個「應用」——它給你一條定義好的 pipeline，而不是讓你自己組合元件。

## 跟其他文件型 RAG 比較

| 面向 | RAG-Anything | LlamaIndex + PDF Reader | Unstructured.io |
|---|---|---|---|
| 文件解析 | MinerU（專注學術文件） | 多種 parser（PyMuPDF、LlamaParse） | 統一 API（支援 20+ 格式） |
| 跨模態處理 | 內建（Image/Table/Equation processors） | 需自行整合 VLM | 無（只做 parsing） |
| RAG 引擎 | 綁定 LightRAG | 多家支援（可換） | 不綁定 |
| 部署方式 | Python 套件，CLI | Python 套件，API | REST API |

RAG-Anything 的核心差異在於**跨模態處理和 context-aware 分析是內建功能，而非選配**。
