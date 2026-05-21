---
repo: hkuds/lightrag
file: 99-questions
studied_at: 2026-05-21
commit_sha: 3bf2297
---

# LightRAG · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 Entity Extraction 不用 JSON mode 而是自訂 delimiter format？**
  - 我目前的推測: JSON mode 在某些 LLM 上不穩定，且 delimiter-based 的格式在 streaming processing 時更容易 incremental parsing [UNVERIFIED]
  - 相關程式碼: [`lightrag/prompt.py:11-61`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/prompt.py#L11)

- [ ] **Function callback 作為 LLM provider 抽象，如何處理需要 state 的 provider（如需要 session 管理的 Gemini）？**
  - 我目前的推測: 每個 provider module 內部自行管理 state，function callback 只是呼叫的介面 [UNVERIFIED]
  - 觀察: `lightrag/llm/gemini.py` 中確實有自己的 client 初始化邏輯

- [ ] **四層 storage 分離的邊界在哪？哪些情況下需要自己開發新的 storage 實作？**
  - 舉例: `DocStatusStorage` 跟 `KVStorage` 的行為很像（都是 key-value），但 LightRAG 把它獨立成一個抽象的動機是什麼？
  - 推測: 為了讓 doc status 有獨立的 flush/checkpoint 策略 [UNVERIFIED]

- [ ] **Workspace 隔離的粒度是否只到 namespace level？同一個 workspace 的兩個 LightRAG instance 會不會互相影響？**
  - 相關程式碼: [`lightrag/namespace.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/namespace.py)
  - 閱讀了 `shared_storage` 模組但尚未完全理解 lock 機制

- [ ] **Document deletion 後的 KG regeneration 是怎麼決定哪些 entity/relation 需要被移除的？**
  - 推測: 透過 `full_entities` 和 `full_relations` 中的 source_id 追蹤 [UNVERIFIED]

## 想問維護者的問題

- LightRAG 的 benchmark 結果顯示它跟傳統 RAG 相比，在什麼場景下退化？實體提取錯誤是否會造成「垃圾進垃圾出」？
- 為什麼內建的 embedding 和 reranker 沒有預設的 model？是為了避免 license 爭議還是效能考量？
- Gleaning 機制的 `max_gleaning` 預設值為 0 的設計考量？（預設不做 gleaning，雖然會降低 recall 但節省成本）

## 下次再看時的待辦

- [ ] 深入研究 `lightrag/kg/shared_storage.py` 中的 lock 機制與 workspace isolation 實作
- [ ] 對照 `nano-graphrag`（本專案的靈感來源），看 LightRAG 做了哪些改進
- [ ] 跑一次 reproduce 腳本（`reproduce/` 目錄），驗證論文的實驗結果是否可以重現
- [ ] 閱讀 `lightrag/utils.py` 中的 `handle_cache()` 實作，理解 LLM cache 的具體機制
- [ ] 對照 `MiniRAG`（姊妹專案），了解「使用小型模型的輕量 RAG」設計取捨

## 跨專案對照備忘

- **Storage Abstraction 設計**跟 `_patterns/llm-provider-abstraction.md` 類似（都是透過中央 registry + dynamic import），但 LightRAG 的 storage abstraction 層次更高（4 個 base class vs 1 個 interface）
- **Function callback 作為 provider 抽象** — 已在 `_patterns/llm-provider-abstraction.md` 中被歸納，LightRAG 的實作是「callback injection」變體，而非 class-based adapter
- **Graph-based RAG 的做法**跟 LangGraph 的 graph concept 完全不同 — LightRAG 的 graph 是資料結構（儲存 entities/relations），LangGraph 的 graph 是控制流（定義 agent 步序）。兩者同名但完全不同的概念，值得在 pattern 層面區分。

### 候選 Pattern（未達 3+ repo 收錄門檻）

以下設計值得追蹤是否在其他 repo 重現：

- **Pattern: Storage Registry + Dynamic Import** — LightRAG 使用 `STORAGES` dict 註冊所有 storage 實作，執行期依名稱動態 import。類似 plugin 註冊機制，但限定在資料層。已在 03-key-patterns.md 的 Pattern 1 中記錄。
- **Pattern: LLM-driven Structured Data Extraction with Delimiter Protocol** — 使用 delimiter 分隔的純文字格式（而非 JSON）控制 LLM 輸出結構化資料，搭配 gleaning 機制提高召回率。已在 03-key-patterns.md 的 Pattern 3 中記錄。
- **Pattern: Dual-Level Retrieval (Local + Global)** — 實體層級的精確檢索與概念層級的廣域檢索合併。已在 03-key-patterns.md 的 Pattern 4 中記錄。

### 已存在於 _patterns/ 的對照

- **Function callback 作為 LLM provider 抽象** → `_patterns/llm-provider-abstraction.md` 中描述的是 class-based adapter（class + abstractmethod），LightRAG 的 function callback 注入是該 pattern 的變體（介面從 class 簡化為 callable）。兩種方式各有適用場景，但背後要解決的問題相同。
- **Storage abstraction 的架構** → 同理，四層 base class + registry 也是「Provider Abstraction」pattern 在 storage 層的應用。
