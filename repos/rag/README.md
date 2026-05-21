# rag/ — RAG 與知識檢索系統

**上層目錄**: [repos/](../README.md)

RAG pipeline、向量檢索、knowledge graph retrieval、hybrid search。
學習重點在於「怎麼把外部知識準確、高效地餵給 LLM」的工程設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **Retrieval 策略** | dense / sparse / hybrid 怎麼選?reranking pipeline 設計?self-RAG / adaptive RAG? |
| **Chunking 與 Indexing** | chunking 策略(fixed / semantic / hierarchical)?embedding 模型選擇?index 結構? |
| **Knowledge Graph** | entity / relation 抽取方式?graph 儲存後端?graph 與 vector 的 hybrid 查詢? |
| **Context 管理** | context window 怎麼壓縮?多輪對話的 context 累積策略?lost-in-the-middle 怎麼處理? |
| **與 LLM 的整合** | prompt 怎麼組裝檢索結果?citation 機制?hallucination 防護? |
| **Evaluation** | retrieval 品質怎麼評估?end-to-end RAG 怎麼評估?faithfulness / relevance 指標? |
| **儲存後端** | vector store 選擇(Chroma / Qdrant / Weaviate)?是否支援 metadata filter?update 策略? |

套用模板:`_templates/agentic.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| LightRAG | graph + vector hybrid RAG | 實體關係圖譜 + 向量雙軌檢索、local / global 查詢模式 |
| LlamaIndex | 全功能 RAG 框架 | 豐富的 data connector、query engine、agent 整合 |
| Haystack | pipeline-based RAG | 宣告式 pipeline、production 導向、強測試覆蓋 |
| Chroma | 向量資料庫 | 開發者友善、輕量 embedding、Python-first |
| Weaviate | 向量 + 知識圖譜 DB | GraphQL 查詢、hybrid search、多模態支援 |
| DSPy | 程式化 prompt 優化 | 用 optimizer 自動調 RAG pipeline 的 prompt |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 以 retrieval 為核心,agent 是選配 | ✅ `rag` | — |
| 以 agent 控制流為核心,RAG 是工具之一 | — | ✅ [`agentic`](../agentic/) |
| 向量資料庫引擎本體(非 RAG pipeline) | — | ✅ [`infra`](../infra/) |
| 評估 RAG 品質的 benchmark 框架 | — | ✅ [`eval`](../eval/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
