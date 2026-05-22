---
repo: microsoft/autogen
file: 3-key-patterns
studied_at: 2026-05-22
commit_sha: 027ecf0
---

# AutoGen · 值得偷學的設計

## Pattern 1: Component 系統 — Declarative Agent Config

**是什麼**: 用 `ComponentModel`（一個 Pydantic model）描述一個 agent 系統的所有元件，透過 `provider` + `config` 兩個欄位即可實例化任何元件。整個系統可以被序列化為 YAML/JSON。

**程式碼位置**: [`autogen_core/_component_config.py`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-core/src/autogen_core/_component_config.py)

**為什麼有效**: 

傳統的 agent 框架中，開發者要透過程式碼組合所有元件（建立 model client、建立 tool、建立 agent、組成 team）。Component 系統讓這個組合過程可以用資料驅動：

```yaml
provider: autogen_agentchat.agents.AssistantAgent
config:
  name: assistant
  model_client:
    provider: autogen_ext.models.openai.OpenAIChatCompletionClient
    config:
      model: gpt-4o
  tools:
    - provider: autogen_ext.tools.http.HttpTool
      config:
        url: "https://api.weather.com"
```

**替代方案**:

| 方案 | 比較 |
|---|---|
| LangGraph 的 config | 沒有內建 declarative 序列化，需自製 |
| FastAPI + Pydantic settings | 只處理 config，不處理元件的命名空間與動態載入 |
| Kubernetes CRD | 更通用但也更複雜，且不綁定 Python/Pydantic |

**何時可以借用**:

任何需要支援「非開發者也能配置 agent 系統」的場景，例如 AutoGen Studio 這類 GUI 工具。如果你正在做一個 agent evaluation 平台、或一個 nocode agent builder，Component 系統是很好的設計參考。

**注意事項**:

- Provider namespace 的安全限制（`_TRUSTED_PROVIDER_NAMESPACES`）雖然保護了安全性，但也增加了第三方 plugin 的整合門檻。
- `ComponentModel` 使用 Pydantic model 做 config schema validation，這意味著「開發者寫的 Pydantic model」和「使用者給的 config dict」需要保持同步 — 這是維護挑戰。

---

## Pattern 2: Agent 的 Stateful 設計 — 只傳新訊息

**是什麼**: AutoGen 的 `ChatAgent.on_messages()` 只接收「新訊息」，agent 內部維護自己的 state（透過 `model_context` 和 instance variables）。這跟傳統的每次都傳完整 conversation history 的做法不同。

