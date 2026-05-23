## 學習對象

- Repo: [ClickHouse/ClickHouse](https://github.com/ClickHouse/ClickHouse)
- Commit: `72b4ed7` (2026-05-23)
- 語言 / 主要技術: C++ / Poco / NuRaft
- 專案類型 / 類別: infra (套用模板: library.md, 依 infra 指引調整)
- 學習深度: standard
- 筆記路徑: `repos/infra/clickhouse__clickhouse/`

## 關鍵入口檔案

| 檔案 | 選擇理由 |
|---|---|
| `programs/main.cpp:370` | 單一二進位 dispatch 入口，所有子命令（server/client/local/keeper）由此分發 |
| `programs/server/Server.cpp:1348-1360` | Server 啟動流程核心：註冊所有元件 → Context → loadMetadata → 啟動 services |
| `src/Interpreters/executeQuery.cpp:2268` | 查詢執行總入口：parse → analyze → plan → execute 五階段管線 |
| `src/Storages/MergeTree/MergeTreeData.h` | MergeTree 儲存引擎核心：immutable part 管理、transaction、multi-index container |
| `src/Planner/Planner.h:27` | 查詢 Planner：將 QueryTree 轉換為 QueryPlan (step DAG) |

## 三個最重要的發現

1. **Immutable Data Part + Background Merge** — ClickHouse 最根本的架構決定。寫入的資料以不可變的 data part 為單位，讀取完全無鎖。後台 merge 程序持續將小 part 合併為大 part，以寫入放大換讀取效率。這個模式被 Delta Lake/Iceberg 等湖格式借鑑。
2. **Processor Pipeline + Two-Phase Scheduling** — 查詢處理建模為 `IProcessor` 的有向圖，每個 processor 有 `prepare()` / `work()` 兩階段生命週期。PipelineExecutor 非同步排程，支援多執行緒平行執行、背壓處理、動態管線修改。
3. **Shared-nothing + ZooKeeper (→ Keeper)** — ClickHouse 選擇不做強一致、不做 distributed transaction，使用 ZK 做輕量協調。後來開發 ClickHouse Keeper（NuRaft-based）替代 ZK，讓叢集只需一種二進位檔。

## 候選 Pattern

| 候選 Pattern | 目前觀察 repo 數 | 說明 |
|---|---|---|
| Immutable Data Part + Background Compaction | 1（ClickHouse） | Data part 寫入後不可變，後台持續合併。Delta Lake / Iceberg 也有類似機制 |
| Layer-Piercing Singleton Registration (register* 模式) | 1（ClickHouse） | 模組以函式註冊到中央 factory，而非使用 plugin 動態載入或 reflection |

## 品質檢查摘要

- 各檔字數: README=3921 / 1-arch=14499 / 2-walk=10654 / 3-pat=13046 / 9-q=4504 bytes
- Mermaid 圖數量: 1-arch=4 張、2-walk=1 張、3-pat=0 張
- path:line 引用總數: 8 處
- 量化資訊出現次數（版本、規模、效能等具體數字）: 20+ 次
- 比較表格出現次數: 1 次（README, vs DuckDB/Druid/TimescaleDB）

## 執行決策日誌

- REPO_SLUG 決定與衝突檢查結果: `clickhouse__clickhouse`，無衝突
- 類別確認: CATEGORY=infra / TEMPLATE=library.md（依 infra 指引調整）
- 類別決策說明: ClickHouse 是分散式 OLAP 資料庫引擎，歸類於 infra
- 關鍵入口檔案選擇與排除理由: 聚焦查詢執行管線（executeQuery → Planner → Processor Pipeline）與儲存引擎（MergeTree），排除測試（40k 檔案）、文件、第三方函式庫
- 競品識別: DuckDB（嵌入式 OLAP）、Druid（即時 OLAP）、TimescaleDB（時序型 PG extension）
- subagent 使用情況: 使用 3 個 subagent 並行分析（查詢管線 / MergeTree / 分散式架構）
- 工具替換：使用 `sparse-checkout` 代替完整 clone，因 ClickHouse 有 40,045 個檔案
- 模板章節未明確規則的處理: library.md 模板原為 library/SDK 設計，"infra" 類別依 AGENTS.md 指引做以下調整：新增分散式架構（Shared-nothing）、資源隔離（MemoryTracker）、協調層（ZK/Keeper）、儲存引擎架構（MergeTree）
- 任務 prompt 覆寫 AGENTS.md 的項目: 學習對象 clone 位置從 `.tmp/studies/` 改為 `.tmp/runs/20260523-130419-clickhouse__clickhouse/study/`
- [UNVERIFIED] 標註的部分清單:
  - MemoryTracker atomic overhead 在 NUMA 機器的影響
  - QueryPlan 到 Pipeline 的轉換成本
  - Patch parts 的實作細節
  - LLVM JIT 編譯範圍
- 規模警告: ClickHouse ~40k 檔案，因使用者未在時限內回應，自行決定繼續 standard 學習

## Git 身份驗證

- Commit author: ksmooi <kaishianmooi@gmail.com>
- Git config 確認指令輸出:
  (貼上 `git log -1 --format='%an <%ae>'` 的結果)

## 驗證版本說明

此 PR 為 Hermes Agent repo 學習功能，學習對象: ClickHouse/ClickHouse。
