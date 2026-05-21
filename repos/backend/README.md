# backend/ — Backend 應用

**上層目錄**: [repos/](../README.md)

Web API 應用、微服務、後端服務型系統。注意:這裡放**應用**,不是框架本體。
學習重點在於「如何用框架組建一個可維護、可擴展的後端服務」的架構設計。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **架構分層** | 是 MVC、Clean Architecture 還是 DDD?分層邊界在哪?依賴方向? |
| **API 設計** | 路由組織方式?版本管理策略?request / response schema 設計? |
| **資料層** | ORM 選擇?migration 工具?是否有 repository pattern?transaction 管理? |
| **中介層** | auth 機制(JWT / session / OAuth)?logging 格式?rate limiting?error handling 統一化? |
| **非同步處理** | background task / queue 用什麼?cron job 怎麼跑?event-driven 設計? |
| **可觀測性** | metrics(Prometheus / OpenTelemetry)?distributed tracing?structured logging? |
| **測試策略** | unit / integration / e2e 比例?DB 測試怎麼 mock?fixture 設計? |

套用模板:`_templates/backend.md`

---

## 典型專案

| 專案 | 技術棧 | 特色 |
|------|--------|------|
| FastAPI + SQLModel 應用 | Python / FastAPI | async-first、Pydantic v2、clean 分層 |
| Django REST API | Python / Django | batteries-included、ORM mature、大型應用 |
| NestJS 微服務 | TypeScript / NestJS | DI container、decorator、module 化 |
| Go 微服務 | Go / Gin / GORM | 高效能、簡潔、service mesh 整合 |
| Rust Axum 應用 | Rust / Axum | 型別安全、zero-cost abstraction、async |

> 這個類別收錄**具體的應用型 repo**,而非框架本體。框架本體見 [`backend-framework`](../backend-framework/)。

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 用框架建的 API 應用、業務系統 | ✅ `backend` | — |
| 框架本體(FastAPI / Django / Gin) | — | ✅ [`backend-framework`](../backend-framework/) |
| AI 後端(以 LLM serving 為主) | — | ✅ [`llm-serving`](../llm-serving/) |
| 資料庫引擎本體 | — | ✅ [`infra`](../infra/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
