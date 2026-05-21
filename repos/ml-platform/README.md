# ml-platform/ — MLOps 與實驗管理平台

**上層目錄**: [repos/](../README.md)

ML 實驗追蹤、pipeline 編排、feature store、model registry、MLOps 工具鏈。
學習重點在於「如何讓 ML 實驗可重現、可比較、可部署」的平台設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **實驗追蹤** | metrics / artifacts / params 怎麼記錄?run 的資料模型設計?compare / diff 功能? |
| **Pipeline 編排** | DAG 定義方式(decorator / YAML / code)?step 快取機制?失敗重跑策略? |
| **Feature Store** | offline vs online feature 的讀寫語意?point-in-time correctness?feature serving latency? |
| **Model Registry** | model 版本管理?stage(staging / production)轉換?lineage 追蹤? |
| **可重現性** | environment snapshot?data versioning 整合?seed / determinism 保證? |
| **Serving 整合** | 從 registry 到 serving 的流程?A/B test 支援?rollback 策略? |
| **Plugin / 擴充** | 自訂 backend(storage / tracking / orchestrator)?hook 機制? |

套用模板:`_templates/ml-dl.md`

---

## 典型專案

| 專案 | 定位 | 特色 |
|------|------|------|
| MLflow | 全功能 ML 平台 | tracking / registry / serving 一體、廣泛生態整合 |
| ZenML | pipeline 框架 | stack-based、cloud-agnostic、production 導向 |
| Metaflow | Netflix 出品 pipeline | Python-native、S3-backed、data flow 清晰 |
| Feast | feature store | offline / online 雙軌、point-in-time join、多後端 |
| DVC | data version control | Git-based、remote storage、pipeline CI/CD |
| Weights & Biases | 實驗追蹤 SaaS | sweeps / artifacts / reports、廣泛框架支援 |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 實驗追蹤、pipeline 編排、feature store | ✅ `ml-platform` | — |
| 訓練 codebase 本體(FSDP / trainer) | — | ✅ [`llm-training`](../llm-training/) |
| 資料 ETL / 前處理 pipeline | — | ✅ [`data-pipeline`](../data-pipeline/) |
| 通用的工作流編排引擎(非 ML 專用) | — | ✅ [`infra`](../infra/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
