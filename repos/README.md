# Repos — GitHub 專案學習目錄

這裡是 [Repo Lens](../README.md) 所有 GitHub 專案學習筆記的存放處,依技術領域分類。

每個類別目錄下,每個學習對象是一個獨立資料夾,命名格式為 `{owner}__{repo}/`。雙底線分隔 owner 與 repo 名稱,全部轉小寫。

```
repos/
└── {category}/
    └── {owner}__{repo}/
        ├── README.md          # 30 秒電梯簡報與競品比較
        ├── 1-architecture.md      # 架構圖與設計決策 trade-off
        ├── 2-code-walkthrough.md  # 一條典型流程的完整追蹤
        ├── 3-key-patterns.md      # 值得偷學的設計(含替代方案)
        └── 9-questions.md         # 未解問題與待辦清單
```

---

## 15 個技術類別

### Agent 與知識系統

| 類別 | 說明 | 典型專案 |
|------|------|----------|
| [agentic](./agentic/) | LLM agent 框架、multi-agent 系統、orchestration、agent runtime | LangGraph、CrewAI、AutoGen、Hermes |
| [rag](./rag/) | RAG pipeline、向量檢索、knowledge graph retrieval、hybrid search | LightRAG、LlamaIndex、Haystack、Chroma |

### 模型訓練與推論

| 類別 | 說明 | 典型專案 |
|------|------|----------|
| [llm-training](./llm-training/) | LLM 預訓練、微調(SFT / LoRA / QLoRA)、alignment(RLHF / DPO) | LLaMA-Factory、Axolotl、TRL、OpenRLHF |
| [llm-serving](./llm-serving/) | 模型推論引擎、serving 框架、continuous batching、KV cache | vLLM、SGLang、TGI、LitServe |
| [model-arch](./model-arch/) | 特定模型架構的 reference implementation 或論文復現 | nanoGPT、RWKV、Mamba、DiT |
| [multimodal](./multimodal/) | 視覺語言模型、語音、影像生成、多模態 pipeline | LLaVA、Whisper、Stable Diffusion、CogVLM |

### 資料與平台

| 類別 | 說明 | 典型專案 |
|------|------|----------|
| [ml-platform](./ml-platform/) | ML 實驗管理、pipeline 編排、feature store、MLOps 工具鏈 | MLflow、ZenML、Metaflow、Feast |
| [dl-framework](./dl-framework/) | 深度學習框架核心、autograd、compiler、CUDA kernel 設計 | PyTorch、JAX、Triton、tinygrad |
| [data-pipeline](./data-pipeline/) | 資料處理、ETL、streaming、dataset 工程、data quality | DLT、Airbyte、Ray Data、Great Expectations |

### 後端與框架

| 類別 | 說明 | 典型專案 |
|------|------|----------|
| [backend](./backend/) | Web API 應用、微服務、後端服務型系統(非框架本體) | FastAPI 應用、Django 應用、NestJS 應用 |
| [backend-framework](./backend-framework/) | 後端框架本體、ORM、API gateway、middleware 框架 | FastAPI、SQLAlchemy、Gin、Axum |

### 工具與基礎設施

| 類別 | 說明 | 典型專案 |
|------|------|----------|
| [devtool](./devtool/) | CLI 工具、build tool、linter、formatter、開發輔助工具 | Ruff、uv、Turborepo、Just |
| [library](./library/) | 通用函式庫、SDK、Python / Rust / TS 套件 | Pydantic、Tenacity、Tokio、Zod |
| [infra](./infra/) | 分散式系統、容器編排、資料庫引擎、可觀測性工具 | Ray、Temporal、ClickHouse、Qdrant |
| [eval](./eval/) | 模型評估、benchmark、red-teaming、safety testing | LMMS-Eval、OpenCompass、DeepEval、Inspect AI |

---

## 類別歸屬速查

同一個 repo 有時橫跨多個主題,下表列出常見的歸屬決策:

| 如果這個 repo 的核心功能是… | 歸屬類別 |
|---------------------------|----------|
| Agent 控制流、tool calling、multi-agent 協作 | `agentic` |
| 向量檢索、RAG pipeline、knowledge graph 問答 | `rag` |
| LLM 微調、alignment 訓練(SFT / RLHF / DPO) | `llm-training` |
| 模型推論加速、continuous batching、KV cache | `llm-serving` |
| 特定論文的模型架構復現 | `model-arch` |
| 圖文 / 語音 / 影像多模態系統 | `multimodal` |
| ML 實驗追蹤、pipeline 編排、feature store | `ml-platform` |
| 深度學習框架的 autograd / compiler 核心 | `dl-framework` |
| 資料 ETL、streaming pipeline、dataset 工程 | `data-pipeline` |
| 以某框架建構的 API 服務或業務系統 | `backend` |
| Web 框架本體、ORM 框架、API gateway | `backend-framework` |
| CLI、build tool、linter 等開發工具鏈 | `devtool` |
| 可被 import 使用的通用函式庫或 SDK | `library` |
| 分散式系統、資料庫引擎、可觀測性平台 | `infra` |
| 模型評估、benchmark、safety 測試框架 | `eval` |

跨類別專案的完整判斷規則見 [`AGENTS.md`](../AGENTS.md#類別清單)。

---

## 模板對照表

每個類別在撰寫時套用對應的 `_templates/*.md`:

| 套用模板 | 對應類別 | 主要分析面向 |
|----------|----------|--------------|
| [`agentic.md`](../_templates/agentic.md) | `agentic`、`rag` | agent 控制流、prompt 系統、tool registry、memory 架構 |
| [`ml-dl.md`](../_templates/ml-dl.md) | `llm-training`、`llm-serving`、`ml-platform`、`dl-framework`、`model-arch`、`data-pipeline`、`multimodal` | 訓練/推論迴圈、模型架構、資料 pipeline、實驗管理 |
| [`backend.md`](../_templates/backend.md) | `backend`、`backend-framework` | API 設計、資料層、中介層、可觀測性接入 |
| [`library.md`](../_templates/library.md) | `devtool`、`library`、`infra`、`eval` | 公開 API 設計、擴充機制、版本相容性策略 |

---

## 推薦的學習路徑

不確定從哪裡開始?幾條有脈絡的路徑:

**從 RAG 出發,往 Agent 走**
`rag/` → `agentic/` → `llm-serving/`

**從模型架構出發,往訓練走**
`model-arch/` → `dl-framework/` → `llm-training/`

**從推論效率出發**
`llm-serving/` → `dl-framework/` → `infra/`

**從後端工程出發,往 AI 系統走**
`backend-framework/` → `backend/` → `rag/` → `agentic/`

**從資料工程出發**
`data-pipeline/` → `ml-platform/` → `llm-training/`

---

> 本目錄由 Hermes Agent 協作維護。新增與命名規則見 [`AGENTS.md`](../AGENTS.md)。
