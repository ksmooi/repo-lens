---
repo: wecoai/aideml
file: README
type: agentic
studied_at: 2026-05-22
commit_sha: 40dcf28
language: Python
framework: 自製 (無 agent framework)
agent_style: single-agent
stars: 1.3k
status: active
---

# AIDE ML · 概覽

AIDE（AI-Driven Exploration in the Space of Code）是 WecoAI 開發的 LLM-driven agent，以**樹搜尋（tree search）** 方式在程式碼空間中探索、撰寫、執行、評估 ML pipeline。它不是傳統的 ReAct agent——不反覆呼叫 tool——而是每次迭代「思考一個改進點 → 產生完整程式碼 → 執行 → 用 LLM 評估」的閉環。

## 解決什麼問題

- **自動化 ML 模型建立**：給定一個資料集與自然語言目標（「預測房價」），agent 自主撰寫、除錯、調校 ML pipeline，直到達到最佳驗證指標。
- **解決「單次生成不夠好」的問題**：與一次性 code generation 不同，AIDE 透過樹搜尋產生多個解決方案，基於 metric 反饋選擇最佳分支迭代改進。
- **提供可複現的研究基準**：做為論文 ([arXiv 2502.13138](https://arxiv.org/abs/2502.13138)) 的 reference implementation，讓 agent 架構研究者能替換搜尋啟發式、評估器或 LLM 後端。

## 為什麼值得研究

1. **樹搜尋取代 ReAct**：AIDE 不以 tool calling 為核心迴圈，而是把「每次改進」視為樹的一個節點，用 metric 作為節點好壞的訊號。這與多數 agent 框架的思考方式根本不同。
2. **Two-LLM 分工**：coding 用高效模型（預設 `o4-mini`），評估/除錯用不同模型（預設 `gpt-4.1-mini`），利用各自擅長的工作。這個分離在 agent 系統中少見但有道理。
3. **極簡架構**：core 只有 ~20 個 Python 檔、沒有 agent framework 依賴、沒有複雜的 graph engine。整份程式碼可以在幾小時內讀完並理解其運作。

## Agent 系統定位

| 面向 | 選擇 |
|---|---|
| Agent 風格 | Tree search in code space（非 ReAct） |
| 數量 | Single agent |
| 編排方式 | 演算法驅動（搜尋策略 + 節點選擇） |
| Memory | session 內（Journal 作為 solution tree） |
| Tool calling | 無——agent 直接寫完整程式碼並執行 |

## 技術棧一句話

`Python` + `OpenAI API` + `OmegaConf (config)` + `rich (CLI UI)` + `dataclasses-json (serialization)`

## 與競品比較

| 面向 | AIDE (wecoai/aideml) | OpenAI MLE-bench | OpenHands |
|---|---|---|---|
| 核心抽象 | Tree search in code space | 評估框架（非 agent） | ReAct agent |
| Agent 風格 | 寫程式碼 → 執行 → 評估 | N/A（基準測試） | Shell + code 互動 |
| 搜尋機制 | 多候選方案的樹搜尋 | 無 | 線性 chain-of-thought |
| 評估方式 | LLM 評估執行結果 + 自訂 metric | 固定測試集 | 隱含在任務中 |
| 目標場景 | Kaggle 競賽型 ML 任務 | ML engineering 基準 | 通用開發任務 |

AIDE 在 MLE-Bench 上比最佳線性 agent（OpenHands）多贏 **4 倍獎牌**，這是樹搜尋相較線性 agent 最直接的量化優勢。

## 健康度信號

- ⭐ Stars: ~1.3k
- 📅 最後 push: 2026-05-02
- 👥 主要維護者: WecoAI 團隊（Zhengyao Jiang、Dominik Schmidt 等）
- 🔄 commit 頻率: 活躍，有多篇外部研究論文建基於此

## 我會在後續筆記中回答的問題

- 這套樹搜尋策略與傳統 MCTS 有什麼差異？
- Two-LLM 分工的設計取捨——為什麼不讓同一個模型既寫程式碼又做評估？
- Journal / memory 的摘要策略如何影響搜尋效率？