**程式碼位置**: [`BaseChatAgent.on_messages()`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-agentchat/src/autogen_agentchat/agents/_base_chat_agent.py#L72) 與 [`AssistantAgent.on_messages()`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py)

```python
# 使用者只傳新訊息
response = await agent.on_messages(
    messages=[TextMessage(content="What's the weather?", source="user")],
    cancellation_token=cancellation_token,
)
# agent 內部維護 history（透過 self._model_context）
```

**為什麼有效**:

1. 減少資料傳輸 — Team 不需要每次都重新序列化整個 history
2. 降低耦合 — Agent 可以選擇自己的 context 管理策略（buffer / token limit / summarization）
3. 支援暫停恢復 — Agent 的 state 可以透過 `save_state()` / `load_state()` 序列化

**替代方案**:

| 方案 | 比較 |
|---|---|
| 傳統 conversation history（每次傳完整 list） | 簡單直接，但無法優化 context window |
| LangGraph's StateGraph（全域 state + reducers） | state 在 graph 層級，agent 間共享但需要 reducer 函式 |
| AutoGen 的 agent-local state | 每個 agent 獨立管理 state，通訊走 message passing |

**何時可以借用**:

當你的 system 有多個 stateful agent，且 agent 間的 context 是隔離的（不像 LangGraph 那樣共用一個全域 state）。

**注意事項**:

- caller 必須保證每次呼叫都傳遞正確的「增量」，這對複雜的分支流程會增加心智負擔
- 如果一個 agent 需要知道其他 agent 說了什麼，它不能直接讀取別人的 state，必須透過 message passing 獲得資訊

---

## Pattern 3: Topic-based Message Routing 搭配 Subscription

**是什麼**: AutoGen 使用 publish/subscribe 模式做 agent 間的通訊。Agent 可以 `publish_message()` 到一個 `TopicId`，runtime 的 `SubscriptionManager` 會根據註冊的 subscription 找到所有應該收到該訊息的 agent。

**程式碼位置**: [`_single_threaded_agent_runtime.py` 的 PublishMessageEnvelope](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py)

```python
# 發送端
await runtime.publish_message(
    message=MyMessage(content="hello"),
    topic_id=TopicId(type="chat", source="default"),
)

# 接收端（透過裝飾器註冊）
@default_subscription
class MyAgent(RoutedAgent):
    @event
    async def on_my_message(self, message: MyMessage, ctx: MessageContext) -> None:
        print(f"Received: {message.content}")
```

**為什麼有效**:

1. 鬆散耦合 — publisher 不知道誰會收到訊息，subscriber 不知道訊息從哪來
2. 動態彈性 — Agent 可以在 runtime 註冊/取消註冊 subscription，改變系統拓樸
3. 原生支援廣播場景 — 一個訊息可以被多個 agent 同時處理

**替代方案**:

| 方案 | 比較 |
|---|---|
| Direct message passing（點對點 method call） | 簡單但耦合度高，擴展 agent 需改多處 |
| LangGraph 的 edge-based routing | 靜態 routing 圖，所有路徑在編譯前決定 |
| AutoGen 的 topic-based | 動態 subscription，agent 可自由加入/退出 topic |

**何時可以借用**:

- 當 agent 數量不固定（例如動態加入/離開）
- 當需要廣播場景（一個事件觸發多個 agent 的處理）
- 當開發者希望降低 agent 間的耦合

---

## Pattern 4: Tool 的 Function-as-a-Service 抽象

**是什麼**: `FunctionTool` 可以從任何 Python function（sync 或 async）自動生成一個 tool，包括 JSON schema 推導、參數驗證、錯誤處理。

**程式碼位置**: [`autogen_core/tools/_function_tool.py`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-core/src/autogen_core/tools/_function_tool.py)

```python
from autogen_core.tools import FunctionTool

async def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    result = await weather_api.get(city)
    return f"Weather in {city}: {result.temperature}°C"

tool = FunctionTool(get_weather, description="Get weather info")
# FunctionTool 自動從 type hint 推導 JSON schema
```

**為什麼有效**:

- type hint → JSON schema 的自動轉換省了大量 boilerplate
- 支援 sync 和 async 函式（透過 `is_coroutine` 判斷）
- `BaseToolWithState` 支援有 state 的 tool
- 錯誤處理統一包裝在 `ToolResult` 中

**替代方案**:

| 方案 | 比較 |
|---|---|
| LangChain's @tool decorator | 功能類似，但綁定 LangChain 生態 |
| 手寫 Pydantic model + function | 需要手動維護 schema 與實作的一致性 |
| OpenAI function calling 規格 | 純 JSON schema，不處理執行 |

---

## Pattern 5: ChatCompletionClient — 供應商中立的 Model 抽象

**是什麼**: `ChatCompletionClient` 是一個 abstract protocol，定義 model inference 的標準化介面。所有 provider（OpenAI、Azure、Ollama、Gemini、llama.cpp）都實作這個介面，切換 provider 只需換 class。

**程式碼位置**: [`autogen_core/models/_model_client.py`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-core/src/autogen_core/models/_model_client.py)

**為什麼有效**: 相較於直接硬綁定 OpenAI SDK，AutoGen 的抽象層讓開發者可以在 local LLM（Ollama/llama.cpp）、雲端 LLM（OpenAI/Azure）、或測試 mock 之間無痛切換。

**替代方案**:

- LangChain 的 ChatModel — 生態最大，但抽象層過厚
- LiteLLM — 專注於 provider 路由，不綁定框架
- 自製 adapter — 最輕量但需要重複發明輪子

**何時可以用**: 任何需要支援多個 LLM provider 的 agent 系統。

---

## Agent 設計的哲學觀察

AutoGen v0.7 的設計者對 agent 系統有幾個明確的信念：

1. **Agent 是 actor，不是 graph** — 每個 agent 是獨立的 message handler，不是 graph 中的一個節點。選擇 actor model 意味著他們更重視並行性和隔離性，而不是可視性
2. **配置應該是資料，不應該是程式** — Component 系統是整個 v0.7 最核心也最用心的設計。其他 agent 框架很少花這麼多力氣在 declarative config 上，這大概也跟 AutoGen Studio 的需求有關
3. **跨語言是第一等需求** — 同時維護 Python SDK 和 .NET SDK，加上 protobuf 定義的 gRPC 通訊協定，這在 agent 生態系中很罕見。這反映 Microsoft 的企業客戶有混合語言 stack
4. **Context 隔離優先於 Context 共享** — 每個 agent 維護自己的 context，不通訊就不共享資訊。這跟 LangGraph 的 design philosophy 幾乎相反

## 跟其他 agent 框架比較

| 面向 | AutoGen (v0.7) | LangGraph | CrewAI |
|---|---|---|---|
| 底層模型 | Actor model (message passing) | StateGraph (state machine) | Task pipeline |
| 配置方式 | Python API + YAML/JSON declarative | Python API only | Python API only |
| State 管理 | Per-agent JSON serializable | Global StateGraph state | 任務輸出累積 |
| 並行 | Actor-level concurrency | Node-level (有限) | 任務級 |
| 跨語言 | Python + .NET + gRPC | Python only | Python only |
| 序列化 | 內建 Component 系統 | 需自製 | 有限 |
| GUI 工具 | AutoGen Studio（內建） | LangSmith（外部） | 無 |
| 學習曲線 | 中等（actor model 需理解） | 高（graph + reducers） | 低 |
