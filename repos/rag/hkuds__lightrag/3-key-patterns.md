---
repo: HKUDS/LightRAG
file: 03-key-patterns
studied_at: 2026-05-21
commit_sha: b62c260
---

# LightRAG · 值得偷學的設計

## Pattern 1: 可插拔 Storage Layer 搭配 Env-Driven Validation

**是什麼**: 四種 storage type（KV / Vector / Graph / DocStatus）各自定義 abstract interface，由 `factory.py::get_storage_class()` 根據初始化字串動態載入對應實作。每個實作標記所需的環境變數，在啟動時可提前驗證。

**為什麼有效**:
- 使用者不需要在 import time 決定後端：可以在開發時用 JSON/NetworkX 快速迭代，部署時無痛切換到 Neo4j/Milvus
- `STORAGE_ENV_REQUIREMENTS` dict 讓 system 在初始化時就 check 環境是否存在，而不是等到第一次 I/O 才炸掉。這對降低「寫入半小時後才發現 DB 連不上」這類慘案很有幫助
- `required_methods` 清單提供了自我文件化的介面契約

**程式碼位置**:
- Registry: [`lightrag/kg/__init__.py:1-45`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/kg/__init__.py#L1-L45)
- Factory: [`lightrag/kg/factory.py`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/kg/factory.py)
- Abstract base: [`lightrag/base.py`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/base.py)

**何時可以借用**: 你的專案有 3+ 種後端實作選項，且使用者可能在不同環境需要不同後端。例如：RAG 系統的儲存層、ML pipeline 的資料來源抽象、cache 後端抽象。

**注意事項**: 這種抽象層的介面設計必須非常穩定。LightRAG 的 `BaseKVStorage` 透過 `required_methods` 只定義了 `get_by_id` 與 `upsert` 這類最小值方法，但各個實作可能有自己的特有方法（例如 Redis 的 expire），這些不會被抽象層涵蓋。使用者若依賴特定後端的特有功能，切換時就會出問題。

**替代方案**:
- **Dependency injection**: 直接在 constructor 注入 storage instance，更靈活但更難序列化配置。LightRAG 的做法是「字串名稱 → factory 解析」，適合配置驅動的情境
- **Plugin system (importlib)**: runtime discover 所有實作。LightRAG 用的是手動註冊到 `STORAGE_IMPLEMENTATIONS` dict，比 importlib discovery 更顯式但也更需維護

---

## Pattern 2: Multi-Gleaning 實體萃取

**是什麼**: 對每個文本 chunk，不是只做一次實體萃取，而是反覆要求 LLM 檢查是否遺漏更多實體，重試次數由 `entity_extract_max_gleaning` 控制（預設 1-3 次）。

**為什麼有效**: 首次萃取常遺漏間接提到的實體或跨句子的關係。第二次呼叫時 LLM 看到「已找到的實體清單 + 原始文本」，更容易補上遺漏項目。這類似 Chain-of-Thought 的逐步推理，但應用在資訊萃取上。

**程式碼位置**: [`lightrag/operate.py:3232-3312`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/operate.py#L3232-L3312)

**何時可以借用**: 任何需要 LLM 從非結構化文字中萃取結構化資訊的場景，且 recall 比 latency 更重要。

**注意事項**:
- LLM 成本是 linear scaling（1 個 chunk × N 次 gleaning）。若你的 pipeline 有大量小 chunk，成本增加明顯
- 有 `max_extract_input_tokens` 護欄（`DEFAULT_MAX_EXTRACT_INPUT_TOKENS`），gleaning 階段若已超過 token 限制則跳過
- 回傳限制：`entity_extract_max_records` 與 `entity_extract_max_entities` 會對每輪回傳的實體數量做 cap，避免 LLM 亂吐一堆無效實體

**替代方案**:
- **Single-pass extraction**: 一次 LLM call 用夠好的 prompt 達到可接受 recall。更快速但 recall 較低
- **Multi-model verification**: 用一個輕量 LLM 做初次萃取，另一個判斷還缺什麼。更有彈性但複雜度大增
- **Rule-based post-processing**: 用 regex 或 NER model 補 LLM 遺漏的。適用於 domain-specific entities，但無法捕捉 LLM 才看得懂的語意關係

---

## Pattern 3: Factory + Env-Requirement 的雙層驗證

**是什麼**: Storage backend 的選擇不只是 class mapping，還包含 `STORAGE_ENV_REQUIREMENTS` dict，記錄每個實作需要的環境變數。`verify_storage_implementation()` 在初始化時就確認所有必要的 env vars 都有被設定。

**為什麼有效**: 傳統的 factory pattern 只解決「字串 → class」的 mapping，但不保證 runtime 環境是就緒的。LightRAG 的雙層設計讓錯誤在 `LightRAG()` constructor 就爆出來，而不是在 `aquery()` 才炸。

**程式碼位置**: [`lightrag/kg/__init__.py:48-100`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/kg/__init__.py#L48-L100)

**何時可以借用**: 你的專案有外部後端依賴（DB、queue、object storage），且使用者可能在不同環境使用不同後端組合。

**注意事項**: 環境變數檢查只能做「存在性」驗證，無法驗證連線是否可用。LightRAG 將連線測試留給 `initialize_storages()`，這是合理的職責分離：env var check 在 constructor、connection test 在一個獨立的 async init 方法。

**替代方案**:
- **Schema-based config validation (Pydantic)**: 用 Pydantic model 定義配置結構，自動驗證。更安全但引入額外依賴
- **Try-construct pattern**: 在 constructor 嘗試建立連線，失敗則拋錯。更徹底但讓 constructor 變慢且有 side effect

---

## Pattern 4: 以 Cache 為中樞的 LLM 呼叫最佳化

**是什麼**: LLM 呼叫前會計算 prompt hash，檢查 KV storage 中是否有相同的 cache entry。若存在則直接回傳 cached response，跳過 LLM 呼叫。

**為什麼有效**: 在多數 RAG 場景中，相同或高度類似的查詢會重複出現。LightRAG 的 cache 涵蓋關鍵詞萃取與實體萃取的 LLM 呼叫，這些通常是 pipeline 中最貴的環節。

**程式碼位置**:
- Cache 計算 hash: [`lightrag/utils.py:compute_args_hash`]
- Cache 讀取/寫入: [`lightrag/utils.py:handle_cache` / `save_to_cache`]

**何時可以借用**: 你的系統有重複性的 LLM 呼叫模式（e.g., 文件導入階段的實體萃取對相同 chunk 重複呼叫）。

**注意事項**:
- Cache 用 `hashing_kv`（也是 KV storage 的一種），所以 cache 後端與資料後端是同一套抽象。若你用 Redis 做 KV storage，cache 也自動是 Redis
- Cache key 是 prompt + model 參數的 hash。若 prompt 稍微不同（例如多了一個空格），就會 miss。這是 trade-off between cache hit rate and correctness
- 沒有 TTL/expire 機制。Cache 終身有效，除非手動清除。這對實驗階段可能有困擾

**替代方案**:
- **Semantic cache**: 用 embedding similarity 判斷 query 是否相近而非 exact hash match。更聰明但更複雜、embedding 本身也有成本
- **TTL-based cache**: 設定時間過期。適用於查詢 cache（訊息會過時），不適用於萃取 cache（通常是確定的）

---

## Pattern 5: Pipeline Concurrency Contract with Per-Workspace Lock

**是什麼**: 文件導入管線的並行控制不使用分散式鎖或 message queue，而是用 in-process `pipeline_status` dict + per-workspace `asyncio.Lock`。五個狀態欄位（`busy`、`destructive_busy`、`scanning`、`scanning_exclusive`、`pending_enqueues`）構成精細的互斥規則。

**為什麼有效**: 在單一 process 的 async 場景中，這比引入 Celery/RabbitMQ 輕量得多。狀態欄位的粒度區分了「普通處理中」、「清除 storage 中」、「掃描分類中」三種不同的忙碌狀態，讓 enqueue 與 scan 操作可以更安全地並發。

**程式碼位置**: [`lightrag/pipeline.py`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/pipeline.py)

**何時可以借用**: 你有一個非同步的 pipeline 系統，操作之間有複雜的共享資源競爭（例如：上傳文件 vs 清除 storage vs 掃描目錄），且 system 在單一 process 內運作。

**注意事項**:
- **跨 process 失效**：因為 `pipeline_status` 是 in-process dict，多 process 部署時每個 process 有自己的狀態副本。這是 LightRAG 從單一 process 場景出發的侷限
- **繁忙 wait**：`request_pending` 是 nudge 旗標，不是 semaphore。處理迴圈在 batch 之間檢查此旗標，若被設定則重新 query doc_status。這是一種非阻塞式通知，但也意味著有 latency（最多一個 batch 的處理時間）
- 若後端 storage 支援 transaction，應優先使用 DB-level 的鎖而非 application-level 的旗標

**替代方案**:
- **Database transaction + advisory lock**: 使用 PostgreSQL `SELECT ... FOR UPDATE` 或 Redis `SETNX`。跨 process 安全，但需 ACID 支援的後端。對應 LightRAG 後端多樣化的場景不適用
- **Message queue (Celery/RabbitMQ)**: 任務式而非狀態式。更正式但需要額外基礎設施

---

## Pattern 6: Role-Based LLM Routing 透過 `functools.partial`

**是什麼**: 四個 LLM role（extract / keyword / query / vlm）各自有獨立的 model function、kwargs、max_async、timeout。`_RoleLLMMixin` 在 constructor 中用 `functools.partial` 將基底 LLM function 包裝成各 role 的專用 wrapper。

**為什麼有效**: 直接使用 `partial()` 比設計 adapter 介面 + adapter registry 更簡單。Role 的數量不多（4 個），需要客製的參數也有限（model、kwargs、max_async、timeout），不需要完整的 plugin 架構。

**程式碼位置**: [`lightrag/llm_roles.py`](https://github.com/HKUDS/LightRAG/blob/b62c260/lightrag/llm_roles.py)

**何時可以借用**: 系統中只有少量（< 10）不同的 LLM 使用情境，且每個情境只差在 model/kwargs/timeout，不需要完全不同的 function signature。

**注意事項**:
- 所有 role 共用同一個基底 function signature。若你要為某個 role 使用完全不同的 LLM library（例如從 OpenAI 切到 Anthropic 且 Anthropic 的 API signature 不同），`partial()` 可能不夠，需要自訂 wrapper
- Role config 支援 runtime hot-update（`aupdate_llm_role_config()`），但 wrapper rebuild 後原先 pending 的呼叫不受影響
- `_RoleLLMState` 追蹤每個 role 的 queue status，支援 `get_llm_queue_status()` 報告，對 debug 與 monitoring 有幫助

**替代方案**:
- **Strategy pattern (adapter interface)**: 定義 `LLMAdapter` interface，每個 provider 實作。更完整但 4 個 role 的場景 over-engineering
- **Config dict per role**: 不封裝 callable，而是存 config dict 讓 caller 自行決定。更靈活但 call site 需要重複配置邏輯

---

## 設計哲學觀察

LightRAG 的作者群對系統設計有幾個明顯偏好：

1. **偏好多型儲存而非單一引擎**：不是用一個萬用向量 DB 解決所有事情，而是將儲存拆分為 KV / Vector / Graph / DocStatus 四種 type，各自選擇最適合的後端。這增加了部署複雜度但給了更大的選擇彈性。

2. **偏好 configuration over convention**：幾乎所有行為都可透過 constructor 參數或環境變數控制。從 chunk size、gleaning 次數、top_k 到 LLM timeout，全部可外部配置。這對需要精細調校的研究場景友好。

3. **對 LLM cache 的重視**：LLM 呼叫是整個 pipeline 中最昂貴的環節，LightRAG 不只在 query 層做 cache，在關鍵詞萃取和實體萃取也做。這在 production 場景下對成本控制非常關鍵。

4. **以 pragmatism 為先**：Pipeline 的 concurrency control 用 in-process dict 而非分散式鎖，儲存 factory 用字串名稱而非 importlib discovery。這些選擇在 80% 的使用場景中足夠好用，不需要為剩下 20% 過早複雜化。

## 跟其他 RAG 框架比較

| 面向 | LightRAG | LangChain | LlamaIndex |
|---|---|---|---|
| 檢索中樞 | Knowledge Graph + Vector | Vector (retriever chain) | Vector + 可插拔 retriever |
| 查詢多樣性 | 6 mode (local/global/hybrid/mix/naive/bypass) | Chain 組合 | Query engine 組合 |
| 圖索引 | Built-in (NetworkX) + 可替換 | 無內建圖索引 | 有限 (KnowledgeGraphIndex) |
| LLM 角色 | 4 role 內建 + 可擴展 | 無角色概念 | 無角色概念 |
| 部署範式 | Single process 為主 | Process-agnostic | Process-agnostic |
| 起步難度 | 低 (pip install + 5 lines) | 中 (需了解 chain/agent 生態) | 中 (需了解 data model) |
