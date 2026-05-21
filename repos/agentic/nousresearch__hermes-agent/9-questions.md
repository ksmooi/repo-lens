---
repo: nousresearch/hermes-agent
file: 9-questions
studied_at: 2026-05-21
commit_sha: 48be2e0
---

# Hermes Agent · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 conversation loop 的 tool argument 型別強轉（coerce_tool_args）只做在 dispatch 層而不是 schema 層？**
  - 目前 `model_tools.py:545-626` 在每個 tool call 時對參數做型別轉換（字串→int、JSON 字串→dict）。如果 schema 層就用 Pydantic / JSON Schema strict mode 定義型別，是否可以讓 LLM 直接產生正確型別的參數，省去這層轉換？
  - 我目前的推測：[UNVERIFIED] 因為不是所有 provider 都支援 strict function calling，且 strict mode 有時會導致不知道原因的回應失敗，所以改在 dispatch 層做寬鬆的容錯處理。

- [ ] **Skill system prompt 的兩層快取（in-process LRU + disk snapshot）在冷啟動時仍然需要掃描 ~/.hermes/skills/。120+ skills 的掃描時間在受限環境（如手機 terminal）是否可接受？**
  - [`agent/prompt_builder.py:1045-1118`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/prompt_builder.py#L1045-L1118)
  - 如果 disk snapshot 的 mtime/size manifest 驗證通過，理論上只需讀一個 JSON 檔案（幾 KB）。但 cold path 需要 walk 整個 skills 目錄樹、parse frontmatter、過濾平台，對 120+ skills 來說是否仍可接受？如何衡量這對 CLI 啟動體驗的影響？

- [ ] **Gateway 的 AIAgent LRU cache（cap=128, TTL=1h）是否真的足夠？**
  - 128 個同時活躍的 session 對一個社區 bot 可能足夠，但每個 AIAgent 持有完整的 messages list（可能數百則對話），記憶體壓力如何？是否有過 OOM 的 issue？
  - 1h TTL 的選擇理由是什麼？如果用戶每天只在固定時間使用（如通勤時），TTL 應該更長還是更短？

- [ ] **為什麼選擇直接修改 `messages` list（mutate in place）而不是 immutable message flow？**
  - Loop 中大量使用 `messages.append(...)`、`messages.insert(...)`、`messages.pop(...)` 來操作 conversation history。這讓訊息流動的 trace 變得很難（需要知道 list 在每個點長怎樣）。是否有考慮過 immutable message chain（每次操作回傳新 list）？trade-off 是什麼？
  - 相關程式碼：[`agent/conversation_loop.py:709-881`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/agent/conversation_loop.py#L709-L881)

- [ ] **Plugin hooks 的執行順序是怎麼決定的？**
  - `pre_tool_call`、`post_tool_call`、`transform_tool_result`、`pre_llm_call`、`post_llm_call` 等 hooks 都有多個 plugin 可以註冊。當多個 plugin 註冊了同一個 hook，執行順序是依賴 import 順序還是註冊順序？是否有明確定義的優先級機制？
  - 相關程式碼：[`model_tools.py:741-894`](https://github.com/nousresearch/hermes-agent/blob/48be2e0e4dbc4489f418e8d58794790c9c830390/model_tools.py#L741-L894)
  - 我目前的推測：[UNVERIFIED] 看起來是依賴 `plugins/` 的 import 順序（檔案系統順序？），但這顯然不是一個可靠的機制。

## 想問維護者的問題

- Conversation loop 為何不給 `tool_dispatch_helpers.py:103-146` 的 `_should_parallelize_tool_batch()` 一個可設定的 parallel/sequential policy？目前是寫死的 heuristic——如果使用者想強制所有工具平行（或全部順序），沒有 config way。
- Skill system 的 `inline_shell` 預設關閉（`skill_preprocessing.py:123-139`），但是否有計畫加入 sandboxed shell 執行（如 Firecracker / gVisor container）讓它可以安全啟用？
- `agent/error_classifier.py` 的 `classify_api_error()` 處理了許多已知的 API 錯誤模式——這個清單是如何維護的？每次遇到新的 provider 錯誤就加一條？是否有統一的 error taxonomy？
- `cli.py` 的 676KB 是 monolith 還是真的需要把所有功能放在一個檔案？從 AGENTS.md 看到 CLI 有 skin engine、command registry、autocomplete、logs、doctor 等——有考慮拆分嗎？

## 下次再看時的待辦

- [ ] 深入研究 `plugins/kanban/` 的 multi-agent 協作——目前只確認是「execute and collect」模式，但 dispatcher 的調度演算法和 worker 的生命週期管理值得深挖
- [ ] 跑跑看 Gateway 模式（`gateway/run.py`）在多平台同時使用時的表現——LRU agent pool 在真實壓力下的行為很重要
- [ ] 對照 Claude Code 和 Codex CLI 的同類功能——它們都是 CLI-first agent，但架構完全不同（閉源 vs 開源），理解它們的設計選擇可以驗證或挑戰 Hermes Agent 的假設
- [ ] 深入研究 `agent/curator.py` 的 skill self-improvement loop——這是 Hermes Agent 最獨特的功能

## 跨專案對照備忘

- **Registry generation counter pattern**（`tools/registry.py`）跟 `paper_lens` 中發現的「version-bumped cache invalidation」是同一個模式。這在 `_patterns/` 中還沒有被歸納 → 候選 pattern
- **Plugin hook system**（multiple plugins register same hook, execution order by import order）跟 LangGraph 的 `@node` decorator 註冊不同——不是 middleware chain，而是「先註冊先用」的順序。這個「implicit ordering by registration sequence」看起來在多個框架中都出現過 → 候選 pattern（需要更多 repo 驗證）
- **AST-based auto-discovery**（`tools/registry.py:29-54`）——我第一次在一個開源專案看到這個做法。這比 file naming convention（如 Django 的 `admin.py`）或 YAML manifest 更精巧但也更 hacky。還需要 2 個以上的 repo 才能收錄到 _patterns/。
- **Imperative loop vs State machine** — Hermes Agent 的 `conversation_loop.py` 使用單一的 synchronous while loop，而 `hkuds/nanobot`（Pattern 1: `TurnState 顯式狀態機`）使用顯式狀態機（`nanobot/agent/loop.py:63`）。這兩個極端設計的對照已經存在於 `_patterns/agent-state-machine.md`，該 pattern 的替代方案列出了 state machine 和 imperative loop 的 trade-off。本次 Hermes Agent 的觀察可以作為「imperative while loop」方案的有力案例，補充到 `agent-state-machine.md` 的「在不同 repo 的實作」一節。
- **LLM Provider 抽象方式** — Hermes Agent 使用「OpenAI-compatible API 作為通用介面」（非泛型 adapter interface），而 `_patterns/llm-provider-abstraction.md` 描述的是傳統的 adapter interface 模式。兩種做法是同一問題的不同取捨。可考慮在 `llm-provider-abstraction.md` 的「觀察到的變體」中補充「以 OpenAI API 為統一標準」這個變體。
