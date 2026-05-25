---
repo: infiniflow/ragflow
file: 9-questions
studied_at: 2026-05-24
commit_sha: e6dd3975
---

# RAGFlow · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼選擇 Quart 而非 FastAPI？**
  - README 說明了 Quart 保留 Flask extension 生態的理由，但沒有解釋為什麼從 Flask 遷移到 Quart（而非從一開始就選型）。
  - 我目前的推測：[UNVERIFIED] RAGFlow 可能在 Flask 上起步，後續為了非同步支援遷移到 Quart。但 Quart 是非同步 Flask 的相容實作，它的非同步生態不如 FastAPI 的 asyncio 深度整合。如果從頭開始設計，選擇 FastAPI 可能更合理。
  - 相關程式碼: [`api/apps/__init__.py:60`](https://github.com/infiniflow/ragflow/blob/e6dd3975/api/apps/__init__.py#L60)

- [ ] **Python 與 Go 之間的邊界如何劃分？**
  - 有些操作在 Python 端直接執行（文件上傳、chunking、LLM 呼叫），有些在 Go（搜尋、storage、tokenizer），有些則兩端都有（如搜尋服務：Python 端的 `search_service.py` 和 Go 端的 handler + service）。這個邊界的決定原則不清楚。
  - 我目前的推測：[UNVERIFIED] Python 為主的操作被拆分的標準是「是否需要直接呼叫第三方 Python SDK（如 OpenAI）」。但這個推測無法解釋為什麼搜尋服務在兩端都有實作。
  - 相關程式碼: [`internal/handler/search.go`](https://github.com/infiniflow/ragflow/blob/e6dd3975/internal/handler/search.go) vs [`api/db/services/search_service.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/api/db/services/search_service.py)

- [ ] **DSL 中的 Loop / Iteration 元件如何處理狀態共享？**
  - 從 `agent/canvas.py` 的程式碼可以看到 `globals` 是單一的 dict。當 Loop 元件反覆執行某個子圖時，每次 iteration 的狀態如何隔離？如果 iteration A 的輸出影響了 iteration B 的輸入，行為是否可預測？
  - 我目前的推測：[UNVERIFIED] Loop 元件可能在每次 iteration 前建立 `globals` 的快照或副本，iteration 結束後將輸出 merge 回主 globals。但具體實作未在程式碼中確認。
  - 相關程式碼: [`agent/component/loop.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/component/loop.py), [`agent/component/iteration.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/component/iteration.py)

- [ ] **DeepDoc 的 vision-based parser 的具體運作方式？**
  - README 提到 DeepDoc 支援 vision-based layout analysis，但實際看 `deepdoc/vision/` 目錄，沒有看到完整的模型架構說明。是使用現成的 vision model（如 LayoutLM / DocTR）還是自訓練的模型？
  - 我目前的推測：[UNVERIFIED] 從 `deepdoc/vision/` 的檔案命名推測可能使用 ONNX runtime 部署的開源 model（如 PP-OCR / LayoutLMv3），但未實際驗證。
  - 相關程式碼: [`deepdoc/vision/`](https://github.com/infiniflow/ragflow/tree/e6dd3975/deepdoc/vision)

- [ ] **Agent with Tools 如何處理 tool call 的錯誤回饋給 LLM？**
  - `agent_with_tools.py` 的 `max_rounds = 5` 限制了 LLM 可以嘗試的 tool call 次數，但如果某個 tool 持續失敗（如 API 斷線），LLM 是怎麼被通知的？
  - 我目前的推測：[UNVERIFIED] 從 `LLMErrorCode` 的設計推測，tool 執行失敗的錯誤訊息會以 `content` 形式餵回 LLM，讓 LLM 決定下一步（重試、換工具、或回報失敗）。這個「把錯誤餵回 LLM」的模式類似 LangChain 的 AgentExecutor。
  - 相關程式碼: [`agent/component/agent_with_tools.py`](https://github.com/infiniflow/ragflow/blob/e6dd3975/agent/component/agent_with_tools.py)

- [ ] **Python 與 Go 之間的通訊為什麼用 HTTP(REST) 而非 gRPC？**
  - 兩個服務之間的資料交換量不大（每次請求通常 < 1KB），HTTP REST 確實足夠。但為什麼不考慮 gRPC 的雙向串流？Python 的 asyncio + Go 的 goroutine 都是串流友善的。
  - 相關程式碼: [`internal/router/router.go`](https://github.com/infiniflow/ragflow/blob/e6dd3975/internal/router/router.go)

## 想問維護者的問題

- RAGFlow 的文件解析在實際生產環境中，跟 PyMuPDF / pdfplumber 等傳統 parser 相比，chunking 品質提升了多少？有沒有內部的 benchmark 數據？
- Go 與 Python 的分工決策是從專案初期就決定的，還是後續重構的結果？如果從頭再來一次，還會選擇同樣的架構嗎？
- Infinity 作為 doc engine，與 Elasticsearch 相比，在 RAGFlow 的實際使用場景中表現如何？有沒有具體的 latency / throughput 比較數據？
- DSL 的版本管理策略是什麼？如果一個已經部署的 agent DSL 被更新，正在進行中的 conversation 是使用舊版還是新版？
- 在專案 81k stars 的規模下，你認為最大的工程挑戰是什麼？（是維護雙語言程式庫？文件格式的相容性？還是 LLM provider 的一致性？）

## 下次再看時的待辦

- [ ] 深入測試 DeepDoc 的 parser 品質：找一份 multi-column PDF 和一份 scan-heavy PDF，比較 DeepDoc vs pdfplumber vs Marker 的 chunking 結果
- [ ] 實測 RAGFlow Agent DSL：製作一個包含 Retrieval → Categorize → Agent with Tools (web search) → Generate 的 DSL，看實際執行時的串流行為與錯誤處理
- [ ] 探索 Infinity 在其他 RAG 框架中的使用方式（如 LlamaIndex 的 Infinity vector store integration）
- [ ] 研究 MCP 整合的深度：RAGFlow 的 MCP server 與一般的 MCP host 有何不同？
- [ ] 對比 AutoGen / CrewAI 的 multi-agent 設計與 RAGFlow 的 DAG DSL，兩者在 agent orchestration 上的取捨

## 跨專案對照備忘

- **LLM Provider 抽象**（Pattern 5）與 `llm-provider-abstraction` pattern 高度吻合：`_patterns/llm-provider-abstraction.md` 中已歸納 4 個 repo 的做法，RAGFlow 的 LiteLLM 整合是第 5 個實例。RAGFlow 的做法特別之處在於它用 LiteLLM 而非自製 wrapper，這是 pattern 中尚未記錄的變體。
- **Component Registry**（Pattern 2）與 LangGraph / AutoGen 的 node/agent registry 類似。如果在 3+ repo 中都觀察到「字串名稱 → 類別」的註冊模式，值得提升為 `_patterns/` 中的 component-registry pattern。
- **候選 Pattern: Dual-language Architecture for RAG Systems**：RAGFlow 使用 Python + Go 的雙語言架構來平衡開發速度與執行效能。這種模式尚未在 `_patterns/` 中記錄，但目前只在 RAGFlow 看到。繼續觀察其他 RAG 系統是否也會走向類似的分工（如 Jina AI 的 Python + Go 堆疊）。
- **候選 Pattern: Declarative DAG DSL for LLM Workflows**：RAGFlow 用宣告式 DAG 而非程式碼定義 workflow。這種做法與 LangGraph（graph-based state machine）共享「圖形化編排」的核心想法，但 RAGFlow 用純 JSON + 可視化編輯器，LangGraph 用程式碼構建。需要觀察是否有第三個 repo 也採用純 JSON DSL 的做法。
