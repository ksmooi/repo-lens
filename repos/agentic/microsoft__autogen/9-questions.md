---
repo: microsoft/autogen
file: 9-questions
studied_at: 2026-05-22
commit_sha: 027ecf0
---

# AutoGen · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 v0.7 選擇 SingleThreadedAgentRuntime 而非真正支援 distribute 的 runtime？**
  - 我目前的推測：[UNVERIFIED] 從程式碼來看，`SingleThreadedAgentRuntime` 的所有 agent 共用同一個 event loop。雖然有 gRPC runtime（在 autogen-ext 中），但核心還是 single-threaded。可能的原因是：
    - 多數使用場景（開發、測試、單機部署）不需要 distributed runtime
    - 分散式 runtime 的除錯和測試複雜度遠高於 single-threaded
    - gRPC 層可以 bridge 多個 SingleThreadedRuntime 實例
  - 但這也意味著目前 AutoGen 還不是真正「distributed」的 agent 框架
  - 相關程式碼：[`SingleThreadedAgentRuntime`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-core/src/autogen_core/_single_threaded_agent_runtime.py)

- [ ] **Component 系統的 namespace whitelist 是為了安全還是為了控制生態？**
  - 我目前的推測：[UNVERIFIED] 兩者都有。從安全性角度看，防止任意 Python module 被載入是合理的；但只允許 `autogen_*` 和 `autogen_ext.*` 也限制了第三方 provider 的發展。環境變數 `AUTOGEN_ALLOWED_PROVIDER_NAMESPACES` 的存在表示團隊知道這個限制並保留了彈性，但對一般開發者來說不夠 discoverable。
  - 相關程式碼：[`_component_config.py`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-core/src/autogen_core/_component_config.py)

- [ ] **Agent 的 stateful 設計（只傳新訊息）在 multi-agent 場景下會不會造成資料遺失？**
  - 我目前的推測：[UNVERIFIED] 如果 Agent A 跟 Agent B 需要共享 context，它們必須透過 message passing。在 GroupChat 中，所有 agent 的訊息會發布到 shared topic thread，但每個 agent 能不能「看見」其他 agent 的訊息，取決於它們的 `model_context` 是否包含那些訊息。這可能導致 agent 間的資訊不對等。
  - 相關程式碼：[`BaseGroupChat`](https://github.com/microsoft/autogen/blob/027ecf0/python/packages/autogen-agentchat/src/autogen_agentchat/teams/_group_chat/_base_group_chat.py)

- [ ] **Memory 系統為什麼這麼簡陋？**
  - 只有 `ListMemory` 一種內建實作，沒有 vector store integration、沒有自動 summarization、沒有跨 session persistence。考慮到這是 Microsoft 的專案（有足夠資源），以及 Memory 在 agent 系統中的重要性，這個現狀令人困惑。
  - 我目前的推測：[UNVERIFIED] 可能 Memory 是 v0.7 最後才實作的功能，或者團隊的優先級是先完成 runtime + team 再處理記憶。也可能是 `autogen-ext` 中的 extensibility 設計還沒有來得及補上 memory 的實作。

## 想問維護者的問題

- v0.2（Legacy pyautogen）的使用者要 migrate 到 v0.7，你們建議什麼路徑？
- Magentic-One 跟 AutoGen 的整合是長期的方向嗎？還是只是暫時的 experimental feature？
- 你們對 `SingleThreadedAgentRuntime` 的未來規劃是什麼？會往 distributed 方向發展嗎？
- 為什麼不在 Core 層內建 tracing/logging dashboard，而是依賴 OpenTelemetry 的外部工具？

## 下次再看時的待辦

- [ ] 深入研究 `GraphFlow`（DiGraph）的實作細節
- [ ] 讀 `autogen-ext/runtimes/grpc/` 了解跨語言通訊的協定
- [ ] 實際跑一個 SelectorGroupChat 的範例，觀察 LLM 如何選擇說話者
- [ ] 對照 `pyautogen`（v0.2 legacy）的架構，了解 rewrite 的具體取捨
- [ ] 測試 component YAML config 的載入流程，理解 `ComponentLoader` 的完整路徑

## 跨專案對照備忘

- **Component 系統**跟 Hermes Agent 的 `skill_manage` 的 declarative skill config 有類似概念（provider + config 分離）。若在其他 2 個以上 repo 觀察到類似設計，可考慮 `_patterns/declarative-component-system.md`。
- **Topic-based message routing** 跟 nats.io / Redis PubSub 的 pattern 相同。若在 3 個以上 agent 框架看到這種設計（CrewAI 是否也有？），可考慮 `_patterns/agent-message-bus.md`。
- **ChatCompletionClient（Pattern 5）**已有對應的 `_patterns/llm-provider-abstraction.md` pattern，不另新增。
- **Actor model vs State machine** — AutoGen 的 actor model 在 `_patterns/agent-state-machine.md` 中被列為 state machine 的替代方案之一。該 pattern 的 `[TODO]` 標記需要補充 AutoGen GraphFlow。這個交會點值得在更新 `agent-state-machine.md` 時納入。

### 候選 Pattern

- **Agent component model** — 用 Pydantic model 描述 agent system 的元件，支援 provider + config 分離與動態載入。
  觀察到於：microsoft/autogen (v0.7)
  尚未達到收錄門檻（需 3+ repo 觀察到）
