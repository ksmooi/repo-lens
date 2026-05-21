---
repo: HKUDS/LightRAG
file: 99-questions
studied_at: 2026-05-21
commit_sha: b62c260
---

# LightRAG · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 `mix` 是預設 mode 而非 `hybrid`？**
  - 我目前的推測：`mix` (KG + vector dual retrieval) 在實證上表現比純 `hybrid` (local + global graph) 更好，因為 vector search 可以補上 graph miss 的資訊。但沒有在 README 或論文中看到明確的 benchmark 數據支持這個選擇。[UNVERIFIED]

- [ ] **Pipeline concurrency contract 的跨 process 失效問題是否有 roadmap 要解決？**
  - `pipeline_status` 是 in-process dict（`shared_storage`），多個 LightRAG process 操作同一個後端 storage 時完全沒有協調機制。但 LightRAG 支援分散式後端（Redis/Milvus/Neo4j），這表示 author 預期多 process 部署。是已知限制且打算引入 distributed lock，還是預設 single-process 部署？
  - 相關程式碼：[`lightrag/pipeline.py`]

- [ ] **Entity extraction 的 multi-gleaning 在 LLM 產出品質上有回報遞減的 threshold 嗎？**
  - `entity_extract_max_gleaning` 的預設值是多少？超過幾次後的邊際效益趨近於零？如果有實驗數據，應該公開在 README 或參數註解中。
  - 相關程式碼：[`lightrag/operate.py:3232-3312`]

- [ ] **為什麼 `@final` decorator 在 `LightRAG` class 上？**
  - 既然已經用 mixin 組合來解決過大的 monolithic class，為什麼還要禁止 subclass？如果使用者想擴展 LightRAG 的行為（例如自訂 query routing），除了 monkey-patch 還有什麼方式？
  - 相關程式碼：[`lightrag/lightrag.py:157`]

- [ ] **Role system 的 `partial()` wrapper 是否會因為 Python 的 `partial()` 對 `kwargs` 的 shallow copy 行為導致問題？**
  - 當 role config 被 `aupdate_llm_role_config()` 熱更新時，`partial()` 內部的 `kwargs` 是 shallow copy。若某個 value 是 mutable dict，role wrapper 之間可能意外共享狀態。

## 想問維護者的問題

- LightRAG 的論文（EMNLP 2025）中，與 Microsoft GraphRAG 的 benchmark 比較結果如何？GraphRAG 使用更重的 global search（community summarization），LightRAG 的做法在什麼場景下勝出？
- 是否有計畫支援 streaming ingestion（邊插入邊查詢，而不是全部插入完才能查）？
- `kg_chunk_pick_method` 的 `WEIGHT` 與 `VECTOR` 兩種策略，在實務上分別適合哪些場景？
- FAQ 中說 LightRAG 支援多模態（multimodal），但 VLM role 實際上是怎麼被整合到 pipeline 中的？是只在 `analyze_multimodal` 階段使用，還是 query 階段也會走 VLM？

## 下次再看時的待辦

- [ ] 深入閱讀 `lightrag/llm/` 下的 provider binding 實作，特別是 Gemini 與 Anthropic 的 signature 適配方式
- [ ] 對照 LightRAG 與 Microsoft GraphRAG 的論文，理解兩者在 graph construction 上的差異
- [ ] 實際部署一次 LightRAG + Redis + Neo4j + Milvus 的全生產配置，驗證 `initialize_storages()` 的 startup 行為
- [ ] 讀 `lightrag/utils.py` 中 `truncate_list_by_token_size` 的實作細節，理解多 budget 的交互邏輯
- [ ] 測試 `ainsert_custom_kg` 跳過 LLM 階段的行為，特別是 schema 驗證規則

## 跨專案對照備忘

- **Mixin-based dataclass 組合**：LightRAG 的 `@final @dataclass LightRAG(_RoleLLMMixin, _StorageMigrationMixin, _PipelineMixin)` 與同一團隊的 `RAG-Anything`（`RAGAnything(QueryMixin, ProcessorMixin, BatchMixin)`）使用完全相同的模式。這是 HKUDS 團隊的 signature pattern——2 個 repo 已觀察到，若在第三個 repo 出現可升級為 `_patterns/` 的正式 pattern 檔。

- **Storage factory + env-var validation 組合**：與 `vLLM` 的 `AsyncEngineArgs` 做法類似──後者也透過 dict 註冊 engine 參數，並在 constructor 驗證。如果未來在第三個 repo 看到同類做法，可升級為 `_patterns/` 中的「factory-with-env-validation pattern」。

- **Role-based LLM routing**：與 `LangGraph` 的 `RunnableConfig` 概念類似──不同的 node 可以設定不同的 LLM binding。但 LightRAG 是用 stateless partial wrapper，LangGraph 是透過 config dict 傳遞。這兩種實作差異值得在 pattern 中記錄。

- **Per-workspace lock for pipeline concurrency**：與 `Ray` 的 placement group lock 概念相似，但 LightRAG 使用 asyncio.Lock 而非 distributed lock。若在更多 repo 觀察到這種「輕量 per-resource lock vs heavy distributed lock」的取捨討論，可歸納為 pattern。
