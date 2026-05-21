---
repo: hkuds/lightrag
file: 00-overview
studied_at: 2026-05-21
commit_sha: 3bf2297
type: agentic
language: Python
framework: LightRAG
agent_style: single-agent
stars: 35.4k
status: active
---

# LightRAG · 概覽

LightRAG 是一個以**知識圖譜 (Knowledge Graph)** 為核心的 Retrieval-Augmented Generation 引擎，由香港大學資料科學團隊開發。不同於傳統 vector-only RAG 僅依賴 embedding 相似度檢索，LightRAG 在文件索引階段由 LLM 提取 entity-relation 結構並建構成圖，讓 query 階段可以同時從「向量相似度」與「圖結構關聯性」兩個維度檢索資訊。

## 解決什麼問題

傳統 RAG 有兩個主要痛點：

1. **語意淺層** — 單純依靠 chunk embedding 的檢索，無法捕捉多個實體之間的間接關係與高層次概念
2. **全域理解薄弱** — 針對單一文件 chunk 的檢索無法回答需要跨文件整合的問題

LightRAG 用知識圖譜解決這兩個問題：圖結構的自然特性讓它可以從特定實體「遊走」到相關實體（雙語檢索的 local 模式），也可以透過高層次 keyword 聚合出跨文件的模式（global 模式）。

## 為什麼值得研究

- **Graph-based RAG 的實戰教科書** — 從 entity extraction prompt 設計、graph storage abstraction、到 dual-level retrieval 的實作，完整展示如何把 KG 整合進 RAG 管線
- **Storage Abstraction 的優秀實例** — 支援 6+ 種 KV/Graph/Vector/文件狀態儲存後端，且所有實作遵循統一的 abstract base class。這是 `_patterns/llm-provider-abstraction.md` 在儲存層的鏡像
- **Prompt Engineering 的系統化** — 所有 prompt template 集中在 `PROMPTS` dict 中管理，涵蓋 entity extraction、keyword extraction、response synthesis、entity summary 等多個環節
- **生產就緒** — 內建 REST API server (FastAPI)、WebUI (React)、Kubernetes 部署配置、Docker 支援、RAGAS 評估整合

## Agent 系統定位

LightRAG 的 agent 風格不是傳統的 ReAct loop，而是 **index-then-query** 的兩階段管線：

| 面向 | 選擇 |
|---|---|
| Agent 風格 | Index-then-query pipeline（非 ReAct） |
| 數量 | Single-agent |
| 編排方式 | Hardcoded pipeline（chunk → extract → build KG → query） |
| Memory | 無 agent memory，但透過 conversation_history 支援多輪對話 |
| Tool calling | 不適用（非 tool-calling agent） |

> 嚴格來說 LightRAG 是一個 RAG library 而非 agent framework。此處分類為 agentic 是任務 prompt 的明確指定。

## 技術棧一句話

`Python 3.10+` + `NetworkX` (graph engine) + `NanoVectorDB` (vector storage default) + multi-provider LLM (OpenAI, Ollama, Gemini, Anthropic, etc.)

## 健康度信號

- ⭐ Stars: ~35.4k
- 📅 最後 commit: 2026-05-20
- 👥 主要維護者: HKUDS team (Zirui Guo)
- 🔄 commit 頻率: 每週活躍

## 關鍵檔案速覽

| 檔案 | 職責 |
|---|---|
| [`lightrag/lightrag.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/lightrag.py) | 核心 orchestrator，`LightRAG` dataclass，管理所有 storage 子系統與對外 API |
| [`lightrag/base.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/base.py#L84) | 抽象基礎設施，`QueryParam`、`BaseGraphStorage`、`BaseKVStorage`、`BaseVectorStorage` |
| [`lightrag/operate.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/operate.py) | 索引與查詢的執行邏輯，含 `extract_entities()`、`kg_query()`、`naive_query()` |
| [`lightrag/prompt.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/prompt.py#L5) | 所有 LLM prompt template 的集中管理 |
| [`lightrag/kg/__init__.py`](https://github.com/hkuds/lightrag/blob/3bf2297/lightrag/kg/__init__.py#L113) | Storage 實作的註冊表與驗證邏輯 |

## 我會在後續筆記中回答的問題

- LightRAG 的 entity extraction prompt 如何設計，才能讓 LLM 穩定產出結構化的 KG 資料？
- 四層儲存抽象（KV/Graph/Vector/DocStatus）之間的關係與協作方式？
- Dual-level retrieval (local vs global) 的實際實作差異？
- 同一個儲存後端（如 PostgreSQL）如何同時扮演 KV/Graph/Vector 三種角色？
