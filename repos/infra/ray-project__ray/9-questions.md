---
repo: ray-project/ray
file: 9-questions
studied_at: 2026-05-23
commit_sha: 7ddc3fa
---

# Ray · 未解問題

## 還沒搞懂的設計決策

- [ ] **GCS 的 Write-Ahead Log 是怎麼實作的？**
  - 描述: 我知道 GCS 支援 persisted port 以便 crash 後恢復，但不清楚它的 WAL/持久化策略。是每個寫入都 fsync 嗎？還是 batch commit？
  - 我目前的推測: [`UNVERIFIED`] 可能使用 `AOF`-like 的 append-only 日誌模式，類似 Redis 的持久化策略，因為 GCS 的後端儲存本身就是 Redis。但我不確定 InMemory 模式是否有 WAL。
  - 相關程式碼: [`src/ray/gcs/gcs_server.h`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/src/ray/gcs/gcs_server.h)

- [ ] **`max_calls` 對 GPU task 設為 1 是 Python 層的 Cython extension memory leak 還是 CUDA 層的問題？**
  - 描述: Worker 預設無限 `max_calls`，但 GPU 任務預設為 1。這是因為 Python 的 GPU library（如 PyTorch）在 Worker 行程中無法完全釋放已分配的 CUDA memory，還是因為 `_raylet` 的 C++ binding 有 memory leak？
  - 我目前的推測: [`UNVERIFIED`] 比較像是 PyTorch CUDA caching allocator 的問題——當 Worker 行程持續存在時，已分配的 CUDA memory 不會返還給作業系統，即使 Python 物件已被 GC。但我不確定 Ray 官方文件是否有正式說明。

- [ ] **Ray 的 lineage-based 物件重建機制跟 Spark 的 lineage recompute 有什麼關鍵差異？**
  - 描述: Ray 的文件提到「若物件因節點失效遺失，GCS 可重新執行產生該物件的任務」。這聽起來很像 Spark 的 lineage recompute。
  - 我目前的推測: [`UNVERIFIED`] 關鍵差異可能在於 Ray 的 lineage 是 task-level（只重跑遺失物件的 producer task），而 Spark 的 lineage 是 partition-level（需要重跑整個 RDD partition 的 lineage graph）。但我不確定 Ray 在何時／如何持久化 task lineage metadata。
  - 相關程式碼: 可能跟 `GcsTaskManager` 和 `gcs/actor_manager.h` 中的任務/演員恢復機制有關

- [ ] **Raylet 和 Plasma 之間的 gRPC 通訊在大量物件時會不會成為瓶頸？**
  - 描述: 每個 Worker 執行完任務後，會將結果透過 Plasma Client（走 Unix domain socket）寫入 Plasma Store。同時間可能有數百個 Worker 在寫入。我推測 Plasma Store 使用單線程事件迴圈（類似 Redis 的設計），但沒有確認。
  - 我目前的推測: [`UNVERIFIED`] 從 `store.h` 的介面推測，PlasmaStore 很可能使用單線程 event loop（類似 `boost::asio::io_context`）處理所有 Create/Get/Seal 請求。如果推測正確，高併發時物件操作會序列化，但因為操作都是 O(1) 的記憶體頁面操作，單線程應該夠用。

- [ ] **Ray Serve 的 LongPoll 機制：狀態不一致窗口有多長？**
  - 描述: Controller 不主動推送 replica 狀態變更，而是由 Router/Proxy 定期輪詢。這意味著 replica 擴縮後有一段時間 Router 不知道。
  - 我目前的推測: [`UNVERIFIED`] 預設 polling interval 可能在 100-500ms 之間，意味著狀態不一致窗口最多 ~500ms。在這段時間內請求可能被路由到已縮減的 replica（請求失敗）或錯過新啟動的 replica（請求排隊）。
  - 相關程式碼: [`python/ray/serve/_private/long_poll.py`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/serve/_private/long_poll.py)

- [ ] **Ray Data 的串流執行器如何處理 backpressure 與 OOM？**
  - 描述: StreamingExecutor 使用 `ResourceManager` 根據 CPU/GPU/記憶體進行資源感知排程，但我不清楚具體的 backpressure 機制——當 Producer 產生 block 的速度快於 Consumer 消費時，誰負責暫停？
  - 我目前的推測: [`UNVERIFIED`] 從 `OutputBackpressureGuard` 的名稱來看，backpressure 是在 operator 的輸出端實作——當輸出 RefBundle 佇列達到閾值時，上游 operator 的 `get_next()` 會阻塞。
  - 相關程式碼: [`python/ray/data/_internal/execution/interfaces/physical_operator.py`](https://github.com/ray-project/ray/blob/7ddc3faa761ab533eaff081be4db7dcea683ea56/python/ray/data/_internal/execution/interfaces/physical_operator.py)

## 想問維護者的問題

- Ray 設計之初就有考慮到 AI workload 的「GPU 優化」嗎？還是說早期只是想做一個通用分散式 runtime，後來才發現 AI 場景最適合？
- `ray.remote()` 的 scheduling strategy 預設是 Hybrid Scheduling，你們在什麼情況下會建議使用者切換成 `PlacementGroupSchedulingStrategy`？
- Ray 2.x 系列的 roadmap 中，是否有計畫讓 Ray Serve 支援真正的 Serverless（scale to zero 且冷啟動 < 100ms）？
- 對於這個專案的巨大 codebase（10k+ 檔案），你們有沒有內部的 code organization 準則？哪些模組被認為是「核心」哪些是「附屬」？

## 下次再看時的待辦

- [ ] 探索 `src/ray/raylet/scheduling/policy/` 下除了 Hybrid 以外的排程政策（`SpreadSchedulingPolicy`、`RandomSchedulingPolicy`、`LocalitySchedulingPolicy`），閱讀它們的設計文件
- [ ] 對照 Spark 的 lineage recompute 機制和 Ray 的 task lineage recovery 的差異
- [ ] 深入看 `GcsActorManager` 的 actor 恢復邏輯——演員復活時會保留哪些狀態？
- [ ] 重看 `python/ray/serve/_private/router.py` 中的 Power of Two Choices 實作，確認 router 和 replica 之間的佇列長度查詢機制
- [ ] 理解 Ray 的 object spill to disk 機制——spill 到哪些目錄？spill 後的物件如何被拉回？

## 跨專案對照備忘

- **Hybrid Scheduling + Power of Two Choices** 都屬於「隨機 + 有限感知」類型的排程策略，在 Google 的 Omega 論文（隨機兩次選擇）和多篇 load balancing 文獻中出現過。目前 `_patterns/` 中還沒有這個 pattern，但 Ray + Spark（spark.scheduler.allocation.file）+ Kubernetes scheduler 都可以算是這類策略的實例。→ **候選 pattern: randomized-spread-scheduling**，等觀察到第 3 個 repo 再寫入 `_patterns/`
- **邏輯/物理計畫分離 + 串流執行**（Ray Data + Spark Catalyst + SQLite 的 query planner）——屬於經典的 query optimization 模式。在 Ray Data 的實作特點是串流層面的管線化。→ 等觀察到第 3 個 repo 再確認是否為 `_patterns/` 候選
- **Shared memory zero-copy**（Ray Plasma + Apache Arrow Flight + Redis Cluster 的 direct I/O 在某些場景）——不是新概念，但 Ray 把它跟分散式物件傳輸綁在一起值得注意。已收錄在 `references/architecture-patterns-from-practice.md`
