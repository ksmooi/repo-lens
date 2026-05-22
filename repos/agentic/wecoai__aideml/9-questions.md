---
repo: wecoai/aideml
file: 9-questions
studied_at: 2026-05-22
commit_sha: 40dcf28
---

# AIDE ML · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼使用 OmegaConf 而不是 Pydantic Settings 或 Hydra？**
  - 我目前的推測：[UNVERIFIED] OmegaConf 的 CLI 參數合併（`OmegaConf.from_cli()`）比 argparse 或 typer 更簡潔——支援 `agent.code.model="claude-4-sonnet"` 這種巢狀 CLI 結構。但缺點是型態安全不足（Config dataclass 只是 type hint，實際 validation 靠後續的 OmegaConf structured merge）。
  - 相關程式碼：[`config.py:96-102`](https://github.com/wecoai/aideml/blob/40dcf28/aide/utils/config.py#L96-L102)

- [ ] **為什麼 search_policy 不使用 UCB 或 Thompson Sampling？**
  - 我目前的推測：[UNVERIFIED] 簡單的 ε-greedy + debug bias 可能已經足夠好，因為每個節點的 metric 評估成本高（需要完整程式碼執行），做 exploration 的代價太大。但這也表示 AIDE 可能會過早收斂到一個次佳分支——如果前 5 個 drafts 都很接近但都不是最好的。
  - 相關程式碼：[`agent.py:61-92`](https://github.com/wecoai/aideml/blob/40dcf28/aide/agent.py#L61-L92)

- [ ] **為什麼改善步驟只給 parent 的完整程式碼，而不做 diff / patch？**
  - 我目前的推測：[UNVERIFIED] 給完整程式碼讓 LLM 可以更方便地做任意修改。如果只給 diff context，LLM 可能無法理解整體結構。但代價是每次 improve call 的 token 消耗很高（完整程式碼在系統 prompt 中）。
  - 相關程式碼：[`agent.py:219-221`](https://github.com/wecoai/aideml/blob/40dcf28/aide/agent.py#L219-L221)

- [ ] **為什麼不支援平行執行多個 Agent？**
  - 這本質上是 sequential 演算法——每個節點依賴前一個節點的執行結果。但 initial drafts（num_drafts=5）理論上可以平行執行。目前完全未利用平行化，可能因為設計意圖是單機單程式的 research code。

- [ ] **execute 過程中如果 child process 因 OOM 被 kill，Interpreter 如何反應？**
  - 我目前的推測：[UNVERIFIED] `interpreter.py:258-265` 中檢查 `self.process.is_alive()`，若 child process 在執行中死亡會 raise `RuntimeError("REPL child process died unexpectedly")`。但這個例外並沒有在 `agent.step()` 中被 catch——表示 AIDE 假設程式碼執行不會導致 child process crash（只會 raise Python exception）。若 child 被 OOM killer 終止或 segfault，整個 run 會中斷。

## 想問維護者的問題

- Journal 中的 `InteractiveSession` 類別（`journal.py:104-131`）看起來是為 Jupyter notebook 互動準備的，但從未被 `agent.step()` 或 `run.py` 使用。這是剩餘的代碼還是未完成的功能？
- OpenRouter backend（`backend_openrouter.py`）的實作方式是什麼？它在 API 相容性上有什麼特殊的處理？
- 為什麼預設 `k_fold_validation: 5`？這對所有 ML 任務都合適嗎？某些任務（如時間序列）的 cross-validation 策略應該不同。
- MLE-Bench 上 AIDE 勝過線性 agent 4 倍——這個實驗中 num_drafts 設了多少？搜尋深度（steps）是多少？

## 下次再看時的待辦

- [ ] 深入研究 `data_preview.py`——資料預覽的內容格式與品質對 coding LLM 的影響很大
- [ ] 跑跑看 AIDE 在實際資料集上的行為，觀察 search_policy 的雜訊決策是否真的影響結果品質
- [ ] 對照 AutoGen 的 Component config（YAML-based）與 AIDE 的 OmegaConf config 在可擴展性上的差異
- [ ] 對照 LangGraph 的 State 管理與 AIDE 的 Journal——兩者對「記憶」的抽象完全不同，各有什麼 trade-off？
- [ ] 看其他 `auto-ML` 相關 repo（如 AutoGluon、H2O AutoML）來對照 AIDE 的 tree search 策略

## 跨專案對照備忘

- **Tree search in code space**：與 AutoGen 的 actor model、LangGraph 的 graph-based workflow 完全不同。AIDE 證明在某些領域（ML pipeline 自動化），「寫程式碼」比「呼叫 tool」更有效率。這個觀察值得關注是否在其他 code-generation agent 中出現。
- **Two-LLM architecture**：Coding model + evaluation model 的分工在這個 repo 中非常明確。這類「writer/editor」分離模式在 agent 框架中應該更常見——目前 LangGraph / AutoGen 都沒有內建這個模式。**候選 pattern**。
- **WorstMetricValue sentinel**：使用 subclass 實作「永遠最差值」的設計模式。這個技巧可以推廣到任何需要 sentinel value 的比較系統，不限於 metric。
