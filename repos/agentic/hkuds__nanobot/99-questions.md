---
repo: hkuds/nanobot
file: 99-questions
---

# Nanobot · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 CLI 入口選擇 typer 而非 argparse/click？**
  - 我目前的推測：[UNVERIFIED] typer 的 type hint-based API 跟 Pydantic 整合較好，且 `nanobot/cli/commands.py` 大量使用 `typer.Argument` / `typer.Option` 來做 CLI 參數解析
  - 相關程式碼：[`nanobot/cli/commands.py`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/cli/commands.py)

- [ ] **為什麼 providers 用 `factory.py` + `registry.py` 雙層抽象，而非單一 factory？**
  - 我目前的推測：[UNVERIFIED] `registry.py` 負責 provider metadata 的管理（name→backend mapping、模型列表、OAuth 支援標記），`factory.py` 負責實際 instance 的建立。分層讓 provider discovery 和 provider 建立可以被分別 mock。但對於只支援 7+ providers 的規模來說，這層分離是否過早？
  - 相關程式碼：[`nanobot/providers/factory.py`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/providers/factory.py)、[`nanobot/providers/registry.py`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/providers/registry.py)

- [ ] **`ToolLoader` 為支援外部 plugin 而保留 entry_point 整合，但目前的 plugin 生態狀況？**
  - `pyproject.toml:114` 有 `[project.entry-points."nanobot.tools"]` 的範例，但實際在程式碼中從 entry_point 載入 plugin 的路徑 (`loader.py:62-84`) 似乎是以備而不用為主。目前有多少第三方 plugin 存在？
  - 相關程式碼：[`nanobot/agent/tools/loader.py:62`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/tools/loader.py#L62)

- [ ] **MCP 連線為何是 lazy one-time connect 而非 per-request？**
  - 我目前的推測：[UNVERIFIED] MCP 連線有 startup overhead（auth handshake、tool discovery），lazy connect 在收到第一條訊息時觸發，之後常駐。但如果 MCP server 掛掉，為什麼沒有自動 reconnect？
  - 相關程式碼：[`nanobot/agent/loop.py:478-498`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/loop.py#L478)

- [ ] **為什麼 context governance 的 `_snip_history` 是用 `estimate_prompt_tokens_chain` 而非直接使用 tiktoken？**
  - 我目前的推測：[UNVERIFIED] `estimate_prompt_tokens_chain` 可能是基於 tiktoken 的 wrapper，但加了 provider-specific 的格式開銷估算（如 Anthropic 的 message format overhead）
  - 相關程式碼：[`nanobot/agent/runner.py`](https://github.com/HKUDS/nanobot/blob/ccbc0bb/nanobot/agent/runner.py) 中的 `_snip_history` / `_apply_tool_result_budget`

## 想問維護者的問題

- 你們從 `litellm` 遷移到原生 SDK（2026-03-21 的 commit）的動機是什麼？是為了減少 dependency、更精細的 control，還是 performance？
- Dream 記憶系統的命名靈感是什麼？為什麼叫「Dream」？
- `unified_session` 模式跟 per-channel session 模式在實際使用上的比例如何？哪個模式更常用？
- `NANOBOT_MAX_CONCURRENT_REQUESTS` 預設 3 的依據是什麼？是 benchmark 過的最佳值還是經驗值？

## 下次再看時的待辦

- [ ] 深入研究 `Consolidator.pick_consolidation_boundary()` 的演算法——如何選擇 token budget 的裁切邊界而不破壞 turn 完整性
- [ ] 看一下 webui 的 WebSocket multiplex 協定——`webui/` 跟 `nanobot/web/` 之間的 API 合約
- [ ] 對照 Hermes Agent 的 `skills` 系統跟 Nanobot 的 `nanobot/skills/` 系統的差異
- [ ] 讀取 `nanobot/utils/file_edit_events.py`——這是 streaming file edit 的實作，值得研究
- [ ] 檢查 `.agent/security.md` 了解安全邊界的詳細定義

## 跨專案對照備忘

<!-- AGENT: agentic 領域 pattern 重複率高,特別留意這節 -->

- **TurnState 顯式狀態機** → `_patterns/agent-state-machine.md` 已歸納。Nanobot 的實作是該 pattern 的「Enum + transition table」變體，跟 LangGraph 的 declarative graph 不同——更輕量但表達力較弱。補充了一種「固定 8 階段線性 FSM」的具體版本。
- **LLM Provider 抽象** → `_patterns/llm-provider-abstraction.md` 已歸納。Nanobot 的實作（`base.py` ABC + `factory.py` + `fallback_provider.py`）是「薄抽象層 + OpenAI-shaped 訊息格式」路線的典範。
- **pkgutil auto-discovery plugin** → 跟 LightRAG 的 `STORAGES` 註冊表模式（`hkuds/lightrag` 的 `kg/__init__.py`）屬於同一類做法——用 Python 的 introspection 機制取代手寫註冊清單。但 LightRAG 用中央 dict 做靜態註冊，Nanobot 用 pkgutil 做動態掃描。這個差異值得關注：如果第二個 repo 也出現類似做法，可能是候選 pattern。
- **Bootstrap files（AGENTS.md / SOUL.md / USER.md）** → Claude Code 系的 `.claude` / `CLAUDE.md` 同一脈絡。如果第三個 agent 框架也支援「透過 markdown 檔案調整 agent 行為」，值得收錄為 pattern。
