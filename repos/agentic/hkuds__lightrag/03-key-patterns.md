---
repo: hkuds/lightrag
file: 03-key-patterns
studied_at: 2026-05-21
commit_sha: 3bf2297
---

# LightRAG · 值得偷學的設計

## Pattern 1: 四層儲存抽象 + 註冊表式實作載入

**是什麼**:
LightRAG 定義了四個抽象的 storage base class（`BaseGraphStorage`、`BaseKVStorage`、`BaseVectorStorage`、`DocStatusStorage`），所有實作透過一個中央註冊表 `STORAGES` dict 管理。

**為什麼有效**:
- 使用者在建構 LightRAG 時只需要指定字串名稱（如 `kv_storage="RedisKVStorage"`），系統自動查表載入對應模組
- 一個物理資料庫可以同時扮演多個 storage 角色（PostgreSQL 同時是 KV + Vector + Graph）
- 抽象的粒度恰到好處 — 不是一個「萬用 storage interface」，而是按 IO pattern 分離為 4 個介面

**程式碼位置**:
- 抽象介面：[`lightrag/base.py:172-430`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/base.py#L172) — `StorageNameSpace`、`BaseGraphStorage`、`BaseKVStorage`、`BaseVectorStorage`
- 註冊表：[`lightrag/kg/__init__.py:1-140`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/kg/__init__.py#L1)
- 動態載入：[`lightrag/lightrag.py:1177-1199`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/lightrag.py#L1177)

**何時可以借用**:
當你的系統需要支援多個 storage backend，且使用者應該能在配置中切換時。特別適合 database 生態工具（資料庫 GUI、migration tool、ETL pipeline）。

**注意事項**:
- 確保每一個 base class 的抽象方法數量合理（LightRAG 的 `BaseGraphStorage` 有 10+ 個 abstract method，實作成本不低）
- 同一個實作類別跨多個 namespace 時要注意資料隔離（LightRAG 用 `namespace: str` 欄位區分）

---

## Pattern 2: Function Callback 注入作為 LLM Provider 抽象

**是什麼**:
不採用 class-based adapter 或 interface，而是讓每個 LLM provider 提供一個符合約定的 callable function。切換 provider 時只需換 function。

**為什麼有效**:
- 比 class adapter 更輕量 — 定義一個符合簽章的 function 比實作一個 class 簡單
- 天然支援 partial application — `hashing_kv` 等共用參數透過 `functools.partial` 注入，provider module 不需要知道 cache 的存在
- 並發控制透過 decorator 而非繼承 — `priority_limit_async_func_call` 包裝在外部

**程式碼位置**:
- 注入點：[`lightrag/lightrag.py:400-401`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/lightrag.py#L400) — `llm_model_func: Callable | None`
- Provider 實例（以 OpenAI 為例）：[`lightrag/llm/openai.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/llm/openai.py)
- Partial injection 包裝：[`lightrag/lightrag.py:742-753`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/lightrag.py#L742)
- 並發控制 decorator：[`lightrag/utils.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/utils.py) 中的 `priority_limit_async_func_call`

**何時可以借用**:
當整合的外部服務數量多且持續增長，而每個服務的呼叫模式簡單（一次一進一出）時。比 interface-based 設計更適合 startup 階段。

**注意事項**:
- 放棄了 type 的靜態檢查優勢 — function signature 的約定是隱性的，要透過 docstring 或型別 stub 補償
- `EmbeddingFunc` 採用了 `dataclass` 來包裝 function + metadata，這是輕量 + 可檢查的折衷

---

## Pattern 3: LLM-driven 結構化資料提取（Entity Extraction Pipeline）

**是什麼**:
使用 LLM 從非結構化文字中提取結構化的 entities 和 relations，輸出格式由 delimiter 分隔的文字控制。

**為什麼有效**:
- 不需要訓練資料 — 只需設計好的 prompt 就能得到實體關係圖
- 透過「delimiter protocol」（`<|#|>` 分隔欄位、`<|COMPLETE|>` 標記結束）讓 LLM 輸出可被穩定 parse
- Gleaning 機制（`entity_extract_max_gleaning`）讓 LLM 補充遺漏的 entities，補償一次性提取的召回率問題
- Few-shot examples 幫助 LLM 理解輸出格式，減少 parse error

**程式碼位置**:
- Prompt 定義：[`lightrag/prompt.py:11-61`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/prompt.py#L11) — `entity_extraction_system_prompt`
- Few-shot examples：[`lightrag/prompt.py:102-183`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/prompt.py#L102)
- 提取邏輯：[`lightrag/operate.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/operate.py) 中的 `extract_entities()`
- Gleaning 控制：[`lightrag/lightrag.py:289-292`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/lightrag.py#L289) — `entity_extract_max_gleaning`

**何時可以借用**:
任何需要從大量非結構化文字中建立知識圖譜的場景。也適用於資料清洗、ontology 建構、文件分類。

**注意事項**:
- LLM 的品質直接影響提取品質 — 筆記中提到 `至少需要 32B 參數的模型`，輕量模型效果有限
- 成本高於傳統 NLP 方法 — 每份文件都會產生多次 LLM call（提取 + gleaning + description summarize）
- 提取結果的「長尾」問題 — 少見實體可能被忽略，需要 gleaning 機制補救

---

## Pattern 4: Dual-Level Text Chunk Retrieval

**是什麼**:
查詢時不是只看 embedding 相似度，而是從 entity-level（局部）和 relation-level（全域）兩個維度檢索，再合併結果。

**為什麼有效**:
- Local mode 適合問具體實體的問題（「Scrooge 在哪家銀行工作？」）
- Global mode 適合問高層次概念的問題（「這個故事的核心主題是什麼？」）
- Mix mode 兩者兼具，加上原始的 text chunks 兜底防漏

**程式碼位置**:
- Query dispatch：[`lightrag/lightrag.py:1326-1347`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/lightrag.py#L1326)
- KG query (local + global)：[`lightrag/operate.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/operate.py)
- Naive query（純向量）：[`lightrag/operate.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/operate.py)

**何時可以借用**:
當你的 RAG 系統需要回答「具體事實」和「抽象概念」兩種不同類型問題時。

**注意事項**:
- Global mode 對 LLM 的摘要能力要求更高
- Mix mode 的 context 可能很大，需要良好的 token 預算管理（`max_entity_tokens`、`max_relation_tokens`、`max_total_tokens`）

---

## Agent 設計的哲學觀察

LightRAG 的設計哲學與傳統 agent 框架形成鮮明對比：

- **偏好確定性而非彈性** — 所有步驟是固定的 pipeline，不讓 LLM 決定下一步要做什麼。這與 ReAct agent 的「LLM decides tool calls」哲學截然不同
- **傾向「資料管線」而非「對話代理」** — 從 index 階段的 batch processing 到 query 階段的結構化檢索，處處體現 ETL 思維而非對話思維
- **Prompt engineering 的系統化態度** — 所有 prompt 是精心設計的模板，不是隨便寫的指令。連 few-shot examples 都經過仔細挑選

## 跟其他 RAG 框架比較

| 面向 | LightRAG | 傳統 Vector RAG |
|---|---|---|
| 索引方式 | Entity-relation graph + vector embeddings | 僅 vector embeddings |
| 檢索維度 | 雙維度（圖結構 + 向量相似度） | 單維度（向量相似度） |
| 實體關係理解 | 透過圖遊走支援間接關係 | 僅語意相似度 |
| LLM 依賴度 | 高（entity extraction 需要 LLM） | 低（主要用於 response synthesis） |
| 查詢 mode | 5 種（local/global/hybrid/mix/naive） | 通常 1 種 |
