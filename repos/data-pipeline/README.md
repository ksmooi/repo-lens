# data-pipeline/ — 資料處理與 ETL

**上層目錄**: [repos/](../README.md)

資料處理、ETL、streaming pipeline、dataset 工程、data quality。
學習重點在於「大規模資料如何被可靠、可擴展地移動與轉換」的工程設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **Pipeline 建構** | declarative vs imperative 定義方式?DAG 的 dependency resolution?step 重用機制? |
| **Batch vs Streaming** | 何時 micro-batch?何時真正 streaming?watermark / late data 怎麼處理? |
| **Connector 架構** | source / sink connector 怎麼設計?schema 如何傳遞?credential 管理? |
| **Schema Evolution** | backward / forward 相容?migration 策略?schema registry 整合? |
| **Backfill / Incremental** | incremental 邏輯怎麼實作?state 怎麼持久化?idempotency 怎麼保證? |
| **Data Quality** | validation rule 怎麼定義?失敗怎麼處理(quarantine / halt)?品質指標監控? |
| **Scalability** | 單機 vs 分散式執行?partition 策略?memory 壓力下的 spill-to-disk? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| DLT | Python ETL 框架 | schema inference、增量載入、多 destination 支援 |
| Airbyte | 資料整合平台 | 300+ connector、ELT 模式、connector development kit |
| Apache Spark | 分散式計算框架 | RDD / DataFrame、大規模 batch、MLlib 整合 |
| Ray Data | 分散式 ML 資料 | streaming ingest、transform、與 Ray Train 整合 |
| Great Expectations | 資料品質框架 | expectation suite、data docs、checkpoint |
| Dagster | data orchestration | asset-based、lineage、embedded catalog |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 資料 ETL、streaming、dataset 工程 | ✅ `data-pipeline` | — |
| ML pipeline 編排(以 ML 實驗為中心) | — | ✅ [`ml-platform`](../ml-platform/) |
| 訓練用 DataLoader(PyTorch Dataset) | — | ✅ [`llm-training`](../llm-training/) |
| 通用工作流編排(非資料導向) | — | ✅ [`infra`](../infra/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
