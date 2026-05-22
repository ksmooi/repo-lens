---
repo: mlflow/mlflow
file: README
studied_at: 2026-05-22
commit_sha: 1c491e7
type: ml-platform
language: Python
framework: FastAPI / Flask / React
stars: 26.1k
status: active
---

# MLflow · 概覽

## 解決什麼問題

ML 的生命週期管理——從實驗追蹤、模型註冊、部署到監控——長期缺乏統一的工具鏈。研究團隊用 TensorBoard / W&B，工程團隊用自訂 CI/CD，production 監控又一套。MLflow 嘗試用一套一致的 API + server 架構，涵蓋 **experiment tracking → model registry → deployment → evaluation → observability（tracing）** 的完整光譜。

近年（2024-2026）的戰略轉向特別明顯：從傳統的 ML 實驗管理平台，擴展為 **AI engineering platform**，重點轉向 LLM tracing、evaluation、prompt management、AI Gateway。README 的描述已從 "MLflow is an open source platform for the complete machine learning lifecycle" 改為 "The open source AI engineering platform for agents, LLMs, and ML models"。

## 為什麼值得研究

這不是一個小巧的專案——`mlflow/` 目錄下 1,109 個 Python 檔案、~155K 行 code——但它的架構演進歷史本身就是一門課：

1. **從單體到模組化**：tracking store 的抽象層（`AbstractStore`）歷經 file → SQLAlchemy → REST → Databricks 多種實作，如何在不 breaking API 的前提下疊加新 backend。
2. **protobuf-driven API design**：所有 REST endpoint 從 `.proto` 定義產生，確保 client-server contract 的一致性——一個值得借鏡的模式。
3. **LLM tracing 的融入策略**：用 OpenTelemetry 作為底層標準，疊上 MLflow 的 span/trace 語意，而非自創 tracing 格式。這讓它能夠無縫整合 OTel 生態系。
4. **autolog 的 monkey-patch 機制**：`safe_patch` 架構如何在 20+ 個 ML framework（OpenAI、LangChain、PyTorch、scikit-learn…）上提供一致的 logging 體驗。

## 健康度信號

| 面向 | 值 |
|---|---|
| ⭐ Stars | 26,052 |
| 🔄 Forks | 5,765 |
| 📅 最近 commit | 2026-05-22（活躍） |
| 👥 贊助方 | Databricks |
| 📝 License | Apache-2.0 |
| 📦 每月下載量 | 60M+ |
| 🔧 主要語言 | Python、TypeScript（React UI） |
| 🧪 測試規模 | 1,015 個測試檔案 |
| 🏛️ 程式碼規模 | 155K LOC（Python）、2,934 JS/TS（前端） |

## 跟競品的比較

| 面向 | MLflow | Weights & Biases | Neptune | LangFuse |
|---|---|---|---|---|
| 部署模型 | 自架 server（OSS） | SaaS only | SaaS only | 自架（OSS）+ Cloud |
| Experiment tracking | ✅ 核心功能 | ✅ 核心功能 | ✅ 核心功能 | ❌ 無 |
| Model Registry | ✅ | ✅（W&B Artifacts） | ✅ | ❌ |
| LLM Tracing | ✅（OTel-based） | ✅（Weave） | ❌ | ✅ 核心功能 |
| Evaluation | ✅ 50+ built-in metrics | ✅ | ✅ | ✅ |
| Prompt Management | ✅ | ❌ | ❌ | ✅ 核心功能 |
| AI Gateway | ✅ | ❌ | ❌ | ❌ |
| 自架自由度 | ✅ 完全開源 | ❌ | ❌ | ✅ |
| 框架整合數 | 20+ | 10+ | 10+ | 少數 |

MLflow 的核心 trade-off：**廣度 vs 深度**。它在每個面向都「有」但不一定「最好」。W&B 的 experiment tracking UX 更流暢，LangFuse 的 tracing 更專門，但 MLflow 是唯一一個在單一 OSS 平台上涵蓋從實驗到 production 全流程的選擇。

## 我會在後續筆記中回答的問題

- [ ] Store 抽象層如何做到 file → SQL → REST → Databricks 四種 backend 無痛切換？
- [ ] `safe_patch` 架構如何在不破壞原始 framework 行為的前提下注入 tracing？
- [ ] OpenTelemetry 的 span/trace 模型如何對映到 MLflow 的 entity 模型？
- [ ] Gateway 模組的 provider registry 如何支援 pluggable LLM provider？
- [ ] 測試策略：1,015 個測試檔案如何組織？integration test 怎麼 mock external services？
