---
repo: wecoai/aideml
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 40dcf28
---

# AIDE ML · 值得偷學的設計

## Pattern 1: Tree Search as Agent Architecture

**是什麼**：
AIDE 不把 agent loop 設計成「思考 → 行動 → 觀察」的線性序列，而是每個 agent step 產生一個**新節點**加入到解決方案樹中。樹的根節點是初始草稿，子節點是改進或除錯版本。搜尋策略（ε-greedy + bug 優先）決定下一步探索哪個分支。

**為什麼有效**：
- 線性 agent（OpenHands、傳統 ReAct）容易 stuck in local optimum——當前的改進方向剛好是錯的，就浪費了一次迭代
- 樹搜尋讓 agent 可以同時保持多條路線：初始 `num_drafts=5` 提供足夠的 diversity，然後貪婪改進最佳路線
- Buggy 節點的自動 debug 形成另一條探索路徑——不是直接拋棄錯誤的嘗試
- MLE-Bench 實證：樹搜尋比最佳線性 agent 多贏 **4 倍獎牌**

**程式碼位置**：[`agent.py:61-92`](https://github.com/wecoai/aideml/blob/40dcf28/aide/agent.py#L61-L92)（search_policy）、[`journal.py:134-192`](https://github.com/wecoai/aideml/blob/40dcf28/aide/journal.py#L134-L192)（Journal/Node 結構）

**替代方案**：
- **傳統 MCTS**：有 UCB 公式平衡 exploration/exploitation，需要模擬（playout）。AIDE 不做 MCTS 因為程式碼執行的成本太高，無法做大量模擬
- **演化演算法**：將節點視為基因，透過 crossover/ mutation 產生後代。AIDE 選擇 LLM-guided mutation（improve/debug）而非隨機 crossover
- **線性 beam search**：保持 top-K 解而不形成樹（無 parent-child 關聯）。樹結構的價值在於除錯路徑的追溯——知道某個節點是從哪個 bug 修正來的

**何時可以借用**：
當你的 agent 任務有**明確的 metric 回饋**、每次評估成本固定（數秒到數分鐘）、且搜尋空間可以離散化時。特別是 ML/AutoML 場景、超參數最佳化、程式碼生成。

**注意事項**：
- 樹搜尋的節點數會線性增長——每次 step 產生一個節點，steps=20 時總共 20 個節點。若你的場景需要成千上萬次探索，這個設計需要改進
- 沒有 prune 機制——所有節點（包含 buggy）都保留在 Journal 中。對長期運行可能造成 memory 增長
- 不支援平行探索（sequential 的 tree search），雖然原則上可以平行化 initial drafts

---

## Pattern 2: Two-LLM Architecture (Code Writer + Metric Evaluator)

**是什麼**：
AIDE 使用兩個不同的 LLM 角色：(1) **coding model**（預設 `o4-mini`）負責生成/改進 Python 程式碼；(2) **feedback model**（預設 `gpt-4.1-mini`）負責評估執行結果、提取 metric、判斷 bug。兩者不共享對話歷史，每次都是獨立呼叫。

**為什麼有效**：

| 面向 | Coding model | Feedback model |
|---|---|---|
| 主要工作 | 產出大量 token（完整程式碼） | 仔細閱讀執行輸出，結構化回報 |
| Token 消耗 | 高（output tokens 多） | 低（僅回傳 function call 結果） |
| Required skill | Python 程式設計、ML pipeline 知識 | 錯誤分析、metric 理解 |
| Output format | 非結構化（自然語言 + code block） | 結構化（function calling） |

將兩者分離讓每個模型專注於自己擅長的工作。Coding model 不需要做「判斷」，它只需要「寫」；feedback model 不需要「寫程式碼」，它只需要「讀結果並判斷」。

**程式碼位置**：
- Coding model 設定：[`config.yaml:46`](https://github.com/wecoai/aideml/blob/40dcf28/aide/utils/config.yaml#L46)（`agent.code.model`）
- Feedback model 設定：[`config.yaml:51`](https://github.com/wecoai/aideml/blob/40dcf28/aide/utils/config.yaml#L51)（`agent.feedback.model`）
- Coding prompt：[`agent.py:175-205`](https://github.com/wecoai/aideml/blob/40dcf28/aide/agent.py#L175-L205)
- Evaluation prompt：[`agent.py:301-321`](https://github.com/wecoai/aideml/blob/40dcf28/aide/agent.py#L301-L321)

**替代方案**：
- **單一模型做所有事**：簡單但要求模型同時擅長寫 code 和做判斷。若用同一個模型，coding 的 token 消耗會壓縮可用於評估的 context
- **Role prompting**（同一模型不同 system prompt）：成本固定但 token 預算仍是問題
- **Rule-based evaluator**：對 metric extraction 可以用正則表達式取代 LLM，但對 bug 判斷和摘要就做不到

**何時可以借用**：
任何 agent 系統中若有「生成→執行→評估」的循環，且評估需要語意理解時。特別適合以下場景：
- 生成階段 token 消耗大（例如寫完整程式碼、長文件）
- 評估階段需要彈性判斷（不只檢查 exact match）
- 評估不需要看全部 context，只看執行結果

**注意事項**：
- 兩個模型間沒有溝通——feedback model 看到的程式碼是生成的，但它不知道 coding 的思考過程。這可能遺漏一些 context
- 成本：兩次 LLM call / step。steps=20 時總共約 40 次 LLM call
- 延遲：要等兩次 LLM call 都完成才能完成一個 step

---

## Pattern 3: MetricValue with Direction-Aware Comparison + WorstMetricValue Sentinel

**是什麼**：
`MetricValue` 是一個封裝 metric 值的類別，知道「越大越好」還是「越小越好」，並實現了方向感知的比較運算子。`WorstMetricValue` 是一個 subclass，永遠比任何真實 metric 值差——作為 buggy node 的標記值。

```python
# 越高越好
acc = MetricValue(0.95, maximize=True)
# 越低越好
rmse = MetricValue(0.5, maximize=False)

assert acc > MetricValue(0.9, maximize=True)     # 正確：0.95 > 0.9
assert rmse > MetricValue(1.0, maximize=False)    # 正確：0.5 is "better" than 1.0
assert MetricValue(0.9, maximize=True) > WorstMetricValue()  # True
```

**為什麼有效**：
- 消除了 metric 方向的 boilerplate——`get_best_node()` 直接 `max(nodes, key=lambda n: n.metric)`，不需要知道 metric 是 accuracy 還是 RMSE
- WorstMetricValue 保證 buggy node「永遠不會被選為最佳」——即使所有正常節點都有異常的 metric 值
- `maximize` 欄位由 LLM 在 evaluation step 決定（`submit_review` 中的 `lower_is_better`）——不強制使用者指定方向

**程式碼位置**：[`utils/metric.py:1-78`](https://github.com/wecoai/aideml/blob/40dcf28/aide/utils/metric.py#L1-L78)

**替代方案**：
- **使用負號統一方向**：將所有 metric 轉成「越大越好」（accuracy 保持原值，RMSE 存負值）。簡單但容易忘記轉換，且日誌不直觀
- **外部比較器**：`sort(key=lambda x: x.value, reverse=not x.maximize)`。可行但散布在各處，不易維護
- **直接存原始值 + 手動判斷**：最常見的做法，但 bug-prone

**何時可以借用**：
任何需要**比較多個 metric 值且方向不固定**的系統。特別適合：
- Benchmark 評估框架（不同任務有不同 metric）
- Agent 系統中的 solution evaluation
- Hyperparameter optimization

**注意事項**：
- `WorstMetricValue` 的 sentinel 值設為 `None`，不能參與算術運算（例如取平均）
- 兩個 MetricValue 的 `maximize` 不同時比較會 assertion error——程式會崩潰而不是靜默錯誤，這是刻意的設計

---

## Pattern 4: Multi-process Execution Sandbox with Queue-based Communication

**是什麼**：
AIDE 使用 `multiprocessing.Process`（而非 `subprocess`）來執行 agent 產生的程式碼。主行程與 child process 之間透過三個 `Queue` 通訊：
- `code_inq`：主行程 → child（傳送程式碼）
- `result_outq`：child → 主行程（stdout/stderr 內容）
- `event_outq`：child → 主行程（狀態事件，如 `state:ready`、`state:finished`）

**為什麼有效**：
- 使用 `exec()` 而非 `subprocess.run("python script.py")`——child process 可以直接在主 Python 環境中使用所有已安裝的套件，不需額外 virtualenv 或路徑設定
- Queue-based 設計讓 stdout/stderr 捕獲是**即時**的（non-blocking），不像 subprocess.PIPE 可能會遇到 buffer 阻塞
- 超時控制精確：透過 SIGINT → SIGKILL 兩段式終止，確保 child process 真的結束
- `RedirectQueue` 包裝 Queue 成 file-like object，直接取代 `sys.stdout`——優雅優雅

**程式碼位置**：[`interpreter.py:91-311`](https://github.com/wecoai/aideml/blob/40dcf28/aide/interpreter.py#L91-L311)
- Process creation：[`interpreter.py:169-180`](https://github.com/wecoai/aideml/blob/40dcf28/aide/interpreter.py#L169-L180)
- Code execution loop：[`interpreter.py:133-167`](https://github.com/wecoai/aideml/blob/40dcf28/aide/interpreter.py#L133-L167)
- Timeout handling：[`interpreter.py:249-283`](https://github.com/wecoai/aideml/blob/40dcf28/aide/interpreter.py#L249-L283)

**替代方案**：
- **subprocess.run**：更標準的作法，但 stdout 捕獲是 blocking，且無法在中途檢查 child 狀態
- **Docker container**：隔離性更好（檔案系統隔離），但啟動 overhead 高（數百 ms ~ 數秒），且需要 Docker 安裝
- **Thread-based execution**：輕量，但無法隔離 crash（Python thread 無法被真正 kill），且 GIL 限制

**何時可以借用**：
當你需要執行**不信任或不可預測的 Python 程式碼**，且需要：
- 精確的超時控制
- 完整的 stdout/stderr 捕獲
- 隔離 crash 不影響主行程
- 不需要完整的 container 級隔離

**注意事項**：
- Child process 與主行程共享檔案系統——agent 程式碼可以讀寫主行程可存取的所有檔案
- 沒有 system call 過濾、網路隔離或記憶體限制
- 若主行程已 import 了某個有 state 的套件，child process 不會繼承該 state（每次是全新的 Python 直譯器）
- Windows 相容性：`multiprocessing.Process` 在 Windows 上運作良好但 spawn 成本較高

---

## Pattern 5: Journal Summary as Self-Contained Memory Context

**是什麼**：
`Journal.generate_summary()` 將整個 solution tree 壓縮成一個字串，供後續 LLM 呼叫作為「記憶」上下文。它只包含**好的節點**（非 buggy）的 `plan`、`analysis`、`metric`——不包含完整程式碼。

**為什麼有效**：
- **Token 效率**：不餵完整程式碼給 LLM，只餵計畫與結果摘要。假設每個節點程式碼約 100 行 → 20 節點 = 2000 行程式碼 → token 消耗爆炸。摘要版本只花費~幾百 token
- **Signal > Noise**：LLM 在 context 中有 2000 行程式碼時，不容易判斷「哪一段是重要的」。摘要強迫 LLM 只看到高層次模式
- **自動去重**：若兩個節點的 plan 相同但結果不同，summary 會顯示兩者的 metric 差異，幫助 LLM 判斷哪個做法較好

**程式碼位置**：[`journal.py:182-192`](https://github.com/wecoai/aideml/blob/40dcf28/aide/journal.py#L182-L192)

**替代方案**：
- **完整 history**：每個 LLM call 都帶上所有先前的程式碼與結果。最完整但 token 消耗最大
- **Sliding window**：只保留最近 k 個節點。簡單但可能忘記早期的好想法
- **Vector RAG**：將節點 embedding 存到 vector DB，動態檢索相關節點。最有彈性但複雜度高出一個數量級

**何時可以借用**：
任何需要 agent 在多次迭代中保持「記憶」但 token 預算有限的場景。這是「中間路線」的最佳示範——比 sliding window 保留更多資訊，比 full history 節省更多 token。

**注意事項**：
- 摘要由 LLM 在 evaluation step 產生——也就是 `submit_review` 的 `summary` 欄位。若該 LLM 的摘要品質不好，會影響後續決策
- Summary 不包含程式碼細節——LLM 無法從摘要知道某個節點使用了特定技術（例如 CatBoost vs XGBoost）。這對後續 improve 步驟可能是遺憾
- `generate_summary()` 中 `include_code=False` 是寫死的——只能透過修改原始碼控制

---

## Agent 設計的哲學觀察

AIDE 的作者群顯然對 agent 設計有幾個明確的立場：

1. **「最大化 metric 就是目標」** — 沒有任務規劃、沒有 subgoal decomposition。整個 agent loop 只圍繞一個數值 metric 運轉。這是 ML 自動化任務的合理假設，但不適合通用 agent。
2. **「LLM 應該寫程式碼，不應該呼叫 tool」** — 沒有 tool registry、沒有 function calling（除了評估）。AIDE 認為「寫一個完整的腳本」比「逐步呼叫 tool」更有效——這與 ReAct 陣營的假設直接衝突。
3. **「評估應該結構化」** — 所有非結構化的 LLM 輸出（程式碼生成）最終都靠一個結構化 function call（submit_review）來收束。評估不是靠「覺得 OK」而是靠明確的欄位。
4. **「成功來自 iteration，不是一次完美」** — steps=20、num_drafts=5 的預設值說明了這點。AIDE 假設第一次生成的程式碼不會最好，但第 20 次迭代的結果會顯著優於第 1 次。

## 跟其他 agent 框架比較

| 面向 | AIDE | LangGraph | AutoGen (v0.7) |
|---|---|---|---|
| 核心抽象 | Tree search in code space | 有向圖（StateGraph） | Actor model（SingleThreadedAgentRuntime） |
| Agent 風格 | 寫完整程式碼 → 執行 → 評估 | 節點 + 邊的圖遍歷 | 多 agent 訊息傳遞 |
| Tool calling | 無（直接寫腳本） | ToolNode（標準 function calling） | Tool registry + 封裝 |
| Memory | Journal summary（session-only） | State object（可自訂持久化） | Context（Component-based） |
| 評估方式 | 內建 LLM evaluation loop | 無內建（需手動實作） | 無內建 |
| LLM Provider | 內建 4 個 provider | 透過 LangChain | 自訂 |
| Configuration | OmegaConf (YAML + CLI) | Python DSL | 宣告式 YAML（Component） |
| 適合場景 | AutoML / 競賽型 ML 任務 | 通用 agent workflow | Multi-agent 協作 |

AIDE 是最「任務特定」的一個——它為 ML pipeline 自動化做了深度最佳化，但代價是通用性較低。若你的目標是讓 agent 自動寫 ML 程式碼，AIDE 的架構值得直接參考；若你的目標是通用 agent 系統，AIDE 的 tree search 與 two-LLM 模式可以局部借鑑。
