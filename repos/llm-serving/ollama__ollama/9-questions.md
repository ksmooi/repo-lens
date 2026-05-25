---
repo: ollama/ollama
file: 9-questions
studied_at: 2026-05-25
commit_sha: f63eea3
---

# Ollama · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 Ollama 選擇「子行程 HTTP + JSON」而非更高效的 IPC？**
  子行程隔離是合理的設計，但用 HTTP + JSON 作為通訊協定是否是最佳選擇？
  在 loopback 上，local TCP 的 overhead 雖然比跨機器小，但每次推論步驟的 JSON
  序列化/反序列化（尤其 streaming 時每 token 一次）累積起來很可觀。
  [UNVERIFIED] 推測可能是 Pragmatic 考量：HTTP 最容易除錯（可用 curl 直接打 runner port）、
  且不需要引入 protobuf/flatbuffers 等依賴。但對比 vLLM 的 Python 同行程 zero-copy 傳遞 tensor，
  Ollama 的序列化成本可能是吞吐量的瓶頸。
  - 相關程式碼：[`llm/server.go:358`](https://github.com/ollama/ollama/blob/f63eea3/llm/server.go#L358)
    子行程啟動，[`server/routes.go:309-324`](https://github.com/ollama/ollama/blob/f63eea3/server/routes.go#L309-L324) streaming 回寫

- [ ] **為什麼 ollamarunner 需要自己的 GGML backend，而不是直接叫 llama.cpp 的 kernel？**
  新引擎的 `ml/backend/ggml/` 似乎是重新包裝了 ggml 的 C API。
  跟 llamarunner 直接叫 llama.cpp 的 `llama_decode()` 有什麼實質差異？
  [UNVERIFIED] 推測：llama.cpp 的 `llama_decode()` 封裝了完整的推論流程（prompt processing + decode），
  而 `ml.Backend` 層次更低（Tensor 層級），讓 ollamarunner 可以精確控制計算圖的每一步。
  但這也意味著 ollamarunner 需要自己實作注意力、FFN 等計算，
  這在 `ml/nn/` 中確實看到了。
  - 相關程式碼：[`ml/backend/ggml/ggml.go`](https://github.com/ollama/ollama/blob/f63eea3/ml/backend/ggml/ggml.go)

- [ ] **Ollama 為什麼沒有 continuous batching？**
  這是 vLLM 的核心賣點。Ollama 的 `numParallel` 參數只是讓多個請求「序列化」地共用一個
  runner session（context window 中的不同 slot），並非真正的 continuous batching
  （在 decode phase 插入新的 prefill）。
  [UNVERIFIED] 可能的原因是：continuous batching 需要 scheduler 層級的 cross-request 協調，
  而 Ollama 的 scheduler 設計是以「每個模型」為單位，不是以「每個請求」為單位。
  從架構上來說，如果 Ollama 要支援 continuous batching，
  scheduler 需要深入了解 runner 的內部狀態（正在處理多少 tokens、哪個 slot 空）。
  目前的子行程隔離架構讓這非常困難。

- [ ] **為什麼 `runner/runner.go` 的分派邏輯是基於 CLI 參數而非編譯時？**
  選擇引擎的程式碼是 Go switch：
  ```go
  if args[0] == "--ollama-engine" {
      return ollamarunner.Execute(args[1:])
  }
  ```
  為什麼不透過編譯標籤（build tags）或 init() 註冊機制來分派？
  [UNVERIFIED] 推測：因為 ollamarunner 和 llamarunner 是同一個 binary 的一部分，
  編譯時分派反而限制了靈活性。CLI 參數的方式讓主 server 可以在執行時期決定用哪個引擎。
  - 相關程式碼：[`runner/runner.go:10`](https://github.com/ollama/ollama/blob/f63eea3/runner/runner.go#L10)

## 想問維護者的問題

- [ ] ollamarunner 的長遠目標是什麼？最終會完全取代 llamarunner 嗎？
- [ ] 你們如何確保 ollamarunner 和 llamarunner 對同一個模型產生相同（或統計上不可區分）的輸出？
- [ ] `x/` 目錄下的實驗性功能（agent、imagegen、safetensors）的 maturing 路徑是什麼？
  什麼條件下會 promotion 到核心目錄？

## 想驗證的推測

- [ ] **子行程隔離對效能的具體影響**：我想在 vLLM 和 Ollama 之間，對同一個模型、同一個 prompt、
  同一個硬體做 latency 比較。推測 Ollama 的 cold start（包含子行程啟動）比 vLLM 慢 2-5 倍，
  但 warm request 的 TTFB 差異不大。
  - 測試方法：`ollama run llama3.2 --verbose` 對比 `vllm serve meta-llama/Llama-3.2-3B-Instruct`

- [ ] **continuous batching 是否真的不適合 Ollama 的架構？**
  理論上，如果 runner 暴露更細節的控制（如「這個 slot 的 prefill 完成了，可以插入新請求的 prefill」），
  continuous batching 可以在子行程架構上實現。但這需要 redefine runner ↔ server 的通訊協定。

## 跨專案對照備忘

- **Ollama 的 Scheduler vs vLLM 的 Scheduler**：兩者都是事件驅動，但 vLLM 的 scheduler
  深入到 KV cache manager 的 page table 層級（PagedAttention），
  而 Ollama 的 scheduler 只管理到「哪個模型在 GPU 上」這個層級。
  這兩者是不同層次的抽象——Ollama 沒有支援 continuous batching 的 scheduler 設計。
  值得追蹤如果 Ollama 加入 continuous batching，scheduler 會如何演進。
  → **候選 pattern**：ML 推論引擎的排程器設計差異

- **Ollama 的子行程隔離 vs vLLM 的 in-process**：這是架構面的根本差異。
  目前觀察到 2 個 repo（Ollama + ???），還需要至少 1 個 repo 才能升級為 `_patterns/` 的正式 pattern。
  → 暫存在此，等第 3 個 repo 出現

- **vLLM 的 Engine frontend/core 分離 vs Ollama 的 server↔runner 隔離**：兩者都選擇了「透過序列化協定隔離元件」的架構，但抽象層次不同。
  vLLM 用 msgspec 在 single process 內做 message-passing，Ollama 用 HTTP+JSON 在跨 process 間做。
  → 已在 vLLM 的 9-questions.md 中標記為候選 pattern「production-grade AI 系統的元件隔離策略」(`repos/llm-serving/vllm-project__vllm/9-questions.md`)

## 下次再看時的待辦

- [ ] 深入了解 `ml/backend/ggml/` 的實作細節，看它跟 llama.cpp 的 ggml API 有何差異
- [ ] 跑一次 `OLLAMA_NEW_ENGINE=true` 對比同一模型的輸出是否一致
- [ ] 深入了解 `x/tools/`（tool calling）和 `x/agent/`（agent loop）的設計
- [ ] 讀 `app/`（desktop app）的跨平台 UI 架構（似乎是 webview？）
