---
repo: ClickHouse/ClickHouse
file: 9-questions
studied_at: 2026-05-23
commit_sha: 72b4ed7
---

# ClickHouse · 未解問題

## 還沒搞懂的設計決策

- [ ] **為什麼 split part 不是由 query planner 決定，而是由 storage 層的 `MergeTreeDataSelectExecutor::read()` 獨自決定？**
  - 我目前的推測：歷史因素 — MergeTree 的 read API 在 Planner 出現前就存在了，而且 part pruning 跟 storage engine 的內部知識（partition layout、index granularity）高度耦合，拉到 planner 層反而破壞抽象邊界
  - 相關程式碼：Planner 只是產生 `ReadFromMergeTree` step，實際的 part 選擇邏輯在 `MergeTreeDataSelectExecutor.cpp`

- [ ] **MemoryTracker 的 `amount` 使用 `std::atomic<Int64>`，在極高頻 allocation 時的性能 overhead 到底多少？**
  - 我目前的推測：[UNVERIFIED] 根據 cache line miss vs atomic increment 的 cost 分析，在 single-socket 機器上 overhead 應 < 5%，但在 NUMA 機器上跨 socket atomic increment 的 penalty 可能更高
  - ClickHouse 社區曾討論過使用 thread-local counters + periodic flush 的方案，但最終選擇了全域 atomic

- [ ] **為什麼 planner 產生的是 QueryPlan（step DAG）而不是直接產生 Pipeline（processor graph）？多一層抽象的 cost 值得嗎？**
  - 我目前的推測：[UNVERIFIED] QueryPlan 可以被序列化（用於 EXPLAIN PLAN）、被不同 executor 消費、被 cache。多一層抽象的 cost 是啟動時多一次轉換，但 QueryPlan → Pipeline 的轉換是 O(steps) 而非 O(processors)（一個 step 可能展開成多個 processor）

- [ ] **Distributed table 的查詢下推到底能推到多深？什麼情況下會讓 initiator 做較多工作？**
  - 我目前的推測：`QueryProcessingStage` 控制下推深度。理想情況是「完全下推」（每個 shard 做完聚合再回傳），但以下情況會讓下推不完整：
    1. 涉及 cross-shard 的 DISTINCT 或排序
    2. 函式不支援 remote pushdown
    3. 查詢中有子查詢引用其他表

- [ ] **為什麼選擇 ZooKeeper 而非內建共識協議來管理複製協調？事後開發 ClickHouse Keeper 是否暗示了這個決定是錯的？**
  - 我目前的推測：最早的 ClickHouse 是單節點系統，分散式功能是後來加的。使用 ZK 可以快速獲得可靠的 leader election + distributed queue，不需要自己實作共識演算法。事後做 Keeper 不是否定這個決定，而是「依賴外部 Java 程序」的運維成本超過了實作 Raft 的成本

- [ ] **Patch parts（MergeTreePartInfo::Kind::Patch）的實作在哪？它跟 regular mutation 的 performance 比較？**
  - 我目前的推測：[UNVERIFIED] Patch parts 應該是只儲存變更欄位的輕量 part，讀取時與原始 part 合併。類似 relational database 的「delta record」，但對應的程式碼在 sparse checkout 下不可見

- [ ] **ClickHouse 的 LLVM JIT 編譯到底涵蓋了哪些部分？只是 expression evaluation 還是包含更大的 scope？**
  - 相關程式碼：[`src/Interpreters/JIT/`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Interpreters/JIT/)

## 想問維護者的問題

- 你們是在什麼時間點決定開發 ClickHouse Keeper 的？是先有「ZK 很痛」的營運體驗，還是先有「我們能做更好的」技術自信？
- 如果今天從零開始設計分散式協調層，你們會選擇 embedded Raft（像 Keeper）還是依賴外部服務？
- MergeTree 的 merge 選擇演算法（selectPartsToMerge）需要平衡哪些維度？能不能抽象一篇決定 merge 策略的論文？
- `PipelineExecutor` 的死結檢測是怎麼做的？手動證明的嗎？

## 下次再看時的待辦

- [ ] 閱讀 `MergeTreeDataMergerMutator::selectPartsToMerge()` 的完整實作，理解 merge 選擇的啟發式演算法
- [ ] 對照 `StorageDistributed` 的 `read()` 實作，理解查詢下推的實際機制
- [ ] 讀取 `src/Interpreters/JIT/` 下的 LLVM 相關程式碼
- [ ] 對照 Arrow/Parquet 的 columnar format vs ClickHouse 的 Wide/Compact format

## 跨專案對照備忘

- **Immutable data part + background merge** 與 Delta Lake/Iceberg 的 compaction 策略高度相似。兩者的 metadata 層差異（ZK vs manifest file）是一個值得追蹤的 pattern
- **Processor pipeline + two-phase scheduling** 跟 LLVM 的 MachineFunctionPass 架構有概念上的相似：都是 DAG of transform passes，由一個 orchestrator 驅動

### 候選 Pattern（首次觀察）

以下設計在 ClickHouse 中觀察到，尚未在其他 repo 學習筆記中確認，是「候選 pattern」的起點：

| 候選 Pattern | 目前觀察 repo 數 | 說明 |
|---|---|---|
| Immutable Data Part + Background Compaction | 1（ClickHouse） | Data part 寫入後不可變，後台持續合併以優化讀取。Delta Lake / Iceberg 也有類似機制，但尚未列入學習對象 |
| Layer-Piercing Singleton Registration (register* 模式) | 1（ClickHouse） | 模組以函式註冊到中央 factory，而非使用 plugin 動態載入或 reflection。需要確認其他 C++ 專案是否普遍採用 |

以上 pattern 待後續學習 2 個以上的同類 repo 後，滿足 `_patterns/` 收錄門檻（3+ repo）時再升級。


