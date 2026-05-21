---
repo: HKUDS/RAG-Anything
file: 00-overview
studied_at: 2026-05-21
commit_sha: e8c0081
type: rag
language: Python
framework: LightRAG + MinerU
stars: 20.5k
status: active
---

# RAG-Anything · 概覽

RAG-Anything 是一個「All-in-One RAG 系統」，主打**從文件解析到跨模態檢索的一條龍流程**。它不是單純的 RAG pipeline 框架（像 LlamaIndex 那樣），而是一個針對**複雜文件（PDF、圖表、公式、掃描件）** 設計的端到端解決方案——你餵文件，它負責解析→理解→索引→檢索。

## 解決什麼問題

現有 RAG 系統（LangChain、LlamaIndex、Naive RAG）對**非純文字文件**的處理普遍薄弱：

- PDF 內的圖片、表格、公式只是文字描述，沒有真正的視覺理解
- 跨模態內容的上下文在 chunking 過程中被切斷
- 圖片、表格、公式的語義被 flatten 成純文字，丟失結構資訊

RAG-Anything 的核心主張：**文件中的圖片應該被 VLM 看過、表格應該被 LLM 分析過、公式應該被保留 LaTeX 語義，然後才進向量庫**。

## 為什麼值得研究

1. **跨模態處理 pipeline 的完整實作** — 不只是文字 chunking，而是對 image/table/equation 各自有專屬 processor，用 LLM/VLM 產生結構化描述後再進 LightRAG 的 graph RAG 索引。這條 pipeline 的設計對任何想做「文件理解型 RAG」的人都有參考價值。

2. **Mixin-based 架構** — `RAGAnything` 本體是 dataclass + 三個 mixin（`ProcessorMixin`、`QueryMixin`、`BatchMixin`），而非傳統繼承。這種組合方式讓文件處理、查詢、批次處理三個職責各自獨立，值得在複雜 pipeline 專案中參考。

3. **Context-aware 的跨模態索引** — 對於文件中的圖片/表格，不只是孤立分析，而是從前後文字中提取 context 餵給 VLM，讓產生的描述更有上下文理解。

## RAG 系統定位

| 面向 | 選擇 |
|---|---|
| 處理流程 | Document Parse → Multimodal Analyze → LightRAG Insert → Query |
| 文件支援 | PDF、圖片、Office 文件（doc/docx/ppt/xls）、純文字 |
| 跨模態處理 | Image（VLM）、Table（LLM）、Equation（LLM）、Generic |
| 上下文整合 | 頁級/區塊級 context window，支援 caption/footnote 提取 |
| 檢索引擎 | LightRAG（Graph RAG + 向量檢索） |
| 文件解析器 | MinerU（預設）、Docling、PaddleOCR |

## 技術棧一句話

`Python 3.10+` + `LightRAG`（Graph RAG 引擎） + `MinerU`（文件解析） + `LLM/VLM`（跨模態分析） + `可選：WeasyPrint、Pandoc、ReportLab（文件輸出）`

## 健康度信號

- ⭐ Stars: ~20.5k
- 📅 最後 commit: 2026-05-21（高度活躍）
- 👥 主要維護者: Zirui Guo（HKUDS 團隊）
- 🔄 commit 頻率: 每日活躍，持續迭代中
- 🔀 Forks: 2,361

## 我會在後續筆記中回答的問題

- RAG-Anything 的 pipeline 各階段（Parse → Analyze → Index → Query）是怎麼串起來的？
- 跨模態處理器的 context 提取策略是什麼？
- LightRAG 在這裡的角色是什麼？RAG-Anything 跟直接用 LightRAG 差在哪？
- Cache 系統（parse cache / multimodal status cache / LLM response cache）的設計是怎樣的？
- 跟其他文件型 RAG 方案（如 LlamaIndex PDF reader、Unstructured.io）比起來有什麼取捨？
