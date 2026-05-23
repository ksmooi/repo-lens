---
repo: ClickHouse/ClickHouse
file: README
studied_at: 2026-05-23
commit_sha: 72b4ed7
type: infra
language: C++
framework: Poco, NuRaft, libc++
stars: 47.5k
status: active
---

# ClickHouse · 概覽

## 解決什麼問題

ClickHouse 是一個**即時 OLAP（Online Analytical Processing）資料庫**，專為大規模分析查詢設計。它的核心承諾是：「對數十億行的資料，在秒級內回傳聚合結果。」與傳統行式資料庫（PostgreSQL、MySQL）不同，ClickHouse 以**欄位式（columnar）儲存**為基石，搭配向量化執行與稀疏索引，讓分析型查詢比行式資料庫快 100-1000 倍。

## 為什麼值得研究

ClickHouse 是開源 OLAP 領域的標竿專案（47.5k stars），它的設計選擇反映了「極致效能 × 分散式可靠 × 營運簡便」之間的一系列深刻取捨：

- **欄位式儲存的實戰範本** — 不是學術論文的 columnar storage，而是經過八年生產驗證的實作，從資料壓縮、稀疏索引到向量化執行都有完整解答
- **Immutable data part + background merge**的儲存引擎典範，被 Delta Lake、Iceberg 等湖格式借鑑
- **全端 C++ 打造的 OLAP 系統** — 從網路層（Poco）、協調層（自研 Raft）、儲存層（MergeTree）到查詢層（Planner + pipeline），全部自製
- **分散式設計的 pragmatism** — 不做強一致（最終一致、「shared-nothing + ZooKeeper」）、不犧牲效能換 ACID，反映「分析型場景不需要 OLTP 等級的一致」

## 技術棧一句話

`C++20` + `Poco` + `NuRaft` + `ZooKeeper` + `LLVM` (向量化引擎) + `SIMD` (SSE4.2 / AVX2)

## 健康度信號

- ⭐ Stars: ~47,500
- 📅 最後 commit: 2026-05-23（每日活躍）
- 📦 最新版本: v26.5（2026-05-21，每月發版）
- 📈 Release 頻率：**每月一次**，極度規律
- 🔄 SemVer 遵循狀況：使用 **CalVer**（YY.M），但有相容性保證
- 🏢 母公司：ClickHouse Inc.，主要維護者為公司全職工程師

## 與競品的比較

| 面向 | ClickHouse | DuckDB | Apache Druid | TimescaleDB |
|---|---|---|---|---|
| **定位** | 分散式 OLAP | 嵌入式 OLAP | 即時 OLAP | 時序型 |
| **架構** | Shared-nothing + ZK | 單程序嵌入式 | 多服務（Broker/Historical/Coordinator） | PostgreSQL extension |
| **儲存引擎** | MergeTree (immutable part + merge) | 自訂 columnar + vectorized | 多層儲存 (Hot + Deep) | Hypertable (PG-based) |
| **部署方式** | 叢集或多個節點 | 單一執行檔 | 多服務 JVM 叢集 | 單節點或 PG cluster |
| **SQL 支援** | 已擴充 SQL（非標準） | 完整 SQL | 已擴充 SQL | 完整 SQL (PostgreSQL) |
| **冷啟動 < 200ms?** | 否，連線建立需數十 ms | 是，嵌入零延遲 | 否 | 是（透過 PG） |
| **單機查詢吞吐** | 數百 MB/s ~ GB/s | MB/s ~ GB/s | 視設定 | 視 PG 設定 |
| **資料更新** | 不支援高效單列 UPDATE（透過 mutation） | 完整 DML | 時序 append-only | 完整 DML |
| **主要 trade-off** | 分析極快但寫入後一致性弱（最終一致） | 單機強但無法水平擴展 | 即時寫入佳但架構複雜 | ACID 強但分析效能不如專用 |
| **去中心化程度** | 中等（ZooKeeper 為 SPOF 心理負擔） | 不適用 | 低（多服務架構） | 低（PG 中心化） |
| **Delta / Iceberg 相容** | 有限（S3 後端可用） | 有（原生支援） | 有（Hive/Iceberg） | 無 |

## 我會在後續筆記中回答的問題

- MergeTree 的 immutable part 架構如何兼顧寫入吞吐與查詢效能？
- ClickHouse 的 query planner 跟傳統資料庫的 optimzer 有什麼不同？
- 為什麼選擇 shared-nothing + ZooKeeper 而不是 Raft-based 分散式共識？
- ClickHouse 的向量化執行引擎（LLVM + SIMD）實際上是怎麼工作的？
- 「假設一次查詢數百萬行只回傳前幾行」這個核心假設如何影響了全系統的設計？
