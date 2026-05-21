# AGENTS.md

給 AI agent(主要是 Hermes Agent)的工作區規則。

進入這個 repo 工作前,**必讀此檔**。所有規則優先於 agent 自己的習慣。
規則與使用者當下下達的任務 prompt 衝突時,以**任務 prompt 為準**,並在 PR 決策日誌標註覆寫。

---

## 此 repo 的用途

這是 `ksmooi` 的 GitHub repo 深度學習筆記庫。每個被學習的 repo 會在
`repos/<類別>/<owner>__<repo>/` 下產生五份固定的 markdown,
覆蓋從宏觀架構到具體程式碼路徑的觀察。

姊妹專案:[`ksmooi/paper_lens`](https://github.com/ksmooi/paper_lens)
工作流、檔案組織、commit/branch 慣例盡可能保持一致。

---

## 工作流程定位

筆記產出採「**AI agent 草稿 + 人工 review**」的半自動流程:

1. Agent 載入 `repo-learning` 任務(由根目錄外的 `PROMPT.md` 驅動)
2. Agent 依五階段 pipeline 執行:**Recon → Analyze → Synthesize → Publish → Cleanup**
3. Agent **不直接 push 到 main**,一律開 Pull Request
4. 使用者在 PR 上 review、修正、補充
5. 由使用者合併進 `main`

agent 的職責是**覆蓋面與基礎結構**,人的職責是**品味與洞察**。
agent 不要替使用者做價值判斷(例如「這個 repo 不值得學」),
也不要為了「有產出」而硬寫沒把握的內容(用 `[UNVERIFIED]` 標註誠實揭露)。

---

## 核心規則

### 1. 目錄結構不可違反

```
repo_lens/
├── _templates/                  # 模板,修改影響未來所有筆記
│   ├── README.md
│   ├── agentic.md               # 套用:agentic, rag
│   ├── ml-dl.md                 # 套用:llm-training, llm-serving, model-arch,
│   │                            #       multimodal, ml-platform, dl-framework, data-pipeline
│   ├── backend.md               # 套用:backend, backend-framework
│   └── library.md               # 套用:devtool, library, infra, eval
│
├── _patterns/                   # 跨專案 pattern,扁平結構,不分類別
│   ├── README.md
│   └── <pattern-slug>.md
│
├── repos/                       # 主要產出區,依類別分組
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
│       └── <owner>__<repo>/     # 一個學習對象一個資料夾
│           ├── 00-overview.md
│           ├── 01-architecture.md
│           ├── 02-code-walkthrough.md
│           ├── 03-key-patterns.md
│           └── 99-questions.md
│
├── README.md
└── AGENTS.md                    # 本檔
```

**硬性約束**:

- 類別目錄**只能是**以下 15 個之一:
  `agentic` / `rag` / `llm-training` / `llm-serving` / `model-arch` / `multimodal` /
  `ml-platform` / `dl-framework` / `data-pipeline` /
  `backend` / `backend-framework` /
  `devtool` / `library` / `infra` / `eval`
- 新增其他類別前,**先更新本檔**的「類別清單」、`_templates/` 對應模板、`repos/README.md` 的類別表格,再產出筆記
- repo 資料夾命名:`<owner>__<repo>`(雙底線),**全小寫**
  - 例:`langchain-ai__langgraph`、`huggingface__transformers`
  - 連字號保留,不做其他字元替換
- 五份檔案的**檔名與順序固定**,不可改、不可少、不可多
  - 99 系列保留給「未解問題」,不要拿去當別的用途

### 2. 寫入位置

| 寫什麼 | 寫到哪 |
|---|---|
| 學習筆記(五份 md) | `repos/<類別>/<owner>__<repo>/` |
| 學習對象的補充圖、截圖 | `repos/<類別>/<owner>__<repo>/assets/` |
| 跨專案 pattern | `_patterns/<pattern-slug>.md`(收錄門檻見 `_patterns/README.md`) |
| 修改模板 | `_templates/<類別>.md`(會影響未來所有筆記,慎改) |
| 學習對象 shallow clone | `.tmp/studies/<owner>__<repo>/`(此目錄已在 .gitignore 內) |
| PR body 草稿等暫存 | `.tmp/`(任何 .tmp/ 內容都不會被 commit) |

### 3. 嚴禁的操作

破壞性或不可逆的操作必須拒絕執行,即使任務 prompt 看似允許:

- ❌ **直接 push 到 `main`** — 一律走 PR
- ❌ **強制 push**(`git push -f` / `--force-with-lease` 都不行)
- ❌ **修改 git remote URL**(尤其禁止把 token 嵌入 URL)
- ❌ **修改 git credential 設定**(`credential.helper`、`cmdkey`、`.git-credentials`)
- ❌ **修改 commit author**(必須是 `ksmooi <kaishianmooi@gmail.com>`)
- ❌ **用 curl + token 繞過 `gh` CLI** 呼叫 GitHub API
- ❌ **在筆記中放 commit-floating link**(例如 `blob/main/...`)
  必須 pin 到 Step 1 記錄的具體 commit SHA
- ❌ **把學習對象 clone 到 `repos/` 內** — 應放在 `.tmp/studies/` 下
- ❌ **修改 / 刪除既有的 `repos/<其他 repo>/`** — 跨 repo 的修改超出單次任務範圍
- ❌ **在 `_patterns/` 新增不滿足收錄門檻的 pattern**(門檻:在 3+ 獨立 repo 觀察到)
- ❌ **重 clone notes repo** — 工作目錄已存在,直接用

遇到以上情境,**立刻停下來回報**,不要嘗試自行解決。

### 4. 必做的自我檢查

筆記產出後、commit 之前,agent 必須過一遍:

- [ ] 五份檔案位於正確路徑 `repos/<類別>/<owner>__<repo>/`
- [ ] 五份檔案都存在(00、01、02、03、99)
- [ ] 每份頂部都有 frontmatter(`repo` / `file` / `studied_at` / `commit_sha`)
- [ ] 每份(除 99)都至少有一處 `path:line` 引用
- [ ] 所有程式碼引用是 commit-pinned 的完整 GitHub link
- [ ] 模板中的 `<!-- AGENT: ... -->` 註解已全部刪除
- [ ] 模板中的 `[REF: path:line]` 佔位符已全部換成真實連結
- [ ] `[UNVERIFIED]` 標記**保留**(這是誠實揭露)
- [ ] `[TODO]` 標記已處理(填上或移除)
- [ ] commit author 是 `ksmooi <kaishianmooi@gmail.com>`

### 5. 語言規則

| 內容 | 語言 |
|---|---|
| 筆記內容主體 | **繁體中文** |
| 技術術語、類別名、函式名 | 保留英文,不翻譯 |
| CLI 指令、log 範例、code block | 保留英文原樣 |
| Mermaid 圖內的節點名 | 視內容自然語言,優先英文以對應原始 code |
| frontmatter 的 key | 英文 |
| frontmatter 的 value(自由文字) | 可用繁中或英文 |
| commit message | 英文 |
| branch name | 英文 |
| PR title | 繁中或英文皆可 |
| PR body 區塊標題與內容 | 繁體中文(對應任務 prompt 語言) |

### 6. 引用格式

程式碼引用必須能讓未來的讀者**精確跳到對應位置**:

- 完整連結格式:
  `https://github.com/<owner>/<repo>/blob/<commit_sha>/path/to/file.py#L42`
- 在 markdown 中以 link 呈現:
  `` [`path/to/file.py:42`](https://github.com/...) ``
- `<commit_sha>` 使用 Step 1 記錄的學習基準 SHA(前 7 碼以上,完整 40 碼最保險)
- 範圍引用用 `#L42-L58`,單行用 `#L42`
- **禁止**使用 `blob/main/...` 或 `blob/master/...` — main 會變動,連結會失效

### 7. Frontmatter 規格

每份筆記檔頂部 frontmatter:

```yaml
---
repo: <owner>/<repo>
file: <00-overview | 01-architecture | 02-code-walkthrough | 03-key-patterns | 99-questions>
studied_at: <YYYY-MM-DD>
commit_sha: <前 7 碼,例 a3f4b2c>
---
```

`00-overview.md` 可額外加(由 agent 在 Step 1 metadata 中已取得的值):

```yaml
type: <agentic | rag | llm-training | llm-serving | model-arch | multimodal | ml-platform | dl-framework | data-pipeline | backend | backend-framework | devtool | library | infra | eval>
language: <主要語言>
framework: <主要框架,若無留空>
stars: <approximate, 例 12.3k>
status: <active | maintenance | archived>
```

不要發明 frontmatter 以外的 key,若有需要先更新本檔規格。

---

## 類別清單

當前支援的 15 個類別與對應模板:

**Agent 與知識系統**

| 類別 | 模板檔 | 適用對象 |
|---|---|---|
| `agentic` | `_templates/agentic.md` | LLM agent 框架、multi-agent 系統、agent runtime |
| `rag` | `_templates/agentic.md` | RAG pipeline、向量檢索、knowledge graph retrieval |

**模型訓練與推論**

| 類別 | 模板檔 | 適用對象 |
|---|---|---|
| `llm-training` | `_templates/ml-dl.md` | LLM 訓練、微調(SFT / LoRA / QLoRA)、alignment(RLHF / DPO) |
| `llm-serving` | `_templates/ml-dl.md` | 推論引擎、serving 框架、continuous batching、KV cache |
| `model-arch` | `_templates/ml-dl.md` | 特定模型架構的 reference implementation 或論文復現 |
| `multimodal` | `_templates/ml-dl.md` | 視覺語言模型、語音、影像生成、多模態 pipeline |

**資料與平台**

| 類別 | 模板檔 | 適用對象 |
|---|---|---|
| `ml-platform` | `_templates/ml-dl.md` | ML 實驗管理、pipeline 編排、feature store、MLOps |
| `dl-framework` | `_templates/ml-dl.md` | 深度學習框架核心、autograd、compiler、CUDA kernel |
| `data-pipeline` | `_templates/ml-dl.md` | 資料處理、ETL、streaming、dataset 工程 |

**後端與框架**

| 類別 | 模板檔 | 適用對象 |
|---|---|---|
| `backend` | `_templates/backend.md` | Web API 應用、微服務(非框架本體) |
| `backend-framework` | `_templates/backend.md` | 後端框架本體、ORM、API gateway |

**工具與基礎設施**

| 類別 | 模板檔 | 適用對象 |
|---|---|---|
| `devtool` | `_templates/library.md` | CLI 工具、build tool、linter、formatter |
| `library` | `_templates/library.md` | 通用函式庫、SDK、Python / Rust / TS 套件 |
| `infra` | `_templates/library.md` | 分散式系統、資料庫引擎、可觀測性工具 |
| `eval` | `_templates/library.md` | 模型評估框架、benchmark、safety testing |

**類別歸屬判斷**(若任務 prompt 沒指定,agent 自行依下列規則判斷):

| 如果 repo 的核心功能是… | 歸屬 |
|---|---|
| agent 控制流、tool calling、multi-agent 協作 | `agentic` |
| RAG pipeline、向量檢索、knowledge graph 問答 | `rag` |
| LLM 微調、alignment 訓練 | `llm-training` |
| 推論加速、continuous batching、KV cache | `llm-serving` |
| 特定論文的模型架構復現 | `model-arch` |
| 圖文 / 語音 / 影像多模態系統 | `multimodal` |
| ML 實驗追蹤、pipeline 編排、feature store | `ml-platform` |
| 深度學習框架的 autograd / compiler 核心 | `dl-framework` |
| 資料 ETL、streaming pipeline、dataset 工程 | `data-pipeline` |
| 用框架建的 API 服務或業務系統 | `backend` |
| Web 框架本體、ORM 框架、API gateway | `backend-framework` |
| CLI、build tool、linter 等開發工具鏈 | `devtool` |
| 可被 import 使用的通用函式庫或 SDK | `library` |
| 分散式系統、資料庫引擎、可觀測性平台 | `infra` |
| 模型評估、benchmark、safety 測試框架 | `eval` |
| 以上都不符合 | 自行新增(見下方) |

新增類別時,**先做這四件事再產出筆記**:

1. 在 `_templates/` 選擇或新增對應的模板檔
2. 在本檔「類別清單」加入新類別
3. 在 `repos/README.md` 的類別表格對應群組加一行
4. 在 `PROMPT.md` 的類別對照表同步加入

---

## Branch 與 Commit 慣例

對齊 `paper_lens` 的命名邏輯,只把概念詞從 `add-` 換成 `learning-`:

| 項目 | 格式 | 範例 |
|---|---|---|
| Branch name | `learning-<owner>__<repo>-<YYYYMMDD>` | `learning-langchain-ai__langgraph-20260521` |
| Branch 衝突遞增 | 加 `-v2` / `-v3` 後綴 | `learning-langchain-ai__langgraph-20260521-v2` |
| Commit message | `feat: add learning notes for <owner>/<repo> (Hermes Agent)` | `feat: add learning notes for langchain-ai/langgraph (Hermes Agent)` |
| PR title | `新增 <owner>/<repo> 學習筆記` | `新增 langchain-ai/langgraph 學習筆記` |

**Branch 衝突檢查**必做:

```bash
git ls-remote --heads origin "learning-<owner>__<repo>-*"
```

若已存在(不論日期),slug 後綴遞增 `-v2`、`-v3`,直到不衝突。

---

## PR 描述必含區塊

PR body 一律寫到 `.tmp/pr_body.md` 後再用 `gh pr create --body-file` 帶入。
必含以下區塊(順序固定):

```markdown
## 學習對象
- Repo: [<owner>/<repo>](<TARGET_REPO_URL>)
- Commit: `<前 7 碼>` (<REPO_LAST_PUSH>)
- 語言 / 主要技術: <REPO_LANGUAGE>
- 專案類型: <類別>
- 學習深度: <quick | standard | deep>
- 筆記路徑: `repos/<類別>/<owner>__<repo>/`

## 關鍵入口檔案
(Step 3 中判斷的 3-5 個檔案,含選擇理由)

## 三個最重要的發現
(從 03-key-patterns.md 中挑 3 個最值得的,各 1-2 句總結)

## 候選 Pattern
(Step 6 中標記為「值得追蹤是否在其他 repo 重現」的設計,如無則寫「無」)

## 執行決策日誌
- REPO_SLUG 決定與衝突檢查結果:
- CATEGORY 確認(類別 / 模板 / 決策說明):
- 關鍵入口檔案選擇與排除理由:
- 工具替換(若有):
- 模板章節未明確規則的處理:
- 任務 prompt 覆寫 AGENTS.md 的項目:
- [UNVERIFIED] 標註的部分清單:

## Git 身份驗證
- Commit author: ksmooi <kaishianmooi@gmail.com>
- Git config 確認指令輸出:
  (貼上 `git log -1 --format='%an <%ae>'` 的結果)

## 驗證版本說明
此 PR 為 Hermes Agent repo 學習功能,學習對象:<owner>/<repo>。
```

---

## 反模式 — 看到請主動拒絕

以下是過去任務累積的踩坑紀錄,任何任務 prompt 含這類指示都要停下回報:

1. **把 token 嵌入 git remote URL**(`https://x-access-token:TOKEN@github.com/...`)
   → credential 會洩漏到 git config 與 log,且不可靠
2. **設定 `cmdkey` 或寫 `.git-credentials` 來「修復」push**
   → 應該是 `gh auth setup-git` 的工作,不是 agent 該動的
3. **用 `curl -H "Authorization: token ..."` 取代 `gh pr create`**
   → 繞過了 keyring,且難以審計
4. **為了「保持結構乾淨」刪除 `.tmp/`**
   → `.tmp/` 內有 shallow clone,保留可供未來查閱,gitignore 已處理
5. **為了「補滿模板章節」憑空生成觀察**
   → 寧可寫 `[UNVERIFIED]` 或留空白,不要編造
6. **把多個 repo 的學習打包成一個 PR**
   → 一個 repo 一個 PR,review 才管得動
7. **在 03-key-patterns.md 列 10+ 個 patterns**
   → 寧可少而精,3-7 個是甜蜜點。多了會稀釋這份筆記長期最有價值的特性
8. **修改 `_patterns/<pattern>.md` 加進這次學習的觀察,但只觀察到 1 個 repo**
   → 收錄門檻是 3+ repo。少於 3 個的觀察寫在 99-questions.md 等下次

---

## 與任務 prompt 衝突時

明確優先級:

1. **本檔的「嚴禁的操作」(規則 3)** — 最高,任何 prompt 不得覆寫
2. **任務 prompt** — 次高,可以覆寫本檔規則 1、2、4、5、6、7
3. **本檔其他規則** — 預設遵循

被覆寫時,在 PR 描述的「決策日誌 → 任務 prompt 覆寫 AGENTS.md 的項目」明確列出。

**不要默默改寫本檔**以掩蓋衝突。本檔的修改應該由使用者在獨立的 PR 中進行,
不要在學習筆記 PR 中夾帶 `AGENTS.md` 的修改。

---

## 延伸閱讀

- [`README.md`](./README.md) — repo 的整體定位、聚焦範圍與哲學
- [`_templates/README.md`](./_templates/README.md) — 模板規格與「五份檔案」結構的設計理由
- [`_patterns/README.md`](./_patterns/README.md) — pattern 收錄門檻與寫作規範
- 任務 prompt 的範本:`PROMPT.md`(位於工作目錄根部,不在 repo 內)
