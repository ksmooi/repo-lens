---
repo: HKUDS/LightRAG
file: README
studied_at: 2026-05-21
commit_sha: b62c260
type: rag
language: Python
framework: LightRAG (自製查詢引擎 + 圖資料索引)
stars: 35.4k
status: active
---

# LightRAG · 概覽

LightRAG 是香港大學資料科學團隊（HKUDS）開發的輕量 RAG 框架，核心創新在於**以 knowledge graph 取代傳統 flat vector index** 作為檢索中樞。它將文件萃取為實體與關係的圖結構，並以此支援六種查詢模式（local / global / hybrid / mix / naive / bypass），讓 LLM 在不同粒度的語境中選擇最合適的檢索策略。

## 解決什麼問題

傳統 RAG 系統的典型缺陷：

- **Flat chunk 缺乏結構**：單純將文件切 chunk 後做向量檢索，無法理解實體之間的語意關係
- **Global query 退化**：跨 chunk 的綜合性問題（「這本書的主題演變是什麼？」）在 flat retriever 下效果差
- **檢索與生成脫鉤**：retriever 不知道 generator 需要什麼樣的資訊，給了一堆不相干的 chunk

LightRAG 的做法：在 insert 階段建構知識圖（實體 → 關係 → 文本 chunks），query 階段根據模式在圖上導航，讓 **retrieval context 本身就是結構化的**。

## 為什麼值得研究

1. **Graph-as-Retrieval-Index 的實作典範**：不是把向量 DB 當唯一索引，而是以圖結構作為中介(lightrag/operate.py:3659)。實體向量搜尋找到候選節點後，沿邊擴散到關聯關係與 chunks，形成語意豐富的 context window。

2. **六種查詢模式的取捨**：從純向量（naive）到純圖（local / global），到混合（hybrid / mix），到跳過檢索（bypass）。每種模式對應不同程度的「需要多少圖結構資訊」，文件中有明確的 trade-off 說明。[`lightrag/base.py:88-120`]

3. **Storage 層的抽象設計**：四種 storage type（KV / Vector / Graph / DocStatus），每種都有多個 backend 實作（Json / Redis / Neo4j / Milvus / Qdrant / PostgreSQL / MongoDB / OpenSearch…），透過 factory pattern 在 runtime 決定 backend。[`lightrag/kg/__init__.py:1-45`]

4. **Pipeline 並行控制**：文件導入管線有精心設計的 concurrency contract（`pipeline_status` dict + per-workspace lock），處理 upload 與 scan 之間的競爭條件。

## 查詢模式比較

| 面向 | LightRAG (graph) | 傳統 flat RAG | 純 GraphRAG |
|---|---|---|---|
| 索引結構 | 實體-關係-文本三層圖 | 單層 flat chunks | 實體-關係圖 |
| Local 查詢 | 實體擴散 → 關係 → chunks | top-k chunk similarity | 圖導航 + LLM |
| Global 查詢 | 高層關鍵詞 → 全局節點擴散 | 通常需多輪檢索 | 文本摘要式 |
| 混合檢索 | hybrid + mix (weight/polling) | 無結構化混合支援 | 有限 |
| 部署複雜度 | 零外部依賴 (NetworkX + NanoVectorDB) | 通常需 PGVector/Pinecone | 需圖 DB |
| 冷啟動速度 | < 30s (JSON mode) | 接近即時 | 需建圖 |
| 主要 trade-off | 圖建構耗時 + LLM 呼叫加倍 | 缺乏語意結構 | 部署成本高 |

## 基本資訊

- **作者**: Zirui Guo (HKUDS)
- **授權**: MIT
- **語言**: Python 3.10+
- **主要依賴**: networkx, nano-vectordb, aiohttp, pydantic, tiktoken, tenacity
- **選擇性後端**: Redis / Neo4j / Milvus / Qdrant / PostgreSQL / MongoDB / Faiss / OpenSearch
- **貢獻者**: 180+（2026-05），相當活躍
- **論文**: EMNLP 2025 — "LightRAG: Simple and Fast Retrieval-Augmented Generation"

## 健康度信號

- ⭐ Stars: 35.4k (非常活躍的開源專案)
- 📅 最後 commit: 2026-05-21（今天仍有活動）
- 👥 主要維護者: Zirui Guo + 多位 committer
- 🔄 commit 頻率: 每週 10-30 commits（極活躍）

## 我會在後續筆記中回答的問題

- 圖索引與向量檢索之間的 hybrid 策略具體是怎麼實作的？
- 四種 storage type 的介面設計有什麼取捨？
- Pipeline 的 concurrency contract 是如何防止 race condition 的？
- LLM role system 讓不同任務使用不同 provider 的設計脈絡是什麼？
- 為什麼選擇 mix 作為預設 mode 而非 hybrid？
