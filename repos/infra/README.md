# infra/ — 基礎設施與分散式系統

**上層目錄**: [repos/](../README.md)

分散式系統、容器編排、資料庫引擎、可觀測性工具。
學習重點在於「如何在不可靠的硬體上建構可靠的系統」的核心設計原理。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **分散式協調** | consensus 演算法(Raft / Paxos)?leader election?membership 管理?partition tolerance? |
| **Storage Engine** | WAL 設計?LSM tree vs B-tree?MVCC 實作?compaction 策略? |
| **Scheduler** | task / resource 排程策略?preemption?locality-aware?SLO-driven? |
| **網路層** | RPC 框架選擇?serialization?connection pool?retry / circuit breaker? |
| **可觀測性** | metrics 採集(pull vs push)?trace propagation?log pipeline?cardinality 控制? |
| **Operator 模式** | CRD 設計?reconcile loop?error handling?status condition? |
| **資料一致性** | 讀寫一致性 level?cross-shard transaction?2PC / saga 模式? |

套用模板:`_templates/library.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| Ray | 分散式計算框架 | actor model、task graph、AI workload 優化 |
| Temporal | 工作流引擎 | durable execution、saga、event sourcing |
| ClickHouse | OLAP 資料庫 | 列式儲存、vectorized execution、MergeTree |
| Qdrant | 向量資料庫 | HNSW index、payload filter、分散式部署 |
| OpenTelemetry | 可觀測性標準 | trace / metrics / logs 三合一、vendor-neutral |
| etcd | 分散式 KV | Raft consensus、watch API、Kubernetes 後端 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 分散式系統、資料庫引擎、可觀測性平台 | ✅ `infra` | — |
| 向量資料庫(以 RAG pipeline 整合為主) | — | ✅ [`rag`](../rag/) |
| ML 工作流編排(以 ML 實驗為中心) | — | ✅ [`ml-platform`](../ml-platform/) |
| 通用函式庫(async runtime、networking) | — | ✅ [`library`](../library/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
