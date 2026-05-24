## 學習對象
- Repo: [hiyouga/LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory)
- Commit: `16ff5a23` (2026-05-21)
- 語言 / 主要技術: Python / PyTorch
- 專案類型 / 類別: llm-training(套用模板:ml-dl.md)
- 學習深度: standard
- 筆記路徑: `repos/llm-training/hiyouga__llama-factory/`

## 關鍵入口檔案
1. **`src/llamafactory/cli.py`** — CLI 入口（31 行），檢查 USE_V1 環境變數後轉發到 launcher。最薄的入口檔案。
2. **`src/llamafactory/launcher.py`** — 命令分派器（185 行），處理 api/chat/export/train/webui 等命令，多 GPU 時自動 fork torchrun。
3. **`src/llamafactory/train/tuner.py`** — 訓練調度核心（347 行），`_training_function()` 以三層優先級 dispatch：use_hyper_parallel > use_mca > stage。
4. **`src/llamafactory/model/loader.py`** — 模型載入管線（241 行），load_model() 執行完整的 config-patch→model-load→patch-model→init-adapter 流程。
5. **`src/llamafactory/data/template.py`** — 121 個 conversation template 的註冊與解析系統（2384 行），支援自動從 tokenizer.chat_template 解析 fallback。

## 三個最重要的發現
1. **Patcher 鏈架構** — 6 個 patch 函數（config/tokenizer/processor/model/valuehead）在模型生命週期不同階段執行，是 LLaMA-Factory 能支援 100+ 模型的關鍵設計。每個 patch 獨立可測、可開關。
2. **6 階段 × 5 方法 × 7 optimizer 的笛卡兒積管理** — 透過單一 `_training_function()` + 回退鏈（fallback chain）的 `create_custom_optimizer()` 模式，將 210 種組合濃縮在一個 130 行的 dispatch 函數中。
3. **Greedy knapsack packing + multi-modal 相容的資料管線** — packing 發生在 dataset tokenization 層而非 collator 層，透過 `PackingParams` 精準追蹤 media 的 sub-sequence 歸屬，支援 Flash Attention 2 的 unpad 處理。

## 候選 Pattern
- **patched-model-loading**: 在模型載入的不同生命週期階段（config→model→adapter）應用 patch 的設計模式。已在 vLLM 中觀察到類似設計，建議持續追蹤在其他 ML 框架的出現情況。

## 品質檢查摘要
- 各檔字數:README=961 / 1-arch=3584 / 2-walk=2653 / 3-pat=3271 / 9-q=1095
- Mermaid 圖數量:1-arch=2 張、2-walk=1 張、3-pat=0 張
- path:line 引用總數:52
- 量化資訊出現次數(版本、規模、效能等具體數字):15+
- 比較表格出現次數:6

## 執行決策日誌
- REPO_SLUG 決定與衝突檢查結果: `hiyouga__llama-factory`，無衝突
- 類別確認:CATEGORY=llm-training / TEMPLATE=ml-dl.md
- 類別決策說明:LLaMA-Factory 是 LLM 微調框架（LoRA/QLoRA/RLHF/DPO），明確符合 llm-training 類別定義
- 新增類別時的附帶動作:無，使用既有類別
- 關鍵入口檔案選擇與排除理由:選擇了 CLI→launcher→tuner 的調度鏈、model loader 和 template 系統，排除 Web UI（LlamaBoard）和 API server 因為偏向應用層而非核心架構
- 競品識別:Axolotl、TRL、Unsloth
- subagent 使用情況:使用 3 個平行子代理分析訓練系統、模型載入系統、資料管線與配置系統
- 工具替換:無
- 模板章節未明確規則的處理:1-architecture.md 增加「關鍵設計決策分析」區塊，為模板中未明確要求但對讀者有價值的內容
- 任務 prompt 覆寫 AGENTS.md 的項目:無
- [UNVERIFIED] 標註的部分清單:
  - v1 保留理由推測（歷史包袱 vs 用戶需求）
  - greedy knapsack 選擇理由推測
  - 101+ template 的實際覆蓋率

## Git 身份驗證
- Commit author: ksmooi <kaishianmooi@gmail.com>
- Git config 確認指令輸出:
  ```
  ksmooi <kaishianmooi@gmail.com>
  ```

## 驗證版本說明
此 PR 為 Hermes Agent repo 學習功能，學習對象:hiyouga/LLaMA-Factory。
