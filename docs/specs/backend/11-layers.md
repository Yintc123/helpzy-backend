# 11 — 分層規格與 Context 規格

## Context 傳遞規則

- Handler → Service → Repository 全程傳遞 `ctx context.Context`
- Repository 所有 DB 查詢都傳入 `ctx`，確保 timeout / cancel 能向下傳播
- 禁止在任何層儲存或複製 context

```go
// repository 層範例
func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    row := r.db.QueryRow(ctx, "SELECT id, email FROM users WHERE id = $1", id)
    // ctx 傳入 DB，client 斷線或 timeout 自動取消查詢
}
```

---

## Interface 定義

每層透過 interface 溝通，實作在各自的檔案：

```go
// internal/services/user.go
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*models.User, error)
    Update(ctx context.Context, user *models.User) error
}

type UserService struct {
    repo UserRepository
}

// internal/handlers/user.go
type UserServiceInterface interface {
    GetUser(ctx context.Context, id string) (*models.User, error)
    UpdateUser(ctx context.Context, id string, input UpdateUserInput) (*models.User, error)
}

type UserHandler struct {
    svc UserServiceInterface
}
```

---

## Dependency Injection（在 main.go 組裝）

依賴一律由 `main.go` 由外而內手動組裝（不使用 wire / fx 等框架），順序：

```
config → logger → validator → pool / redis → txMgr → repo → service → handler → router
```

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

## 各層測試方式

| 層 | 測試方式 | 依賴 |
|----|---------|------|
| `internal/handlers` | `httptest` + mock service | mock |
| `internal/services` | 單元測試 + mock repository / mock TxManager | mock |
| `internal/repository` | 整合測試，連接真實測試 DB | PostgreSQL（testcontainers） |
