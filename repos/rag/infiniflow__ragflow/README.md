---
repo: infiniflow/ragflow
file: README
type: rag
studied_at: 2026-05-24
commit_sha: e6dd3975
language: Python + Go
framework: Quart, Gin, LiteLLM, Elasticsearch/Infinity
agent_style: DAG-based component orchestration
stars: ~81k
status: active
---

# RAGFlow · 概覽

RAGFlow 是一個融合**深層文件解析**與**Agent 工作流**的開源 RAG 引擎，由 InfiniFlow（北京）開發，以 Apache-2.0 授權釋出。它不只是一個 RAG pipeline，而是一個包含完整的文件解析、檢索、agent 編排、LLM provider 抽象的**企業級上下文層**。

## 解決什麼問題

標準 RAG 系統面對的挑戰：非結構化文件（PDF、掃描件、簡報）的解析品質低落、chunking 策略選擇困難、且缺乏靈活的檢索後處理。RAGFlow 用三個設計回應：

1. **DeepDoc 深層文件理解** — 不只是 PDF text extraction，而是用 vision model + layout analysis 理解文件結構（表格、圖表、標題層級），產出可解釋的 chunking 結果
2. **Agentic DSL 工作流** — 不綁定固定的 RAG pipeline，而是提供一個宣告式 DAG 框架，使用者可以用拖放或 JSON 定義從 retrieval → rerank → generate 的完整流程
3. **LiteLLM 統一抽象** — 支援 40+ 種 LLM provider 與 embedding model，切換 provider 不需要改程式碼

## 為什麼值得研究

- **Dual-language 架構**：Python（Quart）負責 API 與 RAG 邏輯，Go（Gin）負責高效內部服務（搜尋引擎抽象、tokenizer、storage）。這在 RAG 開源專案中少見，多數同類專案是純 Python 或純 Go
- **Graph-based DSL** 而非傳統的 ReAct loop：RAGFlow 的 agent 系統不是簡單的 think→act→observe 迴圈，而是一個**宣告式 DAG**（在 `agent/canvas.py` 中定義的 `Graph` 類別），元件之間透過 upstream/downstream 連結，支援分支、迴圈、子圖。這讓非程式設計師也能透過 UI 拖放定義複雜的 RAG + Agent 流程
- **自建搜尋引擎 Infinity**：InfiniFlow 同時開發了 [Infinity](https://github.com/infiniflow/infinity) 向量資料庫，可取代 Elasticsearch 作為 RAGFlow 的底層文件儲存引擎。這種「搜尋引擎 + RAG 應用」的垂直整合策略，類似 Elastic 公司同時維護 Elasticsearch 與 Kibana
- **DeepDoc 文件解析品質**：`deepdoc/` 目錄下的 parser 與 vision 模組是專利級別的文件理解引擎，支援 resume parser、table parser、vision-based layout analysis

## Agent 系統定位

| 面向 | 選擇 |
|------|------|
| 編排風格 | DAG-based 元件圖（宣告式 DSL） |
| 執行模型 | Graph 逐節點執行，支援分支與迴圈 |
| Agent 風格 | 元件組合（非單一 ReAct agent），可嵌入子圖 |
| Memory | Redis-backed，支援 session 與 cross-session |
| Tool calling | LLM native function calling + MCP protocol |
| 程式碼執行 | gVisor sandbox（Python / JavaScript） |

## 技術棧一句話

`Python 3.13+ (Quart)` + `Go 1.25 (Gin)` + `LiteLLM (40+ providers)` + `Elasticsearch / Infinity` + `MinIO` + `MySQL` + `Redis` + `React (TypeScript)`

## 健康度信號

- ⭐ Stars: ~81k（開源 RAG 領域最高之一）
- 🍴 Forks: ~9.3k
- 📅 最後 commit: 2026-05-22（活躍開發中）
- 👥 主要維護者: InfiniFlow 團隊
- 🔄 commit 頻率: 每日活躍

## 與同類專案的比較

| 面向 | RAGFlow | LightRAG | LlamaIndex | LangChain |
|------|---------|----------|------------|-----------|
| 核心抽象 | DAG 元件圖 | Graph + Vector 雙軌 | Query Engine / Pipeline | Chain / Agent |
| 文件解析 | 自有 DeepDoc（vision-based） | 無（依賴外部 parser） | 豐富 Data Connector | 依賴第三方 |
| Agent 能力 | 宣告式 DSL + MCP + Sandbox | 無內建 | Agent + Tool 系統 | Agent 生態最成熟 |
| 部署方式 | Docker Compose（一鍵） | pip install | pip install | pip install |
| 語言 | Python + Go | Python | Python (+ TS) | Python (+ TS) |
| UI | 內建 React 管理後台 | 無 | 無（可整合 Streamlit） | 無（LangSmith 另計） |
| LLM 抽象 | LiteLLM（40+ providers） | 自製 wrapper | 自製 | LangChain 生態 |
| Vector store | Elasticsearch / Infinity | 多種（Nano/Milvus/Neo4j） | 11+ 種 | 大量整合 |
| 主要 trade-off | 部署重量級（需 Docker） | 輕量但缺 UI | 功能豐富但學習曲線陡 | 廣泛但抽象層 leaky |

## 我會在後續筆記中回答的問題

- 為什麼選擇 Quart 而非 FastAPI？異步效能與 Flask 生態的取捨在 RAGFlow 中如何平衡？
- Graph DSL 的 component 註冊機制比 LangChain 的 Chain 組合好在哪裡？
- DeepDoc 的 vision-based 文件解析在實際 chunking 品質上比傳統 parser（PyMuPDF、pdfplumber）好多少？
- Go 與 Python 之間的通訊介面是什麼？為什麼不直接用 gRPC？
- Infinity 作為 doc engine 與 Elasticsearch 相比，RAGFlow 做了哪些整合工作？
