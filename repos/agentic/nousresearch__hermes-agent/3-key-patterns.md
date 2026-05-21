---
repo: nousresearch/hermes-agent
file: 3-key-patterns
studied_at: 2026-05-21
commit_sha: 48be2e0
---

# Hermes Agent · 值得偷學的設計

## Pattern 1: AST-based 工具發現（而非中央註冊表）

[`tools/registry.py:57-74`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/tools/registry.py#L57-L74)

**是什麼**：不使用中央 enum 或 manifest 檔案宣告哪些工具存在，而是掃描 `tools/` 目錄下的所有 Python 檔案，用 AST 解析找出有呼叫 `registry.register()` 的 module，再 import 它們。

```python
def _is_registry_register_call(node):
    """檢查 AST node 是否為 registry.register(...) 呼叫"""
    return (isinstance(node, ast.Expr) and 
            isinstance(node.value, ast.Call) and
            isinstance(node.value.func, ast.Attribute) and
            node.value.func.attr == "register" and
            isinstance(node.value.func.value, ast.Name) and
            node.value.func.value.id == "registry")

def discover_builtin_tools(tools_dir):
    modules = []
    for path in sorted(tools_dir.glob("*.py")):
        if path.name in EXCLUDES: continue
        if not _module_registers_tools(path): continue  # AST scan
        mod = importlib.import_module(f"tools.{path.stem}")
        modules.append(mod)
    return modules
```

**為什麼有效**：
- **Zero boilerplate** — 新增一個 tool 只需在 `tools/` 下建立 .py 檔案，module-level 呼叫 `registry.register()`，不需要改任何其他檔案
- **Import 副作用最小化** — AST scan 確保只 import 真的有註冊行為的 module，helper/utils module 不會被意外 import
- **編譯時檢查** — module-level 的 AST 檢查在 import 前完成，語法錯誤會在第一時間被捕獲

**程式碼位置**：[`tools/registry.py:29-54`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/tools/registry.py#L29-L54)

**何時可以借用**：
- 任何需要 plugin/tool/handler 自動發現的系統
- 當你希望「在目錄下建立一個檔案」就是註冊一個新功能的唯一步驟
- 適合 module-level side effect 可被接受的場景（非 strict dependency injection）

**注意事項**：
- Module-level side effect 測試困難 — 測試時需注意註冊順序和 cleanup（`registry._tools.clear()`？）
- Global mutable singleton（`registry`）— 多線程下需要 `threading.RLock()`（已實作）
- AST scan 只檢查 module body，不深入 function 內部——這是設計選擇，確保 helper module 不被誤掃

**替代方案**：
- **Decorator-based**：`@registry.register(...)` — 視覺上更清晰，但本質仍是 module-level side effect，且 decorator 執行時機取決於 import 順序
- **YAML manifest**：`tools/manifest.yaml` 列出所有 tool — 明確但需要手動同步，容易遺漏
- **Lazy import**：只 import `__init__.py` 中的 tool names，module 在首次使用時才 import — 節省啟動時間但增加延遲

---

## Pattern 2: Thread-safe 三階段 Async Bridging

[`model_tools.py:42-173`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/model_tools.py#L42-L173)

**是什麼**：因為 conversation loop 是 synchronous 的，但許多 tool handler 是 async 的（需要呼叫 httpx/aiohttp API），需要一個 bridging layer 讓 sync code 能執行 async handler。Hermes Agent 實作了三種路徑，每種有自己的 event loop 生命週期策略：

```python
def _run_async(coro_factory, timeout=300):
    try:
        loop = asyncio.get_running_loop()
        # 路徑 1: 已經有 running loop (gateway/RL)
        return _run_in_new_thread(loop, coro_factory, timeout)
    except RuntimeError:
        # 路徑 2: worker thread → thread-local 持久 loop
        # 路徑 3: main thread (CLI) → 全域共享持久 loop
```

**為什麼有效**：
- 解決了 `asyncio.run()` create-and-destroy 導致的「Event loop is closed」錯誤——cached httpx/AsyncOpenAI 在 GC 時試圖在已關閉的 loop 上 cleanup
- 三路徑覆蓋了 agent 的所有執行場景（CLI、gateway、delegate_task worker）
- 持久 loop 讓 cached HTTP clients 保持有效

**程式碼位置**：[`model_tools.py:47-81`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/model_tools.py#L47-L81)

**何時可以借用**：
- 任何 sync + async 混合的程式碼庫
- 當你無法全面改為 async（legacy codebase、frameworks 不支援）
- 當 cached HTTP clients 需要在不同 async context 間共享

**注意事項**：
- 300s timeout 是寫死的——long-running tool 可能不夠
- Worker thread loop 洩漏風險——timeout 時 `pool.shutdown(wait=False)` 可能留下 zombie loop
- 全域 loop (`_tool_loop`) 不關閉——長時間不使用的工具執行後會留下一個 idle loop

**替代方案**：
- 全面 async：conversation loop 改為 async/await — 更乾淨但需要 CLI framework 支援（prompt_toolkit 支援 async）
- `anyio`/`trio` 統一 event loop：用 single event loop 管理模式，讓 sync 和 async 共享同一 loop
- 每個 tool handler 自帶 timeout：更精細的控制，但每個 handler 都需要手動管理

---

## Pattern 3: 三層 System Prompt 快取架構

[`agent/system_prompt.py:10-22`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/system_prompt.py#L10-L22)

**是什麼**：System prompt 被分為三個層級，分別有不同的更新頻率：

```python
# stable — 身份、guidance、tool 描述  (session 內不變)
# context — system_message + AGENTS.md  (session 開始時計算)
# volatile — memory + timestamp         (每個 turn 重新產生)

"\n\n".join([stable_prompt, context_prompt, volatile_prompt])
```

**為什麼有效**：
- Anthropic prefix cache：stable 層的完全相同 byte string 讓 prefix cache 在整個 session 中保持熱（~90% hit rate）
- Date-only timestamp：只用 `YYYY-MM-DD` 而非 `YYYY-MM-DD HH:MM:SS`，確保同一天的 session 產生完全相同的 system prompt
- Memory snapshot 凍結策略：session 開始時快照一次記憶體──寫入發生在 mid-session，但不會觸發 prompt 重建

**程式碼位置**：[`agent/system_prompt.py:50+`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/system_prompt.py#L50)

**何時可以借用**：
- 使用 Anthropic 或其他支援 prefix caching 的 provider 時
- 高頻 talking agent（同一個 session 內數十到數百個 turn）
- 對 token cost 敏感的應用（prefix cache 可節省 50-90% input tokens）

**注意事項**：
- Mid-session memory 變更不會反映在提示中——模型只能透過 tool call 讀取最新記憶體
- 如果 context layer 依賴 CWD（如 AGENTS.md 掃描），CWD 改變時需要手動重建
- 此設計假設 stable 層在 session 內真的不變——如果 tool registry 動態改變（如 MCP refresh），cache 可能失效

**替代方案**：
- 每次 turn 重建全部：最簡單但無法利用 prefix cache
- 差量更新（delta prompt）：只傳送變化部份——更複雜但更節省，需要 server 端的支援
- 後綴注入（suffix injection）：只在 user message 前加 context，不碰 system prompt——某 provider 較少支援

---

## Pattern 4: Registry 的 Generation Counter 自動無效化

[`tools/registry.py:224-232`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/tools/registry.py#L224-L232)

**是什麼**：Registry 維護一個單調遞增的 `_generation` counter，每次有 tool 註冊/更新時 +1。依賴 registry 的 cache（如 `model_tools.py` 的 `_tool_defs_cache`）將 `_generation` 納入 cache key，自動失效：

```python
class ToolRegistry:
    def register(self, ...):
        ...
        self._generation += 1  # 觸發所有 cache 失效
    
    def get_definitions(self, tools_to_include):
        with self._lock:
            return list(self._tools.values())  # 快照

# 在 model_tools.py 中的 cache key:
cache_key = (
    frozenset(enabled_toolsets),
    frozenset(disabled_toolsets),
    registry._generation,  # ← 自動無效化
    config_file_fingerprint,
)
```

**為什麼有效**：
- Fine-grained 而非全域無效化——`_generation` 讓所有 cache holder 發現變化，但不需要事件廣播
- Thread-safe——`_generation` 在 `_lock` 保護下遞增，讀取時不需要鎖（Python GIL 保證 int read atomic）
- 無需手動管理 cache invalidation——register() 的自動副作用

**程式碼位置**：[`tools/registry.py:224-232`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/tools/registry.py#L224-L232)

**何時可以借用**：
- 任何有「中央資料 + 多個快取消費者」的場景
- Plugin/MCP 動態載入需要通知所有 cache 失效的場景
- 不想引入事件匯流排或 observer pattern 的簡單場景

**注意事項**：
- 如果 cache 在 registry 更新時正在生成，可能拿到半新半舊的資料——`_lock` 在 register() 內是 `threading.RLock()`，但 cache 消費者可能已在 lock 外讀了 `_generation`
- Global counter 在每次註冊時都 bump——頻繁的 MCP refresh 會導致 cache 不停失效
- Cache 依賴者需要知道註冊 `_generation` 的存在，這是隱式耦合

**替代方案**：
- **Event bus**：register() 發出 `tools.changed` 事件，cache 監聽——更解耦但更重
- **Immutable registry + copy-on-write**：讀取無鎖，但 GC 壓力大（每次寫入複製整個 registry）
- **TTL-based cache**：固定時間後自動失效——簡單但延遲不確定

---

## Pattern 5: Skill 的 Conditional Activation（執行環境感知）

[`agent/prompt_builder.py:966-994`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/prompt_builder.py#L966-L994)

**是什麼**：不是所有 skill 都應該顯示給所有 agent 實例。Skill 的 frontmatter 可以宣告執行條件，讓 system prompt 中的技能索引只包含符合當前環境的 skill：

```yaml
metadata:
  hermes:
    fallback_for_toolsets: [browser]   # 當 browser toolset 可用時隱藏此 skill
    requires_toolsets: [terminal]       # 沒有 terminal toolset 時隱藏
    fallback_for_tools: [web_search]   # 當 web_search tool 可用時隱藏
    requires_tools: [execute_code]     # 沒有 execute_code 時隱藏
```

**為什麼有效**：
- 減少 system prompt 的長度——不相關的 skill 不會被加到索引中
- 避免「同一功能有 skill 版本和 built-in tool 版本」的衝突（`fallback_for_toolsets`/`fallback_for_tools`）
- 讓 skill 作者宣告自己的執行環境需求（如 `requires_toolsets: [terminal]`）

**程式碼位置**：[`agent/prompt_builder.py:966-994`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/prompt_builder.py#L966-L994)

**何時可以借用**：
- 你的 skill/plugin 系統有「此 skill 需要特定運行環境」的需求
- 同一個 ecosystem 中同時有「built-in 實作」和「skill 實作」的同一功能
- 想要在系統提示中動態過濾內容，減少 token 消耗

**注意事項**：
- 條件檢查發生在 `build_skills_system_prompt()` 中，不會即時反映 toolset 的 runtime 變化
- `fallback_for_tools` 需要 tool name 的精確匹配——如果內建工具被改名，condition 會失效
- 條件是「全部符合才顯示」還是「任一符合就顯示」？目前的實作是 all-of（requires）和 any-of（fallback_for）

**替代方案**：
- **純手動管理**：使用者在 config.yaml 中列出要啟用的 skill——最簡單但人為負擔大
- **LLM 自行判斷**：把條件描述放在 system prompt 中，讓模型自己決定要不要載入——更靈活但浪費 token
- **Hot-reload**：在 toolset 切換時重新觸發 skill indexing——即時但複雜

---

## Pattern 6: Iteration Budget 的 Grace Call 機制

[`agent/iteration_budget.py:17-62`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/iteration_budget.py#L17-L62)、[`agent/conversation_loop.py:617-618`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/conversation_loop.py#L617-L618)

**是什麼**：當 IterationBudget 耗盡時（`max_iterations=90`），不直接中止 agent，而是注入一次「這是最後一次 API call，請總結」的機會（grace call）：

```python
# 在 loop 頂部的 budget 檢查
if not self.iteration_budget.consume():
    if self._budget_grace_call:
        self._budget_grace_call = False  # 只用一次
        # 注入「請總結」訊息
        messages.append({"role": "user", "content": BUDGET_EXHAUSTED_TEXT})
    else:
        break  # grace call 也用完了
```

**為什麼有效**：
- 避免「正在關鍵工作時突然被切斷」——模型有機會優雅地結束正在進行的工作
- 注入的「請總結」訊息會觸發模型的總結行為，確保不會遺失已完成的工具結果
- 一次性的設計確保耗盡後一定結束——不會無限 grace

**程式碼位置**：[`agent/conversation_loop.py:424-431`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/conversation_loop.py#L424-L431)

**何時可以借用**：
- 任何有 iteration/token/resource 上限的 agent 系統
- 當「優雅終止」比「嚴格截斷」更重要時

**注意事項**：
- 不支援中間壓力警告——只有耗盡時一次通知（issue #7915 指出太多警告導致模型提前放棄）
- Grace call 被誤用時（模型在 grace call 中又發出 tool call），`_handle_max_iterations()` 會強制阻擋 tool calls（line 3797-3848）
- `execute_code` 有退款機制（`refund()`）——當批次只有一個工具且是 `execute_code` 時退款，不消耗 budget

**替代方案**：
- **比例式警告**：70%、85%、95% 時分別警告——精細但可能干擾
- **無限制**：相信模型自己判斷何時結束——風險高、不可控
- **Hard cap + immediate abort**：簡單粗暴但體驗差

---

## Pattern 7: Gateway 的 LRU Agent 池（而非每個訊息新建 Agent）

[`gateway/run.py`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/gateway/run.py)、[`gateway/session.py`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/gateway/session.py)

**是什麼**：Gateway 模式不為每條訊息建立新的 AIAgent 實例，而是維護一個 LRU cache（cap=128, TTL=1h），用 `build_session_key()` 產生的唯一 key 映射到 agent 實例：

```python
class AgentPool:
    def __init__(self, max_size=128, ttl=3600):
        self._cache = OrderedDict()  # session_key → (agent, last_used)
    
    def get_or_create(self, session_key, factory):
        if session_key in self._cache:
            self._cache.move_to_end(session_key)  # LRU update
            return self._cache[session_key]
        if len(self._cache) >= self.max_size:
            self._cache.popitem(last=False)  # 踢除最舊
        agent = factory()  # 建立新 AIAgent
        self._cache[session_key] = agent
        return agent
```

**為什麼有效**：
- 避免每次訊息都重建 AIAgent（~60 個參數的初始化成本很高）
- 保持 agent 的 conversation history 在記憶體中，不需要每次從 SQLite 恢復
- LRU + TTL 防止無限增長——不活躍的 session 自動被踢除

**程式碼位置**：[`gateway/run.py`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/gateway/run.py)

**何時可以借用**：
- 任何多用戶、多 session 的 agent 服務
- 當 AIAgent/LLM client 初始化成本高時（需要建立 HTTP client、load skills、restore memory）
- 希望保持多個 session 的 conversation history 在記憶體中

**注意事項**：
- 128 cap 需要根據用戶量和記憶體調整——每個 AIAgent 持有一個完整的 messages list
- 1h TTL 代表 session 在 1 小時無活動後自動回收——長期不活躍的 session 下次需要從 SQLite 恢復
- LRU 並不提供隔離——不同用戶的 session 共享同一 pool，大量活躍用戶會擠壓其他人的 session

**替代方案**：
- **每秒鐘新建 agent**：簡單但昂貴，無法利用 prefix cache
- **獨立的 agent 進程**：每個 session 一個 process 或 container——完全隔離但資源消耗大
- **無狀態 agent + 每次從 DB 恢復**：不需要記憶體 cache，但每次都需 SQLite read + prompt rebuild

## Agent 設計的哲學觀察

Hermes Agent 的設計反映了 Nous Research 對 agent 的幾個明確立場：

1. **信任模型**：沒有複雜的護欄系統（guardrails 非常輕量）、沒有 tool 權限控制、沒有 prompt injection 防護。設計者相信（或實務證明）當代 model 足夠可靠，過度防護帶來的 false positive 比 security risk 更糟。

2. **Imperative 勝於 Declarative**：不是 graph、不是 state machine、不是 workflow engine —— 就是一個巨大的 while loop。這表示設計者認為 agent loop 的複雜度來自於邊界情況（錯誤處理、重試、壓縮、中斷），而不是控制流結構。Declarative graph 在這方面沒有優勢。

3. **Plugins over Inheritance**：幾乎所有擴充點都是 plugin-based（memory providers、model providers、gateway platforms、observability），而不是 subclassing。這讓第三方貢獻者不需要理解龐大的 class hierarchy。

4. **SQLite 至上**：Session 資料、kanban board、記憶體 —— 全部 SQLite。沒有 Redis、沒有 PostgreSQL、沒有專屬 storage service。這讓整個系統可以裝在一台機器上執行（甚至沒有網路），但也明確放棄了水平擴展的可能性。

## 跟其他 agent 框架比較

| 面向 | Hermes Agent | LangGraph | Claude Code |
|---|---|---|---|
| Loop 實作 | Synchronous imperative | Declarative graph | Closed source |
| Tool 註冊 | Self-registering (AST) | Node-level | Built-in |
| Memory 持久化 | SQLite FTS5 | User-managed | JSONL |
| 平台支援 | CLI + 40+ messaging | Python library only | CLI only |
| Self-improve | Skill system | 無 | 無 |
| 程式碼規模 | ~142K LOC (1789 files) | ~50K LOC | N/A (closed) |
| 主要 trade-off | 功能完整但學習曲線高 | 靈活但需組裝 | 簡單但封閉 |
