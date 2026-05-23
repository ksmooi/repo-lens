---
repo: ray-project/ray
file: 3-key-patterns
studied_at: 2026-05-23
commit_sha: 7ddc3fa
---

# Ray · 值得偷學的設計

## Pattern 1: 混合排程政策（Hybrid Scheduling Policy）

**是什麼**: Raylet 的預設任務排程策略。它不是單純的「選資源最充裕的節點」或「選資料所在的節點」，而是將叢集中的節點分為「可用」（已有 worker 且能執行）和「可行」（資源夠但需冷啟動 worker）兩類，優先選擇「可用」節點，並在 top-k 中隨機選擇。

**為什麼有效**: 有三個設計假設（寫在原始碼註解中）：(1) 冷啟動 Worker 行程開銷大；(2) 過載節點有吵鬧鄰居問題；(3) 對於 AI workload，資料傳輸成本通常小於 Worker 啟動成本。因此偏好在已有 Worker 的節點執行，避免反覆冷啟動。

**程式碼位置**: [`src/ray/raylet/scheduling/policy/hybrid_scheduling_policy.h:29-49`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/src/ray/raylet/scheduling/policy/hybrid_scheduling_policy.h#L29-L49)

**何時可以借用**:
- 你的系統也有「隨機請求抵達」、「節點同質性高」的特性
- Worker/Container 啟動成本遠大於資料傳輸成本
- 你需要一個不依賴全域佇列（避免瓶頸）的排程策略

**替代方案**: 一致性雜湊（簡單但不感知負載）、全域 FIFO 佇列（中央瓶頸）、單純的最少負載優先（驚群效應）。

**注意事項**: Hybrid Scheduling 假設節點間同質性高。若節點差異很大（如一臺 A100 + 兩臺 T4），你可能需要自訂 `scheduling_strategy`（Ray 支援 `PlacementGroupSchedulingStrategy`）。

---

## Pattern 2: GCS 集中式控制面 + Plasma 分散式資料面

**是什麼**: Ray 將系統的控制路徑（中繼資料、節點註冊、演員管理）集中到一個 GCS 服務，但將資料路徑（大型物件傳輸）分散到每個節點的 Plasma 共享記憶體中，透過點對點傳輸。

**為什麼有效**: 這個分離解決了分散式系統中最核心的權衡——**一致性 vs 效能**。中繼資料的一致性由單一 GCS 負責（透過 Redis 或 InMemory 後端），實現簡單且可預測；而大型物件的傳輸不經過中央瓶頸，避免 GCS 成為效能瓶頸。

**程式碼位置**: GCS — [`src/ray/gcs/gcs_server.h`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/src/ray/gcs/gcs_server.h)，Plasma Store — [`src/ray/object_manager/plasma/store.h`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/src/ray/object_manager/plasma/store.h)

**何時可以借用**:
- 你的分散式系統也有「頻繁的 metadata 查詢」+「偶爾的大型資料傳輸」的混合 workload
- 你可以承擔控制面單點故障的風險（或已有容錯機制）

**替代方案**: 全分散式（所有節點 peer-to-peer，無中央角色——如 Cassandra）、全集中式（所有資料走中央——經典 client-server）。

**注意事項**: GCS 是 SPOF（單點故障），雖然支援重啟後恢復，但重啟期間的中繼資料遺失會影響正在運行的任務。

---

## Pattern 3: Raylet 的二合一設計（排程器 + 物件管理器）

**是什麼**: 每個 Ray 節點上只有一個 Raylet C++ 行程，它同時扮演**本地排程器**（決定哪個 Worker 執行哪個任務）和**物件管理器**（管理 Plasma 物件的推拉與記憶體壓力）。

**為什麼有效**: 排程決策需要考量資料 locality——如果任務需要的物件在本機 Plasma，直接從共享記憶體讀取；否則需要從遠端拉取。合併兩者讓排程器能即時感知本機記憶體壓力與物件位置，無需跨行程查詢。

**程式碼位置**: [`src/ray/raylet/node_manager.h`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/src/ray/raylet/node_manager.h)

**何時可以借用**: 當兩個緊密耦合的職責（排程 + 物件管理）之間的通訊頻率極高，分離為獨立服務的 gRPC 延遲成本超過耦合帶來的風險時。

**替代方案**: 分離為獨立服務（排程服務 + 物件服務）——降低耦合但增加延遲；或是分散式排程（每個任務各自決策，如 Spark）——但需要更複雜的一致性機制。

**注意事項**: 耦合度高的直接後果是——Raylet 當機時整個節點停擺。容錯策略只能靠任務重試在其他節點重新執行。

---

## Pattern 4: Power of Two Choices 路由策略（Ray Serve）

**是什麼**: Ray Serve 的 Router 每次隨機選取兩個 replica，向它們查詢當前請求佇列長度，選擇負載較輕的一個，然後將請求轉發過去。

**為什麼有效**: 這是一個經典的「two choices suffice」結果——從 N 個選項中隨機選兩個，選較好的一個，其負載分佈效果遠優於完全隨機，且不需維護全域佇列。它避免了一致性雜湊（不感知負載）的缺點，同時沒有全域排程器（瓶頸）的風險。

**程式碼位置**: [`python/ray/serve/_private/request_router/pow_2_router.py:27`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/serve/_private/request_router/pow_2_router.py#L27)

**何時可以借用**:
- 你的系統有 N 個無狀態的 worker/replica
- 請求大小不一，無法用 round-robin 均勻分配
- 你不想引入中央佇列或 dispatcher

**替代方案**: 一致性雜湊（簡單但無負載感知）、全域 FIFO 佇列（中央瓶頸）、最少連線優先（需要全域狀態）。

**注意事項**: 每次路由都需向兩個 replica 發送查詢（通常是極輕量的 gRPC/RPC），在 replica 眾多時查詢延遲會累積。Ray Serve 的 Router 使用 `DeploymentHandle._call_queued` 取得 replica 佇列長度。

---

## Pattern 5: 邏輯計畫 → 物理計畫 → 串流執行（Ray Data）

**是什麼**: Ray Data 將查詢處理分為三層：(1) **LogicalPlan** — 運算子 DAG（Read → Filter → MapBatches），純語意層，不涉及執行細節；(2) **Planner** — 將邏輯運算子轉為物理運算子（`PhysicalOperator`），加入規則式最佳化（投影下推、謂詞下推）；(3) **StreamingExecutor** — 在獨立的 thread 中執行排程迴圈，使用 `ray.wait()` 非同步監聽 task 完成。

**為什麼有效**: 這個模式解決了「使用者友善性」和「執行效率」之間的衝突——使用者寫的 `ds.map_batches(fn).filter(pred)` 看起來是 chain call，但內部被視為一個可優化的 DAG。邏輯/物理分離讓最佳化可以在這兩層之間插入，而串流執行確保 block 層級的管線化（不用等所有上游完成才開始下游）。

**程式碼位置**: Dataset — [`python/ray/data/dataset.py:196`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/data/dataset.py#L196)，StreamingExecutor — [`python/ray/data/_internal/execution/streaming_executor.py:100`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/data/_internal/execution/streaming_executor.py#L100)

**何時可以借用**: 任何需要「使用者寫 chain API，但內部需要最佳化執行」的系統——例如資料處理引擎、查詢建構器、pipeline 編排。

**替代方案**: Eager execution（立即執行——簡單但無法最佳化）、單層 Lazy DAG（如 Spark RDD lineage——可最佳化但邏輯/物理混合）。

**注意事項**: 三層架構增加了程式碼複雜度。只有當你真的需要在中間插入多種最佳化規則時，才值得付出這個複雜度。

---

## Pattern 6: Shared Memory Zero-Copy via mmap（Plasma Object Store）

**是什麼**: Plasma 是一個基於 `mmap` 的共享記憶體物件儲存。當一個 Worker 產生物件（任務結果），它將資料直接寫入 `mmap` 區域，並 Seal（標記為不可變更，是一個原子操作）。其他進程（同節點的其他 Worker、Driver）可以直接讀取同一記憶體頁面，不需要拷貝。

**為什麼有效**: Python 中傳遞大資料（>100KB）的代價中，序列化/反序列化通常佔主導，而不是傳輸本身。透過 mmap，Worker 對結果的「寫入」在共享記憶體中完成，讀取端看到的是同一個實體記憶體頁面（頁面映射）。對於數 MB 的 tensor 或 dataframe，這可以省下數毫秒的複製時間。

**程式碼位置**: [`src/ray/object_manager/plasma/store.h`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/src/ray/object_manager/plasma/store.h) — `PlasmaStore::Create`, `PlasmaStore::Get`, `PlasmaStore::Seal`

**何時可以借用**:
- 你的 Python 系統在同一個節點上有多個行程需要共享大資料
- 你追蹤過瓶頸確實是序列化/反序列化
- 你的資料大小相對穩定（mmap 適合固定大小物件）

**替代方案**: 透過 Redis 傳遞（網路延遲 + 序列化開銷）、透過檔案系統傳遞（IO 延遲）、凍結共享記憶體（Python `multiprocessing.shared_memory`）。

**注意事項**: Plasma 的「零拷貝」只針對同節點。**跨節點傳輸**仍然需要經過網路序列化（Raylet 的 Object Manager 負責拉取遠端 Plasma 資料）。`ray.get()` 的 transparent remote fetch 是一個讓開發者感覺「一切都是透明的」的優雅設計，但跨節點時不是 zero-copy。

---

## Pattern 7: Auto Init Hook — 選擇性的惰性初始化

**是什麼**: Ray 的部分公開 API（`ray.get()`、`ray.put()`、`ray.wait()`、`ray.kill()` 等）在首次呼叫時如果尚未 `ray.init()`，會自動觸發初始化。而另一部分 API（`ray.remote()`、`ray.shutdown()`、`ray.init()` 本身）則不會。

**為什麼有效**: 這是一個**有選擇性的惰性初始化**設計。常見的 pattern 是「要用的時候再 init」（如 `pytest`、各種 client libraries），但 Ray 做的更細——它把 API 分為「應該自動 init」和「不應該自動 init」兩組。前者的使用者體驗更好（新手不必先呼叫 `ray.init()`），後者避免在沒有叢集時做出無意義的行為（例如 `ray.remote()` 在沒有叢集時只是裝飾器，不應觸發初始化）。

**程式碼位置**: [`python/ray/_private/auto_init_hook.py`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/_private/auto_init_hook.py)，呼叫處在 [`python/ray/__init__.py:245-248`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/__init__.py#L245-L248)

**何時可以借用**:
- 你的 library 有一個昂貴的初始化步驟，但不是每個 API 都需要它
- 你想讓新手使用者體驗流暢，同時不犧牲進階使用者的精準控制

**替代方案**: 全部顯式 init（使用者體驗差）、全部自動 init（浪費資源）、使用 decorator 統一處理所有 API。

**注意事項**: 自動 init 的副作用是**隱藏了叢集連接的錯誤**。如果叢集連接失敗，首次呼叫 `ray.get()` 時才噴錯，而不是在 `ray.init()` 時，這讓除錯變得更難。

## API 設計品味的觀察

- **裝飾器作為核心抽象**: `@ray.remote` 是 Ray 最重要的 API 設計選擇。它利用 Python 的裝飾器語法，讓「把一段程式碼變成分散式執行」看起來是**改一個單詞的事情**。這跟 `asyncio` 的 `async/await` 有類似的心理效應——降低使用者的認知負擔。
- **非同步回傳 `ObjectRef`**: 所有 `f.remote()` 立即回傳 `ObjectRef`（類似 Future/Promise 模式）。使用者可以選擇 `ray.get()`（阻塞）、`ray.wait()`（阻塞但可超時）或將 ObjectRef 傳遞給其他任務（Ray 會自動解析依賴，無需顯式 `get` 再傳遞）。
- **從 `@ray.remote` 到高階 Library 的層層堆疊**: `Deployment` 內部是 `Actor`，`Dataset.transform()` 內部是 `remote task`，`Trainer.fit()` 內部是 `remote task + actor`。每一層抽象都建在上一層之上，不使用特殊的 IPC 機制。這讓 Debug 時可以降級到底層 API。
- **寬鬆的驗證策略**: 大多數參數驗證推遲到排程時才做，而非在呼叫 `remote()` 時。這意味著參數錯誤只有在任務實際提交到 Raylet 時才會被發現。

## 對相容性的態度

- **DeprecationWrapper**: [`python/ray/__init__.py:150-171`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/__init__.py#L150-L171) — `_DeprecationWrapper` 對被棄用的私有屬性訪問發出警告，但不立即中斷
- **`__all__` 的顯式管理**: `__init__.py` 末尾有 `assert set(__all__) == AUTO_INIT_APIS | NON_AUTO_INIT_APIS` — 確保 `__all__` 與實際分類一致，這是一個不錯的防護措施
- **SemVer 寬鬆**: Ray 雖然採用語意化版本，但 major version（目前 2.x）的跳躍並不頻繁，新功能多透過 minor/patch 發布
