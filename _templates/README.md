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

不同類型專案值得關注的面向不同。每份模板在「該追問什麼」上做了客製,而結構保持一致:

| 類型 | 模板檔 | 額外關注的面向 |
|---|---|---|
| Backend | [`backend.md`](./backend.md) | API 路由、ORM、中介層、可觀測性 |
| Agentic | [`agentic.md`](./agentic.md) | Agent 控制流、prompt、tool registry、memory |
| ML/DL | [`ml-dl.md`](./ml-dl.md) | 模型架構、訓練迴圈、實驗管理 |
| Library / SDK | [`library.md`](./library.md) | API 設計、擴充點、發布策略 |

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

若加入新類型(例如 `data-pipeline.md`、`devtool.md`),記得同步更新本檔案的對照表跟 `AGENTS.md` 中的判斷規則。
