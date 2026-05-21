---
repo: hkuds/nanobot
file: 3-key-patterns
---

# Nanobot · 值得偷學的設計

## Pattern 1: TurnState 顯式狀態機驅動 Agent Loop

**是什麼**：
Agent loop 不是用單一的 `while think() → act()` loop，而是用 `TurnState` enum + `_TRANSITIONS` dict 定義的顯式狀態機。每個狀態有獨立的 handler method，狀態轉換由 event string 驅動。

**為什麼有效**：
- **Traceability**：每個 turn 產生 `StateTraceEntry` 記錄每個階段的耗時與錯誤，方便 debug「agent 在哪個階段卡住」
- **Composability**：新功能可以透過新增狀態來加入，不需要修改主 loop 邏輯（例如 `/goal` 功能可以插入 StateTrace）
- **Testability**：每個狀態 handler 可以獨立測試

**程式碼位置**：[`nanobot/agent/loop.py:63-158`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/loop.py#L63)

**何時可以借用**：
當你的 agent loop 需要處理多個階段（載入→檢查→建構→執行→儲存→回應），且每個階段的邏輯足夠獨立時。不適合極簡的 3-step ReAct（單一 `async def run()` 就夠了）。

**替代方案**：
- **單一 `run()` 方法**（Hermes Agent）：更簡單，但添加 hook 需要侵入式修改
- **Graph-based**（LangGraph）：更強大但更複雜，適合 DAG 而非線性狀態機
- **Event-driven**（有限狀態機 + event bus）：更靈活但更難 trace

**注意事項**：
狀態機的 driver loop（`loop.py:789` 的 `while self._running`）跟狀態轉換表的 driver（`dispatch` 內的 `_on_RESTORE()`→`_on_COMPACT()`→...）是分離的。確保 driver loop 不繞過狀態機的 entry/exit point。

---

## Pattern 2: MessageBus 以 asyncio.Queue 解耦頻道與 Agent

**是什麼**：
頻道（Telegram、Discord、CLI）和 agent core 之間不直接呼叫，而是透過 `asyncio.Queue` 組成的 `MessageBus` 進行非同步訊息傳遞。

**為什麼有效**：
- **Zero dependency**：頻道不需要知道 agent 的存在，agent 不需要知道頻道的存在——它們只跟 `MessageBus` 合約互動
- **Backpressure native**：`asyncio.Queue(maxsize)` 自然提供背壓，queue 滿時 publish 會 blocking
- **測試友善**：測試頻道時只要 mock MessageBus，測試 agent 時只要產生 `InboundMessage`

**程式碼位置**：[`nanobot/bus/queue.py:8`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/bus/queue.py#L8)

**何時可以借用**：
任何需要解耦資料生產者與消費者的場景。特別適合 single-consumer、multi-producer 的架構。

**替代方案**：
- **Callback/Handler 模式**（較常見）：頻道持有 agent 的 reference，直接呼叫方法——更緊密耦合但 latency 更低
- **Pub/Sub Event Bus**（如 Hermes Agent）：支援 multiple consumers、dynamic subscription——但需要 event type registry 和 dispatcher
- **Message Queue**（如 RabbitMQ、Redis Queue）：持久化、跨行程——但對 in-process agent 來說太重

**注意事項**：
`asyncio.Queue` 不會持久化——如果 agent 掛掉，queue 裡的訊息會遺失。Nanobot 透過 session 的原子寫入（`memory.py` 的 `temp file + fsync + rename`）來補償這點，但 MessageBus 本身不保證 delivery。

---

## Pattern 3: Multi-Layer Context Governance Pipeline

**是什麼**：
每次 LLM call 前，messages 依序通過五層處理：drop orphan tool results → backfill missing tool results → microcompact → apply tool result budget → snip history。每層處理一個特定的 context pollution 問題。

**為什麼有效**：
- **Defense in depth**：沒有一層能處理所有 edge case——microcompact 處理內容過長，snip 處理 token budget overrun，兩者一起確保
- **Stateless & composable**：每層是 pure function（或接近 pure），容易測試、容易重排
- **Double cleanup**：`_drop_orphan_tool_results` 在 pipeline 首尾各跑一次，因為 snip 可能創造新的 orphan（被裁剪的 tool_call 還留下 tool result）

**程式碼位置**：[`nanobot/agent/runner.py:273-280`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/runner.py#L273)

**何時可以借用**：
任何需要管控 LLM context 的 agent 系統。重點是「分層」而非「一塊大函式做所有事」。

**替代方案**：
- **單一 token counting + 裁剪**：簡單但無法處理 tool result 內部的冗餘
- **Summarization**（由 LLM 總結舊訊息）：更智慧但更慢、更貴——Nanobot 把這層留給 Consolidator 做背景處理
- **Context window 內完全信任**（不做治理）：適用於 context window 夠大且不擔心 pollution 的情境

**注意事項**：
Pipeline 的順序很重要。`_microcompact` 應該在 `_snip_history` 之前——如果先 snip 再 microcompact，已裁掉的訊息就沒有 compact 機會了。`_drop_orphan_tool_results` 需要在首尾各跑一次，因為中間的步驟可能創造新的 orphan。

---

## Pattern 4: Dream 兩階段非同步記憶處理

**是什麼**：
Nanobot 的長期記憶系統 `Dream` 分兩個階段在 background 非同步執行：第一階段 (`learn()`) 從新 history entries 提取事實並寫入 `MEMORY.md`；第二階段 (`dream()`) 跨條目交叉比對，識別矛盾並修正。兩個階段都依賴 LLM，但透過 background task 不阻塞主 agent loop。

**為什麼有效**：
- **Non-blocking**：記憶處理是 I/O-bound（LLM call）但不需要即時結果，background 執行不影響 user-facing latency
- **Cursor-based incremental**：透過 `.dream_cursor` 追蹤已處理到哪個 history entry，增量處理而非每次重跑
- **Git-versioned**：記憶檔案（`MEMORY.md`、`SOUL.md`、`USER.md`）透過 `GitStore` 做版本控制，可以 rollback

**程式碼位置**：[`nanobot/agent/memory.py`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/memory.py) — Dream class

**何時可以借用**：
當你的 agent 需要長期記憶但不想犧牲響應速度時。

**替代方案**：
- **Vector DB 檢索**：更靈活（可以 semantic search），但需要外部服務依賴
- **KV store + 直接寫入**：更簡單但缺乏 cross-entry 比對
- **每次 turn 前全量重新整理**：保證最新但成本高、latency 高

**注意事項**：
Dream 依賴 LLM 來「理解」history entries，這意味著：
- LLM 的品質直接影響記憶品質
- LLM call 需要 token/cost 預算
- 第一階段寫入 `MEMORY.md` 的內容會在下一次 system prompt 組裝時被讀取，所以 Dream 的影響有延遲

---

## Pattern 5: Auto-Discovery Plugin 架構（Tools / Channels / Providers）

**是什麼**：
Tools、channels、providers 都透過 `pkgutil.iter_modules` 自動掃描 package 子模組來發現，而非手寫註冊清單。第三方可以透過 `package.entry_points` 註冊 plugin。

```python
# ToolLoader.discover() — 自動發現所有 Tool subclass
for _importer, module_name, _ispkg in pkgutil.iter_modules(self._package.__path__):
    module = importlib.import_module(f".{module_name}", self._package.__name__)
    for attr_name in dir(module):
        attr = getattr(module, attr_name)
        if isinstance(attr, type) and issubclass(attr, Tool) and attr is not Tool:
            registry.register(attr.create(ctx))
```

**為什麼有效**：
- **Zero registration overhead**：新增一個 tool 只要在 `nanobot/agent/tools/` 下新增一個 `.py` 檔案即可，不需要修改註冊清單
- **Plugin 與 built-in 平等**：第三方 plugin 透過同一個 `register()` 路徑，語意相同
- **Conditional loading**：每個 tool 的 `enabled(ctx)` classmethod 可以根據 context 決定是否載入

**程式碼位置**：[`nanobot/agent/tools/loader.py:30`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/tools/loader.py#L30)

**何時可以借用**：
任何需要動態擴展的 plugin 系統——前提是你的 plugin 數量不會超過數十個（大量 plugin 時 import overhead 會累積）。

**替代方案**：
- **手寫 registry**（`工具清單 = [...]`）：顯式但需要每次新增時修改清單
- **Decorator-based**（`@tool.register`）：更 Pythonic，但需要確保 decorator 在 import 時被執行
- **Entry-point 插件**（純 `importlib.metadata.entry_points`）：對外部 plugin 好，但 built-in 仍需手動註冊

**注意事項**：
`_SKIP_MODULES`（`loader.py:14`）列出不應掃描的 module（base, schema, registry, context, loader, config, file_state, sandbox, mcp, runtime_state）。當新增非 tool module 進 tools package 時，記得更新這個集合。

---

## Pattern 6: Atomic File Writes with fsync + Rename

**是什麼**：
Nanobot 對 `history.jsonl` 的寫入使用「寫入 temp file → fsync → rename → fsync directory」的原子寫入模式，確保任何 crash 不會導致檔案損毀。

```python
# nanobot/agent/memory.py:367-390
tmp_path = self.history_file.with_suffix(self.history_file.suffix + ".tmp")
with open(tmp_path, "w", encoding="utf-8") as f:
    f.write(...)
    f.flush()
    os.fsync(f.fileno())
os.replace(tmp_path, self.history_file)
with suppress(PermissionError):
    fd = os.open(str(self.history_file.parent), os.O_RDONLY)
    os.fsync(fd)
    os.close(fd)
```

**為什麼有效**：
- **Crash-safe**：`os.replace`（POSIX 的 `rename()`）在檔案系統層級是原子的——要么舊檔完整、要么新檔完整，不會有部分寫入
- **Directory fsync**：確保 rename 的 metadata 也被寫入磁碟（避免 power loss 後 rename 沒生效）
- **Windows 相容**：Windows 不支援 `os.open(O_RDONLY)` on directory，用 `suppress(PermissionError)` 優雅跳過

**程式碼位置**：[`nanobot/agent/memory.py:367-391`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/memory.py#L367)

**何時可以借用**：
任何需要持久化重要資料但不想引入資料庫的情境。

**替代方案**：
- **直接 `open("w")`**：最快但 crash 時可能留下截斷檔案
- **資料庫（SQLite）**：更安全但引入外部依賴
- **Write-ahead log**：更複雜但更高效能

**注意事項**：
這個模式只在單一 writer 的情境下安全。如果有多個行程同時寫同一個檔案，需要 file locking 機制。

---

## Agent 設計的哲學觀察

Nanobot 的設計哲學在 `.agent/design.md` 中有清楚表述：

1. **核心小，邊緣延伸**：`agent/loop.py` 和 `agent/runner.py` 是 critical path，新功能應加到 channels、tools、skills、或 MCP servers
2. **少結構，多智慧**：偏好簡單可讀的程式碼勝過 framework 層層包裝
3. **重複好過過早抽象**：channels 和 providers 允許重複類似邏輯，不要為了消除重複而引入複雜的 base class
4. **顯式勝過魔法**：config 必須在 `schema.py` 中透過 Pydantic 明確宣告，錯誤處理應拋出清楚異常

這跟 LangGraph 的設計哲學（graph as first-class abstraction）形成對比——Nanobot 更接近「Pythonic utility library」而非「framework」。

## 跟其他 agent 框架比較

### Tool 註冊方式

| 面向 | Nanobot | Hermes Agent | LangGraph |
|------|---------|-------------|-----------|
| 註冊機制 | `pkgutil` 自動掃描 + entry_point 插件 | skills 系統 | 手動 `@tool` 裝飾器 |
| Schema | 繼承 `Tool` base class + JSON Schema | SKILL.md + YAML frontmatter | Pydantic / JSON Schema |
| 發現範圍 | 內建 module + 第三方 entry_point | 內建 + plugin 載入器 | 僅顯式註冊 |

### Context 管理

| 面向 | Nanobot | Hermes Agent |
|------|---------|-------------|
| System prompt | Jinja2 templates + bootstrap files（SOUL.md/USER.md） | 程式碼內建 + skills |
| Token 治理 | 多層 pipeline（microcompact→snip→budget） | 內建 context window 管理 |
| 長期記憶 | Dream two-phase（background LLM） | memory tool + session_search |
| Session 隔離 | per-session key + per-session lock | per-session（Hermes） |

### Agent Loop 模型

| 面向 | Nanobot | LangGraph |
|------|---------|-----------|
| Loop 驅動 | TurnState FSM（8 個狀態） | StateGraph（DAG-based） |
| State 管理 | Session messages + metadata | TypedDict state（顯式 schema） |
| 中斷/續跑 | Checkpoint（`_RUNTIME_CHECKPOINT_KEY`） | 內建 interrupt/resume |
| 學習曲線 | 中（~1.6k 行核心） | 高（graph abstraction） |
