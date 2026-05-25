## 學習對象
- Repo: [ollama/ollama](https://github.com/ollama/ollama)
- Commit: `f63eea3` (2026-05-24T22:29:53Z)
- 語言 / 主要技術: Go / C++
- 專案類型 / 類別: llm-serving（套用模板: ml-dl.md）
- 學習深度: standard
- 筆記路徑: `repos/llm-serving/ollama__ollama/`

## 關鍵入口檔案

| 檔案 | 選擇理由 | 深度精讀摘要 |
|---|---|---|
| `cmd/cmd.go` (`NewCLI()`) | CLI 入口，所有使用者操作從這裡開始 | 使用 cobra 建構指令樹，`runInteractiveTUI()` 使用 Bubbletea 建立對話介面。支援 server 自動啟動 |
| `server/routes.go` (`GenerateRoutes()`, `GenerateHandler()`, `ChatHandler()`) | API 伺服器核心，路由定義與請求處理 | 同時支援 Ollama native API、OpenAI 相容端點（/v1/chat/completions）、Anthropic Messages 端點。透過 `middleware/` 做格式轉換 |
| `server/sched.go` (`Scheduler`) | 模型生命週期管理核心 | 純 Go channel 事件驅動排程器，管理多模型載入/卸載/GPU 記憶體。關鍵設計：單一 goroutine 處理所有排程決策、三階段載入（Fit→Alloc→Commit） |
| `llm/server.go` (`NewLlamaServer()`, `LlamaServer` interface) | 推論引擎管理與執行策略 | 決定引擎選擇（llamarunner vs ollamarunner），管理子行程生命週期。崩潰檢測透過 `Ping()` 方法 |
| `runner/runner.go` (`Execute()`) | 引擎分派點 | 根據 CLI 參數分派到 llamarunner（預設）、ollamarunner（--ollama-engine）、mlxrunner（--mlx-engine）、imagegen（--imagegen-engine） |

## 三個最重要的發現

1. **子行程隔離架構**：Ollama 選擇將推論引擎以獨立子行程（subprocess）啟動，而非 in-process CGo 或 Python 同行程。這讓引擎崩潰（C++ 常見的 segfault）時主 server 不受影響，但代價是每次推論步驟的 HTTP+JSON 序列化 overhead。與 vLLM 的 in-process 架構形成根本性的設計對比。

2. **雙引擎遷移策略**：Ollama 同時維護 llamarunner（CGo/llama.cpp）和 ollamarunner（全新的純 Go 引擎），以模型架構為單位逐步遷移。新模型（Gemma 3/4、Llama 4、Qwen 3）強制使用 ollamarunner，傳統模型繼續使用 llamarunner，若初始化失敗自動 fallback。這是在大型開源專案中進行核心重構的典範級實作。

3. **排程器是簡潔設計的範例**：Ollama 的 Scheduler 只用 932 行 Go 程式碼就實現了多模型併發管理、GPU 記憶體追蹤、keep-alive 過期驅逐、崩潰恢復。核心是 4 個 Go channel 的事件驅動設計，沒有複雜的狀態枚舉或鎖設計。對比 vLLM 的 scheduler（數千行，包含 PagedAttention 的 page table 管理），Ollama 的 scheduler 更輕量但缺少 continuous batching。

## 候選 Pattern

- **Scheduler 設計差異**：Ollama 的「事件驅動 channel-based scheduler」和 vLLM 的「page table-aware scheduler」是 inference engine scheduler 的兩種設計極端——一個只管「模型層級」的生命週期，一個深入到「KV cache block 層級」的記憶體管理。目前觀察到 2 個 repo，需第 3 個才能升級到 `_patterns/`。
- **子行程隔離 vs in-process**：Ollama 的 subprocess runner 和 vLLM 的 in-process EngineCore 是 AI inference engine 的兩種元件隔離哲學。已在 vLLM 的 9-questions.md 中標記。目前觀察到 2 個 repo（Ollama + vLLM），等待第 3 個 repo（可能是 TGI 或 SGLang）。

## 品質檢查摘要

- 各檔字數（bytes）: README=2,937 / 1-arch=10,661 / 2-walk=9,658 / 3-pat=12,174 / 9-q=5,556
- Mermaid 圖數量: 1-arch=8 張、2-walk=2 張、3-pat=2 張（合計 12 張）
- path:line 引用總數: 39
- 量化資訊出現次數（版本、規模、效能等具體數字）: 15+（包含環境變數預設值、競品比較數字、效能推測）
- 比較表格出現次數: 1（README 競品對照表）

## 執行決策日誌

- REPO_SLUG 決定與衝突檢查結果: `ollama__ollama`，無 branch 衝突
- 類別確認: CATEGORY=llm-serving / TEMPLATE=ml-dl.md
- 類別決策說明: ollama 是本地 LLM 推論引擎，專注於推論加速、模型載入/卸載管理，屬於 llm-serving 類別
- 新增類別時的附帶動作: 無，使用既有類別（llm-serving）
- 關鍵入口檔案選擇與排除理由: 選擇 CLI、server routes、scheduler、llm server、runner dispatch 五個點，排除 app/（桌面應用，架構較獨立）、x/（實驗性功能，尚未穩定）、openai/（轉換層，依賴 routes）
- 競品識別: llama.cpp（底層引擎）、vLLM（生產級 serving）、Ollama 的差異定位在「最低入門門檻」
- subagent 使用情況: 使用 2 個 subagent 平行分析（ollamarunner vs llamarunner 差異分析、排程器與模型生命週期分析）
- 工具替換: 無
- 模板章節未明確規則的處理: `ml-dl.md` 模板原為 ML 訓練專案設計，對 inference engine 做了大量調整——加強架構圖與資料流圖、加入子行程隔離設計決策、調整 code-walkthrough 為請求路徑追蹤而非 training step
- 任務 prompt 覆寫 AGENTS.md 的項目: 無
- [UNVERIFIED] 標註的部分清單:
  - 「ollamarunner 和 llamarunner 對同一個模型的輸出是否一致」→ 3-key-patterns #P3
  - 「continuous batching 是否真的不適合 Ollama 的架構」→ 9-questions.md
  - 「未來是否會增加其他 backend」→ 3-key-patterns #P6

## Git 身份驗證

- Commit author: ksmooi <kaishianmooi@gmail.com>
- Git config 確認指令輸出:
  (將在 commit 後補充)

## 驗證版本說明
此 PR 為 Hermes Agent repo 學習功能，學習對象:ollama/ollama。
