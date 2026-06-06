# 13 — TDD 測試策略

## 各層測試方式

| 層 | 測試方式 | 依賴 |
|----|---------|------|
| `pkg/auth` | 單元測試，純函式 | 無 |
| `internal/config` | `t.Setenv` 模擬環境，覆蓋 required / default / 邊界 helper | 無 |
| `pkg/logger` | 單元測試（buffer handler 斷言 JSON 欄位） | 無 |
| `pkg/validator` | 單元測試（table-driven 涵蓋自訂規則） | 無 |
| `pkg/apperror` | 單元測試（`errors.Is` / `Unwrap` / `Wrap` 行為） | 無 |
| `pkg/httpx` | `httptest` 驗證錯誤 → HTTP 回應映射 | 無 |
| `pkg/database/tx` | 整合測試（rollback / commit / 巢狀 / fallback） | PostgreSQL（testcontainers） |
| `pkg/middleware` | `httptest` 模擬 HTTP | 無 |
| `internal/handlers/health` | mock `Pinger` interface 斷言 200/500 | mock |
| `internal/handlers` | `httptest` + mock service | mock |
| `internal/services` | 單元測試 + mock repository / mock TxManager | mock |
| `internal/repository` | 整合測試，連接真實測試 DB | PostgreSQL（testcontainers） |
| `db/queries` | `sqlc vet` 驗證 SQL 語法 | 無 |

---

## Mock 方式

手動定義 mock struct，不使用 mock 框架：

```go
// 測試用 mock
type mockUserRepo struct {
    findByIDFn func(ctx context.Context, id string) (*models.User, error)
}

func (m *mockUserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    return m.findByIDFn(ctx, id)
}
```

---

## 測試指令

```bash
go test ./...                    # 所有測試
go test ./internal/handlers/...  # 只跑 handlers
go test ./... -cover             # 覆蓋率
go test ./internal/repository/... -tags integration  # 整合測試
```

---

## 命名規範

```go
func TestFunctionName_條件_預期結果(t *testing.T) { ... }

// 範例
func TestHealthCheck_WhenCalled_ReturnsStatusOK(t *testing.T) { ... }
```
