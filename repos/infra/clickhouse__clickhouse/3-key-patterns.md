---
repo: ClickHouse/ClickHouse
file: 3-key-patterns
studied_at: 2026-05-23
commit_sha: 72b4ed7
---

# ClickHouse · 值得偷學的設計

## 目錄

- [Pattern 1: Immutable Data Part + Background Merge](#pattern-1-immutable-data-part--background-merge)
- [Pattern 2: Processor Pipeline + Two-Phase Scheduling](#pattern-2-processor-pipeline--two-phase-scheduling)
- [Pattern 3: Multi-Index Part Container (boost::multi_index)](#pattern-3-multi-index-part-container-boostmulti_index)
- [Pattern 4: Layer-Piercing Singleton Registration (the "register*" pattern)](#pattern-4-layer-piercing-singleton-registration-the-register-pattern)
- [Pattern 5: Hierarchical Memory Tracking with Atomic Layout](#pattern-5-hierarchical-memory-tracking-with-atomic-layout)
- [Pattern 6: Skip Index Evaluation via RPN](#pattern-6-skip-index-evaluation-via-rpn)
- [API 設計品味的觀察](#api-設計品味的觀察)
- [對相容性的態度](#對相容性的態度)

---

## Pattern 1: Immutable Data Part + Background Merge

**是什麼**：寫入的資料以不可變（immutable）的 data part 為單位，寫入後不可修改。後台有一個獨立的 merge 程序，持續將小 part 合併為大 part。

**程式碼位置**：
- [`MergeTreeData::Transaction::commit()`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/MergeTreeData.h#L355) — part 的兩階段提交（PreActive → Active）
- [`MergeTreeDataMergerMutator::mergePartsToTemporaryPart()`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/MergeTreeDataMergerMutator.h#L80) — 實際執行 merge 的方法
- [`BackgroundJobsAssignee`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/BackgroundJobsAssignee.h) — 背景 merge 排程器

**為什麼有效**：
- 讀取完全無需加鎖（沒有「正在寫入的 part」），支援完全平行的查詢
- Part 是「完整的目錄」，複製 = `cp -r`，大幅簡化分散式複製
- Merge 是「寫入放大換讀取效率」的經典 trade-off：小 part 查詢慢（大量隨機讀），合併後的大 part 掃描快（順序 + 更少的檔案）

**替代方案**：
- **LSM-tree 的 compaction**（RocksDB）：key-value 結構，compaction 按 level 觸發。MergeTree 的 merge 更聰明：可以根據 partition、size、level、mutations 等維度決定合併策略
- **Delta Lake / Iceberg 的 compaction**：類似，但 metadata 儲存在檔案目錄中而非外部 ZooKeeper

**何時可以借用**：
- 寫入頻繁（大量 append/batch insert）但讀取也需要高效工作的系統
- 不需要即時單列 UPDATE 的場景（分析、日志、監控）
- 使用 object store（S3）作為儲存層時，immutable file + background compaction 是最佳搭配

**注意事項**：
- 需要一個可靠的 metadata 管理層來追蹤哪些 part 是活躍的（ClickHouse 用 ZK，你自己得有等價方案）
- Merge 會造成「寫入放大」，高峰時期 merge 可能佔用大量 IO
- Immutable part 的 DELETE/UPDATE 需要 mutation 機制，實作複雜

---

## Pattern 2: Processor Pipeline + Two-Phase Scheduling

**是什麼**：查詢處理被建模為一個由 `IProcessor` 組成的有向圖，每個 processor 有 `prepare()` / `work()` 兩階段生命週期。PipelineExecutor 非同步排程這些 processor。

**程式碼位置**：
- [`IProcessor`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Processors/IProcessor.h) — 核心介面，定義 `prepare()` 和 `work()`
- [`PipelineExecutor`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Processors/Executors/PipelineExecutor.h) — 驅動 pipeline 的排程器
- [`IOutputPort` / `IInputPort`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Processers/Port.h) — processor 之間的連線

**為什麼有效**：
- `prepare()` / `work()` 分離讓排程器可以避免 busy-wait：如果 processor 的 input port 沒資料，它可以「休眠」直到前一個 processor 通知有新資料
- Processor 圖（而非線性管線）可以表達更複雜的拓撲（fork、join、multiple inputs）
- 天然的平行化單位：多個獨立的 processor 可以由不同執行緒同時執行

**替代方案**：
- **Volcano iterator model**（傳統資料庫）：每個 operator 是 pull-based iterator。ClickHouse 的 processor 更靈活，支援 push + pull 混合
- **Micro-batch pipeline**（Spark/Flume）：每個 stage 是 sync barrier。ClickHouse 的 continuous pipeline 延遲更低

**何時可以借用**：
- 設計一個需要執行 DAG-like task 的系統
- 資料處理管線需要複雜的 topology（fork/merge/join）
- 需要精確控制執行緒使用（非 parallel for-each）

**注意事項**：
- Processor pipeline 的排程是其最複雜的部分：死結檢測（port level 的 cycle）、背壓處理、動態修改（INSERT 情境）
- 效能分析困難：processor 的 latency 分佈非均勻，需補齊 profiling tooling

---

## Pattern 3: Multi-Index Part Container (boost::multi_index)

**是什麼**：MergeTree 儲存引擎使用 `boost::multi_index_container` 管理所有 data part，同時透過「part info」和「(state, info)」雙索引查詢。

**程式碼位置**：
- [`MergeTreeData::DataPartsIndexes`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/MergeTreeData.h#L1498-L1512) — 使用 boost::multi_index 定義雙索引
- [`ActiveDataPartSet`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/ActiveDataPartSet.h) — 基於 `MergeTreePartInfo` 的 active part 管理

**為什麼有效**：
- 相比用兩個 `std::unordered_map` 維護，multi_index_container 保證 part 物件只有一份拷貝，不會兩份容器狀態不一致
- 索引 1（ordered by Info）：支援 `partition_id` 範圍掃描，快速找出特定 partition 的所有 part
- 索引 2（ordered by State+Info）：rapid 找出所有 Active 或 Outdated 的 part

**替代方案**：
- 用多個 `std::map` + `std::set` 手動維護（容易同步 bug）
- 用 `std::vector` + `std::sort`（查詢 O(n log n) vs O(log n)）

**何時可以借用**：
- 需要根據不同屬性同時查詢同一個物件集合
- 物件的兩個索引都是 ordered（而非 hash）
- 記憶體可以容納數千到數十萬個物件

**注意事項**：
- `boost::multi_index_container` 的編譯時間長（template 爆炸）
- 迭代器操作方式與 STL 容器略有不同（`indices.get<0>()` vs `begin()`）
- Debug build 下效能可能明顯下降

---

## Pattern 4: Layer-Piercing Singleton Registration (the "register*" pattern)

**是什麼**：ClickHouse 在 server 啟動初期，系統性地呼叫一系列 `register*` 函式來註冊所有 functions、storages、formats、table functions、aggregate functions、dictionaries、disks 等元件。

**程式碼位置**：
- [`programs/server/Server.cpp:1348-1360`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/programs/server/Server.cpp#L1348) — 啟動時的註冊序列
- [`registerFunctions()`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Functions/registerFunctions.h) — 每個類別一個 `registerXxx()` 函式
- [`registerStorages()`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/registerStorages.h) — 儲存引擎註冊

**為什麼有效**：
- 極佳的模組邊界：每個元件自己負責初始化，不需要中央的「所有元件列表」
- 編譯期解耦：新增一個 function 不需要修改執行器、planner、或中央註冊表
- 執行期選擇：透過 `#if USE_ROCKSDB` / `#if USE_S3` 等編譯選項，可以完全排除不需要的元件

**替代方案**：
- **Plugin system**（PostgreSQL dynamic shared library）：可動態載入，但有安全風險（dlopen）。ClickHouse 的 `main.cpp` 甚至直接用全域 override 了 `dlopen()` 回傳 nullptr
- **Reflection + annotation**（Java Spring 的 component scan）：編譯期隱式，不需要顯式註冊

**何時可以借用**：
- C++ 專案需要類似「plugin」的擴充機制，但不想引入動態載入
- 模組數量多（ClickHouse 有數百個 functions），需要系統化管理

**注意事項**：
- 啟動時要呼叫大量 `register*()`，啟動時間會隨元件數線性增加
- 遺漏註冊 = 執行期找不到，會拋 `NOT_FOUND` 例外（需靠測試覆蓋）

---

## Pattern 5: Hierarchical Memory Tracking with Atomic Layout

**是什麼**：`MemoryTracker` 實現了一個樹狀的記憶體追蹤框架，每個查詢和執行緒都分配一個 tracker，形成 parent-child 關係。使用 `std::atomic<Int64>` 並做 cache line 優化。

**程式碼位置**：
- [`MemoryTracker`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Common/MemoryTracker.h#L60) — 核心類別
- `amount` (`:75`、`std::atomic<Int64>`) — 當前記憶體使用量
- `soft_limit` / `hard_limit` (`:82-83`) — 記憶體軟/硬限制
- `parent` (`:84`) — 父 tracker，形成樹狀

**為什麼有效**：
- 層級式架構讓「全域限制」和「查詢限制」可同時存在，child tracker 的 allocation 自動累計到 parent
- `std::atomic` 而非 mutex：在高頻 allocation 場景（OLAP 查詢每秒數萬次 `malloc`），mutex 會是災難
- Hot/cold field 拆分：`amount`、`peak`、`soft_limit`、`parent`（每個 allocation 都讀寫的欄位）放在 hot cache line；`fault_probability`、`sample_probability`（很少使用的設定）放在 cold area

**替代方案**：
- **mutex + 全域 counter**：精確但慢
- **TCMalloc 的 sampled allocation**：透過 hook malloc 取得統計，但無法做「查詢層級」的限制
- **Rust 的 `#[global_allocator]` + thread-local counters**：類似，但 Rust 的 ownership 模型讓層級式管理更簡單

**何時可以借用**：
- C++ 專案需要精確的 per-request / per-connection 記憶體控制
- 多執行緒高頻 allocation（每秒數十萬次）

**注意事項**：
- `std::atomic` 仍是 memory ordering cost（`memory_order_relaxed` 或 `memory_order_seq_cst` 取捨）
- 無法攔截 kernel-level 的記憶體（mmap 大檔案），需要用 `/proc/self/status` 的 RSS 輔助

---

## Pattern 6: Skip Index Evaluation via RPN

**是什麼**：ClickHouse 的跳躍索引使用 **Reverse Polish Notation（RPN）** 評估索引查詢的可行性。表達式被轉換為 RPN，縮短評估路徑以便快速跳過不需要的 mark range。

**程式碼位置**：
- [`MergeTreeIndices.h`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/MergeTreeIndices.h) — RPN 狀態與索引基類
- [`MergeTreeIndexGranularity`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/MergeTree/MergeTreeIndexGranularity.h) — index granularity（每個 mark 對應多少行）
- [`KeyCondition`](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/src/Storages/KeyCondition.h) — primary key 條件評估

**為什麼有效**：
- RPN 沒有括號歧義，評估是簡單的 stack machine，適合硬體高效執行
- RPN 的求值結果為 `TRUE / FALSE / ALWAYS_TRUE / ALWAYS_FALSE` 四值邏輯，支援短路評估（`RPNElement::evalAnd()`）
- 一次求值可以決定整個 mark range 是否需要讀取

**替代方案**：
- **Zonemap**（Parquet 最常見）：只儲存 min/max，評估時直接與查詢條件比較。RPN 更通用，支援任意索引類型
- **Bloom filter pushdown**（ORC）：只做 membership test。Skip index 支援 bloom filter、ngram、token 等多種型別

**何時可以借用**：
- 資料儲存需要有「分塊跳過」的能力（類似 Parquet 的 row group skipping），但想支援多種索引型別
- 查詢條件複雜（AND/OR 混合），需要通用的條件評估路徑

注意事項：
- 跳躍索引只在每個 mark range（預設 8192 行）的粒度工作，不是 row-level
- 索引建立和評估需要額外 CPU 和磁碟空間

---

## API 設計品味的觀察

ClickHouse 的 SQL 方言遵循「**對使用者寬容，對實作嚴格**」：

- **偏好非標準擴充而非標準相容** — `LIMIT 10 OFFSET 5` 支援，`SELECT ... FORMAT ...` 也是非標準。不追求 SQL-92 完整相容，而是讓常見的 OLAP 查詢語法更短
- **不要隱含的行為** — `SELECT count(*) FROM t` 在沒有 FROM 子句時不會自動推測資料來源（不像某些 embedded databases）
- **錯誤訊息有價值** — parse error 不只是「syntax error」，會指出精確位置和期望的 token

## 對相容性的態度

- 使用 CalVer（YY.M）但發布 **相容性保證**：小版本之間不引入 breaking change
- Deprecation 流程：先在 changelog 標註 deprecated，保留 3-6 個月，再移除
- 設定檔格式相容：舊版 config.xml 可在新版使用，但不保證反向
- 在 [CHANGELOG](https://github.com/ClickHouse/ClickHouse/blob/72b4ed7227d/CHANGELOG.md) 中有明確的「Backward Incompatible Change」區塊
