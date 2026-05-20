
# Agent State Machine

> 把 agent 的執行流程顯式建模為有限狀態機或有向圖,而不是隱含在一個 while loop 裡。

## 它解決什麼問題

傳統的 agent 實作常常長這樣:

```python
while not done:
    response = llm.call(messages)
    if response.has_tool_call:
        result = execute_tool(response.tool_call)
        messages.append(result)
    else:
        done = True
        return response.content
```

這種寫法問題很多:

- **難以中斷與恢復** — state 散落在 local variables,沒辦法序列化讓 agent 跑一半暫停、之後續跑
- **難以加入分支邏輯** — 想加「先檢查 cache 再決定要不要呼叫 LLM」,要動到主迴圈
- **難以平行化** — 兩個獨立的子任務想並行,需要重寫整個迴圈
- **難以視覺化與除錯** — 流程藏在控制流裡,新人看不出 agent 會做什麼
- **難以測試** — 每個分支都得跑完整迴圈才能驗證

State machine pattern 把上述每個「狀態」變成顯式的節點,節點間的轉移由邏輯決定,而不是程式碼流程決定。

## 核心想法

三個關鍵抽象:

1. **State** — agent 在任一時刻的完整狀態,可序列化。包含 messages、scratchpad、工具呼叫紀錄等。
2. **Node** — 對 state 做一次轉換的單元函式。例如 `call_llm`、`execute_tool`、`summarize`。
3. **Edge** — 從一個 node 結束後跳到下一個 node 的規則。可以是靜態的(`A → B`)或條件的(`A → B if X else C`)。

執行迴圈不再是 hardcoded 的 while,而是:

```
current_state = initial_state
while True:
    next_node = graph.decide_next(current_state)
    if next_node is END:
        break
    current_state = next_node.run(current_state)
```

關鍵性質:

- State 在每一步都可以被 dump / restore,實現持久化與時間旅行除錯
- Graph 結構在執行前就確定,可以靜態分析、視覺化、套用最佳化
- Nodes 是純函式 `(state) → state`,可單獨測試與替換

## 觀察到的變體

| 變體 | 怎麼運作 | 代表實作 |
|---|---|---|
| **顯式 graph** | 使用者直接宣告 nodes 跟 edges | LangGraph |
| **Process / Task** | 把 graph 包成一個 high-level task,內部 node 由框架決定 | CrewAI |
| **Code as graph** | 使用者寫一般 Python,框架透過追蹤/裝飾器自動建構 graph | Burr、部分 DSPy |
| **YAML 宣告** | 整個 graph 寫在配置檔,程式碼只負責執行 | 部分企業內部框架 |

## 跟替代方案的對照

| 方案 | 優點 | 缺點 |
|---|---|---|
| **裸 while loop** | 簡單、學習成本低 | 狀態散亂、難擴充 |
| **State machine** | 顯式、可序列化、易視覺化 | 概念負擔較重、簡單任務顯得 over-engineering |
| **Actor model** | 適合 multi-agent、天然平行 | 需要訊息協定設計、除錯複雜 |
| **DAG (像 Airflow)** | 平行化強、結構清晰 | 缺乏迴圈(many agents 本質需要迴圈) |

State machine 的最佳位置是「比 while loop 複雜、但又用不到 actor model 的程度」——大多數 production agent 都落在這個區間。

## 何時用 / 何時不用

**用 state machine 當**:

- Agent 需要支援中斷續跑(例如使用者離開又回來、長任務需要人工介入)
- 有多種分支邏輯(不同 tool 結果走不同路徑)
- 需要對 agent 流程做視覺化或審計
- 多人協作,需要對 agent 行為有共同的 mental model
- 計畫加入 human-in-the-loop checkpoint

**不要用當**:

- Agent 邏輯極簡(就是「LLM call + 一兩個 tool」)——加 graph 抽象只是徒增噪音
- 是純 chatbot,沒有複雜 tool 編排
- 團隊還在 prototype 階段——先用 while loop 跑通再考慮重構
- 流程本身就是 DAG 而非迴圈——直接用 DAG 框架更合適

## 在不同 repo 的實作

### LangGraph

最完整的顯式 graph 實作。`StateGraph` 類別讓你註冊 nodes、宣告 edges、編譯成可執行的 graph。

- 核心 graph 結構: [`langgraph/graph/state.py`](https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/graph/state.py) `[UNVERIFIED 確切行號]`
- 設計亮點: state 用 TypedDict 定義,每個 node 回傳的 dict 會 merge 進 state,支援自訂 reducer 函式做精細控制
- 特殊功能: `interrupt()` 機制可以在任何 node 暫停執行等待外部輸入

### CrewAI

抽象層次更高的變體。使用者不直接定義 graph,而是定義 Agents 跟 Tasks,框架內部把它們組織成執行流程。

- 核心執行邏輯: `crewai/crew.py` `[UNVERIFIED]`
- 設計亮點: 把 multi-agent 協作的 boilerplate(訊息傳遞、結果聚合)隱藏在抽象後面
- Trade-off: 比 LangGraph 容易上手,但客製化空間較小

### Hermes Agent

採用較傳統的 loop-based 實作,但加上了 skill system 跟 trajectory compression,某種程度上是「以資料為中心」而非「以 graph 為中心」的不同思路。

- 主 agent loop: `agent/` 目錄 `[UNVERIFIED 具體檔案]`
- 對照價值: 展示了不用顯式 state machine 也能做到大部分能力的路線,適合對照思考 graph 抽象的必要性

> `[TODO]` 補充第 4-5 個觀察 repo:Burr、DSPy 的執行模型、AutoGen GraphFlow

## 我的建議

如果你正在設計一個新的 agent 系統:

1. **不要一開始就上 state machine** — 先用一個 while loop 把流程跑通,確認真的需要的功能是什麼
2. **痛到了再抽象** — 當你發現你在 while loop 裡寫 `if state == "X"` 超過 3 次,就是時候改寫成 graph
3. **State 的 schema 比 graph 結構更重要** — 80% 的設計決策應該花在「state 裡放什麼」,而不是「graph 怎麼連」
4. **保留逃生艙口** — 設計 graph 框架時,允許某個 node 內部跑一個小的 while loop。不要強迫所有事情都用節點表達
5. **視覺化能力是落地關鍵** — 如果 graph 不能被畫出來給非工程師看,它對團隊的價值會被低估

最後一個觀察:**很多 agent 框架的 graph 抽象都過度設計**。實際 production agent 往往只有 5-10 個 node、結構接近線性。如果你發現自己在框架的 abstraction 上花的時間比寫 node 邏輯還多,可能要回頭檢查方向是否走偏。
