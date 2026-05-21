# Templates

這個資料夾放各類型專案的筆記模板。每份模板包含**五個檔案**的骨架:

```
00-overview.md          # 30 秒電梯簡報
01-architecture.md      # 靜態結構與設計決策
02-code-walkthrough.md  # 動態路徑追蹤
03-key-patterns.md      # 值得偷學的技巧
99-questions.md         # 還沒搞懂的東西
```

## 為什麼模板要分類型

不同類型專案值得關注的面向不同。每份模板在「該追問什麼」上做了客製,而結構保持一致。

四份模板對應 15 個類別:

| 模板檔 | 對應類別 | 額外關注的面向 |
|---|---|---|
| [`agentic.md`](./agentic.md) | `agentic`、`rag` | Agent 控制流、prompt 系統、tool registry、memory 架構 |
| [`ml-dl.md`](./ml-dl.md) | `llm-training`、`llm-serving`、`model-arch`、`multimodal`、`ml-platform`、`dl-framework`、`data-pipeline` | 模型架構、訓練迴圈、資料 pipeline、實驗管理 |
| [`backend.md`](./backend.md) | `backend`、`backend-framework` | API 路由、ORM、中介層、可觀測性 |
| [`library.md`](./library.md) | `devtool`、`library`、`infra`、`eval` | API 設計、擴充點、發布策略 |

## 類別目錄對照

模板檔與 `repos/` 下的類別目錄對應關係:

| 模板檔 | 對應的 repos 子目錄 |
|---|---|
| `agentic.md` | `repos/agentic/`、`repos/rag/` |
| `ml-dl.md` | `repos/llm-training/`、`repos/llm-serving/`、`repos/model-arch/`、`repos/multimodal/`、`repos/ml-platform/`、`repos/dl-framework/`、`repos/data-pipeline/` |
| `backend.md` | `repos/backend/`、`repos/backend-framework/` |
| `library.md` | `repos/devtool/`、`repos/library/`、`repos/infra/`、`repos/eval/` |

這個對應關係很重要 — 它讓 agent 在 Step 5 寫入時,看 `CATEGORY` 就能同時決定「套用哪份模板」跟「寫到哪個目錄」。

新增類別時務必四處同步:
1. 在 `_templates/` 選擇或新增對應模板
2. 在 `repos/README.md` 的類別表格對應群組加一行
3. 在 `AGENTS.md` 的類別清單加一行
4. 在 `PROMPT.md` 的類別對照表同步加入

## 模板裡的標記約定

| 標記 | 意思 |
|---|---|
| `<!-- AGENT: ... -->` | 給 AI agent 的指示,完成後應**刪除**此註解 |
| `[UNVERIFIED]` | 推測但未驗證的內容,**保留**在最終文件中 |
| `[TODO]` | 待補充的項目,review 時應移除或填上 |
| `[REF: path/to/file.py:42]` | 程式碼引用佔位符,實際使用時換成完整 GitHub 連結 |

## 怎麼用

**Agent 自動產出**(主要場景): Hermes 的 `repo-learning` skill 會根據判斷的專案類型自動套用對應模板。

**手動使用**(偶爾): 把整份模板複製到 `repos/<owner>__<repo>/`,刪掉所有 `<!-- AGENT: ... -->` 註解,逐節填寫。

## 修改模板

修改模板會影響**未來**的所有新筆記,不會回頭改舊筆記。修改後建議先在一個小 repo 上跑一次,確認 agent 產出符合預期再正式啟用。

若加入新類別(例如 `robotics.md`、`security.md`),記得同步更新本檔案的對照表、`repos/README.md`、`AGENTS.md` 與 `PROMPT.md`。
