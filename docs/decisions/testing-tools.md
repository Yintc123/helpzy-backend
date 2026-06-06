# 測試工具決策：testing + testify + testcontainers-go

**決定**：`testing`（stdlib）+ `testify/assert` + `testcontainers-go`，不使用 mock 框架。

## 工具組合

| 工具 | 用途 | 理由 |
|------|------|------|
| `testing` | 測試框架 | Go 內建，無需依賴 |
| `net/http/httptest` | HTTP 測試 | Go 內建，完整 |
| `testify/assert` | Assert | 減少樣板，最廣泛使用 |
| `testcontainers-go` | DB 整合測試 | 自動管理 PostgreSQL container，不污染本地環境 |
| 手動 mock struct | Mock | 保持簡單，符合 Clean Architecture |

## 不使用 mock 框架的理由

手動定義 mock struct 更透明，不引入額外依賴，與 Clean Architecture 的 interface 設計自然搭配。

## 各層測試方式

| 層 | 測試方式 |
|----|---------|
| `pkg/auth` | 單元測試，純函式 |
| `pkg/middleware` | `httptest` 模擬 HTTP |
| `internal/handlers` | `httptest` + mock service |
| `internal/services` | 單元測試 + mock repository |
| `internal/repository` | 整合測試（testcontainers） |
