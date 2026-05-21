# backend-framework/ — 後端框架本體

**上層目錄**: [repos/](../README.md)

後端框架本體、ORM、API gateway、middleware 框架。注意:這裡放**框架**,不是用框架建的應用。
學習重點在於「框架如何設計公開 API、如何讓使用者擴充、如何維持向後相容」。

---

## 學習聚焦面向

| 面向 | 核心問題 |
|------|----------|
| **API 設計哲學** | convention over configuration 還是顯式配置?magic 程度?decorator vs class-based? |
| **Request Lifecycle** | 一個 HTTP 請求從進入到回應,經過幾個層?每層的職責? |
| **Middleware / Plugin** | middleware 怎麼註冊?執行順序怎麼決定?context 怎麼傳遞? |
| **Dependency Injection** | DI 機制設計?scope(request / singleton / transient)?circular dependency 處理? |
| **ORM 設計(若適用)** | 查詢 API 設計?lazy vs eager loading?unit of work 模式?migration 工具? |
| **擴充點** | 哪些是 public contract?哪些是 internal?extension point 怎麼設計? |
| **相容性策略** | breaking change 策略?deprecation 流程?semver 遵循嚴格程度? |

套用模板:`_templates/backend.md`

---

## 典型專案

| 專案 | 語言 | 特色 |
|------|------|------|
| FastAPI | Python | Pydantic 型別整合、OpenAPI 自動生成、async-first |
| SQLAlchemy | Python | Unit of Work、Identity Map、Core + ORM 雙層 API |
| Django REST Framework | Python | serializer、viewset、permissions、browsable API |
| Gin | Go | 極簡 router、middleware chain、高吞吐 |
| Axum | Rust | tower middleware、extractors、型別安全 handler |
| Hono | TypeScript | edge-first、Web Standard API、多 runtime |

---

## 跟相鄰類別的邊界

| 情況 | 歸這裡 | 歸其他 |
|------|--------|--------|
| 框架本體、ORM 本體 | ✅ `backend-framework` | — |
| 用框架建的應用、服務 | — | ✅ [`backend`](../backend/) |
| 通用 HTTP client 函式庫 | — | ✅ [`library`](../library/) |
| API gateway / service mesh 基礎設施 | — | ✅ [`infra`](../infra/) |

---

## 已收錄的專案

<!-- 由 Hermes Agent 在每次 PR 合併後自動更新 -->

| 專案 | 學習日期 | 深度 | 亮點 |
|------|----------|------|------|
| — | — | — | 尚無紀錄 |
