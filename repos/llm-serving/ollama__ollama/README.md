---
repo: ollama/ollama
file: README
studied_at: 2026-05-25
commit_sha: f63eea3
type: llm-serving
language: Go / C++
framework: llama.cpp (自訂引擎)
stars: 172.2k
status: active
---

# Ollama · 概覽

## 解決什麼問題

Ollama 解決的是「**在本地（不用 GPU 雲端）跑大型語言模型**」這個問題。
它讓使用者只要一條指令 `ollama run llama3` 就能下載、載入、並與一個 LLM 對話，
把模型部署的複雜度從「編譯 llama.cpp → 下載 GGUF → 設定 HTTP server → 串 API client」
壓縮到一個 binary 搞定。

## 為什麼值得研究

1. **170k+ stars** — 開源 AI 生態中少數真正抵達大眾使用者的專案
2. **從簡單到複雜的演化路徑清晰** — 最早的 Ollama 只是 llama.cpp 的 Go wrapper，
   現在已經長出自己的 Go 推論引擎（ollamarunner）、ML 張量抽象層（`ml/`）、
   OpenAI compatible API、desktop app、cloud 整合
3. **雙引擎策略** — 同時維護 llamarunner（CGo/llama.cpp）和 ollamarunner（純 Go 引擎），
   展示了跨語言推論引擎的遷移策略
4. **排程器設計** — 單機多模型的 GPU 記憶體管理、keep-alive、驅逐策略，
   跟 vLLM 的 continuous batching 形成有趣的對照

## 技術棧一句話

Go CLI + HTTP 伺服器，背後以子行程（subprocess）或 CGo 方式管理 llama.cpp / 自製推論引擎，
支援 GGUF / safetensors 模型格式，提供 REST API 與 OpenAI / Anthropic 相容端點。

## 競品比較

| 面向 | Ollama | llama.cpp | vLLM | Ollama 的策略 |
|---|---|---|---|---|
| 目標使用者 | 一般開發者、個人使用者 | C++ 開發者、嵌入式 | 生產環境、高吞吐 | 最低的入門門檻 |
| 安裝 | 單一 binary，下載即用 | 需編譯 C++ | pip install，有系統需求 | 犧牲彈性換取零配置 |
| 模型管理 | 自動下載 + 版本追蹤 | 手動管理 GGUF | 從 HF Hub 或其他來源 | 整合「取得模型」進 workflow |
| 多模型 | 內建排程器，自動載入卸載 | 一次一個 | 可同時 serve 多模型 | 對個人夠用，比 vLLM 輕量 |
| API | Ollama native + OpenAI/Anthropic 相容 | 需自行架 server | OpenAI 相容 | 相容性夠開發使用 |
| GPU 支援 | CUDA / ROCm / Metal / Vulkan | 同左 | CUDA / ROCm | 覆蓋 Mac （Metal） 是關鍵差異 |
| 生產等級 | ❌ 單機工具 | ❌ | ✅ | 定位不同，互補 |

## 健康度信號

- ⭐ Stars: 172,223
- 📅 最後 commit: 2026-05-24（極活躍）
- 👥 主要維護者: Ollama Inc.（約 20+ 人團隊）
- 🔬 引擎持續重構中，`x/` 目錄包含大量實驗性功能

## 我會在後續筆記中回答的問題

- Ollama 的**四層架構**（CLI → API server → Scheduler → Runner）如何分工？
- 雙引擎（llamarunner vs ollamarunner）的選擇策略是什麼？
- 排程器如何管理多模型的 GPU 記憶體？
- OpenAI API 相容層是怎麼實作的？
