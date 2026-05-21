# Repo Lens 💻🔍

用五份筆記的篇幅，把一個 GitHub 專案讀透。

這是我的個人程式碼閱讀筆記庫，涵蓋 agentic AI、RAG、LLM 訓練與推論、模型架構、多模態、ML 平台、深度學習框架、資料管線、後端系統、開發工具等 15 個技術領域。每份筆記圍繞一個**目標 repo** 展開，從架構全貌到具體程式碼路徑，試圖把該專案的**設計決策與核心取捨**講清楚。

筆記以繁體中文撰寫，主要是寫給未來的自己看，但歡迎任何路過的人閱讀、討論、糾錯。

取名 Repo Lens，是因為讀一個陌生的 codebase 很像用放大鏡看東西——不是看得更廣，而是看得更深、更清楚。每份筆記試圖把一個專案的關鍵設計放大到足夠清楚，讓人真正「看見」它在做什麼、為什麼這樣做，而不只是知道它存在。

姊妹專案 [`paper_lens`](https://github.com/ksmooi/paper_lens) 用同樣的透鏡對準 AI 論文。

---

## 這是什麼

`repo-lens` 是我深入研究 GitHub 開源專案的學習筆記庫。每個被研究過的 repo 都會留下五份結構化的 markdown,覆蓋從宏觀架構到具體程式碼路徑的觀察。

這個 repo 是 [`paper_lens`](https://github.com/ksmooi/paper_lens) 的姊妹專案 — 同樣的「透鏡」,一個對準論文,一個對準程式碼倉庫。

## 為什麼做這件事

讀過跟讀懂是兩件事,讀懂跟記得又是兩件事。我相信:

- **寫得出來才代表真的懂** — 強迫自己把觀察組織成文字會逼出沒想清楚的地方
- **三個月後的我會感謝現在的我** — 程式碼會變,但抽象出來的設計決策不會
- **跨專案累積才有複利** — 看過 10 個 agent 框架後,你會開始看到大家共同的取捨

## 聚焦範圍

依技術領域分為 15 個類別,每個類別對應一份專屬的分析模板:

**Agent 與知識系統**

| 類別 | 我關注什麼 | 典型對象 |
|---|---|---|
| `agentic` | Agent 控制流、prompt 系統、tool registry、memory 架構 | LangGraph、CrewAI、AutoGen |
| `rag` | retrieval 策略、chunking、indexing、reranking、hybrid search | LightRAG、LlamaIndex、Haystack |

**模型訓練與推論**

| 類別 | 我關注什麼 | 典型對象 |
|---|---|---|
| `llm-training` | 訓練迴圈、LoRA / RLHF / DPO、分散式策略 | LLaMA-Factory、Axolotl、TRL |
| `llm-serving` | continuous batching、KV cache、quantization | vLLM、SGLang、TGI |
| `model-arch` | 論文對應、關鍵元件、hyperparameter | nanoGPT、Mamba、RWKV |
| `multimodal` | 多模態融合、encoder 設計、inference pipeline | LLaVA、Whisper、Stable Diffusion |

**資料與平台**

| 類別 | 我關注什麼 | 典型對象 |
|---|---|---|
| `ml-platform` | 實驗追蹤、pipeline 編排、feature store | MLflow、ZenML、Feast |
| `dl-framework` | autograd、compiler、CUDA kernel | PyTorch、JAX、Triton |
| `data-pipeline` | ETL、streaming、schema evolution、data quality | DLT、Airbyte、Ray Data |

**後端與框架**

| 類別 | 我關注什麼 | 典型對象 |
|---|---|---|
| `backend` | 分層架構、API 設計、資料層、中介層 | FastAPI 應用、Django 應用 |
| `backend-framework` | 框架 API 哲學、middleware、DI、相容性策略 | FastAPI、SQLAlchemy、Gin |

**工具與基礎設施**

| 類別 | 我關注什麼 | 典型對象 |
|---|---|---|
| `devtool` | CLI 設計、build graph、plugin 機制 | Ruff、uv、Turborepo |
| `library` | 公開 API 設計、擴充機制、版本策略 | Pydantic、Tenacity、Tokio |
| `infra` | 分散式協調、storage engine、可觀測性 | Ray、ClickHouse、Qdrant |
| `eval` | 任務定義、metric 實作、model adapter | LMMS-Eval、DeepEval、Inspect AI |

## 目錄結構

```
repo-lens/
├── _templates/              # 各類型專案的筆記模板
├── _patterns/               # 跨專案累積的設計模式庫 ← 長期最有價值
├── repos/                   # 每個學習對象一個資料夾,依類別分組
│   ├── agentic/
│   ├── rag/
│   ├── llm-training/
│   ├── llm-serving/
│   ├── model-arch/
│   ├── multimodal/
│   ├── ml-platform/
│   ├── dl-framework/
│   ├── data-pipeline/
│   ├── backend/
│   ├── backend-framework/
│   ├── devtool/
│   ├── library/
│   ├── infra/
│   └── eval/
│       └── <owner>__<repo>/
│           ├── README.md       # 30 秒電梯簡報
│           ├── 1-architecture.md   # 靜態結構與設計決策
│           ├── 2-code-walkthrough.md  # 一條完整路徑的動態追蹤
│           ├── 3-key-patterns.md   # 值得偷學的技巧
│           └── 9-questions.md      # 還沒搞懂的東西
└── AGENTS.md                # 給 AI agent 的工作區規則
```

> 命名規則:
> - 15 個類別目錄見上方「聚焦範圍」表格,跟模板檔名對齊
> - repo 資料夾用 `<owner>__<repo>`(雙底線),全小寫,類別不寫進檔名
> - 底線開頭的資料夾(`_templates`、`_patterns`)是跨切面內容
> - 若 15 個類別都不合適,可自行新增;新增規則見 `AGENTS.md`

## 每份筆記長什麼樣

每個被研究的 repo 都產出**固定五份** markdown,目的明確不重疊:

- **`README.md`** — 這個專案解決什麼問題?為什麼值得研究?技術棧一句話。
- **`1-architecture.md`** — Mermaid 架構圖、分層說明、關鍵設計決策。
- **`2-code-walkthrough.md`** — 挑一條典型流程(一個請求、一次訓練、一輪 agent loop),從入口追蹤到出口,附上 `path:line` 引用。
- **`3-key-patterns.md`** — 3 到 7 個可以偷學或移植的技巧。**這是長期最有價值的檔案**。
- **`9-questions.md`** — 還沒搞懂的設計決策、想問維護者的問題、下次再看時的待辦清單。

## 跨專案沉澱:`_patterns/`

當同一個設計模式在 **3 個以上**不同 repo 出現,就會抽出來寫進 `_patterns/`,例如:

- `agent-state-machine.md` — 各家 agent 框架怎麼管 state
- `llm-provider-abstraction.md` — 不同 repo 如何抽象 LLM provider 切換
- `tool-registry-design.md` — Tool/function 註冊機制的常見做法
- `config-driven-experiments.md` — ML 專案如何讓實驗可重現

這層才是讀完一堆 repo 後真正屬於自己的知識資產。

## 工作流程

筆記產出採用「**AI agent 草稿 + 人工 review**」的半自動流程:

1. 用 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 載入 `repo-learning` skill
2. Agent 執行五階段 pipeline:**Recon → Analyze → Synthesize → Publish**
3. Agent 開 Pull Request,**不直接 push 到 main**
4. 我在 PR 上 review、修正、補上 agent 看不出來的判斷
5. 合併進 `main`

人工 review 是這個流程能產出可靠筆記的關鍵 — agent 負責覆蓋面與基礎結構,人負責品味與洞察。

## 怎麼瀏覽這個 repo

- 想找特定類別的學習成果 → 進 `repos/<類別>/`(15 個類別見上方表格)
- 想找特定 repo → `repos/<類別>/<owner>__<repo>/`
- 想看跨專案的設計模式 → 翻 [`_patterns/`](./_patterns/)
- 想了解 15 個類別的詳細定義 → 看 [`repos/README.md`](./repos/README.md)

> 想找特定主題或關鍵字,直接用 GitHub 的 repo 搜尋(按 `/` 或 `t`)比任何手工維護的索引都快。

## 給訪客的話

這個 repo 主要是寫給未來的我自己看的,內容會有以下特性:

- **不是教學** — 預設讀者已經有對應領域的基礎,筆記是觀察與抽象,不是入門指南
- **不重複 README** — 如果某段話原始 repo 的 README 已經寫得很好,我不會抄一遍,會直接連結過去
- **誠實標註不確定** — `[UNVERIFIED]` 標籤代表我推測但沒驗證的內容
- **會有偏見** — 這些是我的觀察與品味,不是客觀評論

如果你發現任何錯誤或想交流不同看法,歡迎開 Issue。

## 相關連結

- 姊妹專案:[`ksmooi/paper_lens`](https://github.com/ksmooi/paper_lens) — 同樣的方法論,對準論文
- 學習工作流的設計理由與細節,見 [`AGENTS.md`](./AGENTS.md)

## License

[MIT](./LICENSE) — 筆記內容可自由引用,但建議標註來源。被引用的原始 repo 各自遵循其授權條款。

---

<sub>持續累積中。</sub>
