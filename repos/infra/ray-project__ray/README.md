---
repo: ray-project/ray
file: README
studied_at: 2026-05-23
commit_sha: 7ddc3fa
type: infra
language: Python
framework: C++ (core), Python (API)
stars: 42.6k
status: active
---

# Ray · 概覽

## 解決什麼問題

在 GPU 叢集上寫分散式 Python 程式極度困難。Ray 提供的抽象是：你寫一般的 Python 函數，加上 `@ray.remote` 裝飾器，它就變成可在任意節點上非同步執行的任務。核心設計假設是**把分散式系統的複雜性封裝在一個底層 runtime 中**，讓上層的 AI 應用（訓練、服務、資料處理、超參數調優）不需要關心底層的排程、容錯、物件傳輸。

## 為什麼值得研究

Ray 是**唯一一個同時提供低階分散式 primitives（task/actor/object）和高階 ML 框架（Serve、Data、Train、Tune、RLlib）的專案**，且所有高階框架都是建在同一個 C++ runtime 之上的 Python API。這讓 Ray 成為一個絕佳的案例，可以用來觀察：

- **分散式系統 runtime 的設計權衡**——集中式控制面(GCS) vs 分散式資料面(Plasma)，混合排程策略，零拷貝物件傳輸
- **API 抽象層次的分層**——如何從底層 `ray.remote` 向上建構 Serve 的 deployment abstraction 和 Data 的 dataset abstraction
- **C++ runtime + Python API 的最佳實務**——pybind11 binding、gRPC 通訊、共享記憶體
- **42k+ stars 的開源治理**——Anyscale 贊助、擴展到 LLM 領域的產品策略

## API 風格一句話

`@ray.remote` 裝飾器驅動 + 非同步 task/actor 模型，搭配 `ray.get()` / `ray.put()` / `ray.wait()` 作為同步與資料傳遞基元。

## 技術棧一句話

Python + C++ (Raylet core) + gRPC (跨行程通訊) + Protocol Buffers (序列化) + Arrow (零拷貝資料格式) + Bazel (建構系統)

## 健康度信號

- ⭐ Stars: ~42.6k
- 📅 最後 commit: 2026-05-23（每日活躍）
- 📝 語言: Python (4,700+ .py files) + C++ (900+ cc/h files) + Java + TypeScript
- 🔄 License: Apache-2.0
- 🏢 贊助方: Anyscale（創辦人即 Ray 原創團隊）

## 跟其他分散式計算框架的比較

| 面向 | Ray | Apache Spark | Dask | Celery |
|------|-----|-------------|------|--------|
| 主要抽象 | task + actor + object | RDD / DataFrame | graph of delayed tasks | task queue |
| 排程粒度 | 單一 function call | dataset partition | array chunk | message |
| Actor 支援 | ✅ 一等公民，可遷移 | ❌ | ❌ | ❌ |
| 物件儲存 | Plasma (共享記憶體, zero-copy) | Tungsten (off-heap) | shared memory (較輕量) | broker (Redis/RabbitMQ) |
| 容錯模型 | 任務重試、演員重啟、物件重建 | lineage-based recompute | task retry | worker prefork |
| 高階 ML 框架 | Serve, Train, Tune, Data, RLlib | MLlib, Structured Streaming | Dask-ML, XGBoost | none |
| 主要 trade-off | 排程決策偏好在已有 worker 的節點執行（避免冷啟動），而非 locality-aware | 偏好資料 locality（RDD lineage），但 shuffle 昂貴 | 輕量但沒有 actor model | 成熟穩定但只適合非同步任務 |
| 語言 | Python + C++ (core) | Scala (core), Python API | Python (pure) | Python |
| 適用場景 | AI workload、分散式 ML、模型服務 | 大規模資料 ETL、SQL 分析 | 資料科學、陣列計算 | 非同步任務處理 |

## 我會在後續筆記中回答的問題

- Ray 的分散式 runtime 是怎麼從 `@ray.remote` 到任務實際在遠端節點執行的？
- Raylet 的混合排程政策為什麼偏好「已有 worker 的節點」而非「資料所在的節點」？
- GCS (Global Control Service) 的集中式設計會不會是瓶頸？它怎麼容錯？
- Plasma 的零拷貝物件傳輸對 Python 使用者是透明的——內部怎麼做到的？
- Ray Serve 的 Power of Two Choices 路由策略跟一致性雜湊比起來好在哪？
- Ray Data 為什麼選擇「邏輯計畫 → 物理計畫 → 串流執行」的三層設計？
