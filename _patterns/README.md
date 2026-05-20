# Patterns

跨專案累積的設計模式庫。**這是 `repo-lens` 長期最有價值的資產**。

單一 repo 的筆記會隨原始程式碼演進而逐漸貶值——版本變、API 變、設計被推翻。但從多個 repo 抽象出來的設計模式不會。這個資料夾就是把零散的閱讀沉澱成自己的設計直覺的地方。

## 收錄門檻

不是看到什麼有趣的東西都寫進來。新增 pattern 要同時滿足以下條件:

1. **至少在 3 個獨立 repo 觀察到** — 避免過早抽象。一兩個專案的做法可能只是個人偏好,3+ 個才開始有「這是一類做法」的訊號。
2. **可以脫離特定框架描述** — 如果只能說「LangGraph 是這樣做的」,那它還沒抽象到 pattern 層次。
3. **存在合理的替代方案** — Pattern 的價值來自跟其他選擇的對照。如果一件事大家都這樣做、沒有其他選項,那是常識,不是 pattern。
4. **你能說出何時不適用** — 若想不出 anti-use-case,代表你還沒真的理解這個 pattern,先別寫。

## 不收錄什麼

- **語言層級的成語** — 例如 Python 的 context manager、TypeScript 的 discriminated union。這些屬於語言知識,不放這裡。
- **特定函式庫的 idiom** — 例如「在 FastAPI 用 Depends 做 DI」。這應該寫在該 repo 的 `03-key-patterns.md`。
- **單純的演算法** — 例如 reservoir sampling。屬於演算法知識,有更好的地方記。
- **過於泛化的原則** — 例如「分層架構」、「依賴倒置」。這些抽象到沒有實作參考價值,屬於設計原則範疇。

## 命名規範

| 規則 | 範例 |
|---|---|
| 全小寫 + 連字號 | `agent-state-machine.md` ✓ |
| 名詞或名詞短語為主 | `tool-registry-design.md` ✓ |
| 避免動詞開頭 | `managing-llm-providers.md` ✗ |
| 避免框架名 | `langgraph-state.md` ✗ → `agent-state-machine.md` ✓ |
| 一個檔案一個 pattern | 不要塞「相關 pattern 大合集」 |

## 每份 pattern 的標準結構

每個 pattern 檔包含以下章節,順序固定:

```
# <Pattern 名稱>

> 一句話定義(< 30 字)

## 它解決什麼問題
## 核心想法
## 觀察到的變體
## 跟替代方案的對照
## 何時用 / 何時不用
## 在不同 repo 的實作
## 我的建議
```

詳細寫法參考 [`agent-state-machine.md`](./agent-state-machine.md) 跟 [`llm-provider-abstraction.md`](./llm-provider-abstraction.md),它們是 canonical example。

### 「在不同 repo 的實作」這節特別重要

這節是 pattern 跟一般技術文章的最大差別。要列出至少 3 個你實際讀過的 repo,並用 `path:line` 形式指向具體程式碼。它讓這份 pattern 可驗證、可追溯,也避免變成空泛的「我覺得應該這樣設計」。

## 升級流程

從觀察到 pattern,通常經過這些階段:

1. **單一 repo 觀察** — 寫在該 repo 的 `03-key-patterns.md`,標註「這個做法有趣」
2. **第二次看到** — 在第二個 repo 的 `99-questions.md` 寫「跟某 repo 類似 → 候選 pattern」
3. **第三次看到** — 滿足收錄門檻,新增到 `_patterns/`
4. **持續觀察** — 後續看到同類做法時,回頭更新「在不同 repo 的實作」這節

Agent 會在 Phase 5 主動偵測候選 pattern 並提議升級,但最終決定權在人——避免 agent 為了「有產出」而強行歸納。

## 修訂規則

Pattern 不是寫完就定案。當你:

- 看到第 4、5、6 個 repo,實作經驗多了
- 自己嘗試套用了,有了實戰心得
- 發現原本的歸納太窄或太寬

都應該回來改 pattern 檔。Pattern 檔的 commit history 本身就是你對這類設計理解演進的紀錄。

修改 pattern 時建議:

- 把舊版本的觀點保留在 commit message,不要直接覆蓋掉
- 重大修訂在檔案頂部加 `> **2026-03**: 大幅改寫「核心想法」一節,原版觀點過於受 LangGraph 影響` 之類的註記

## 索引

<!-- AGENT: 新增 pattern 時更新這個清單 -->

| Pattern | 已觀察 repo 數 | 最後更新 |
|---|---|---|
| [agent-state-machine](./agent-state-machine.md) | 3 | YYYY-MM-DD |
| [llm-provider-abstraction](./llm-provider-abstraction.md) | 4 | YYYY-MM-DD |
