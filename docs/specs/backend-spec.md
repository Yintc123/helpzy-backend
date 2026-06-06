# Backend 基礎架構規格書

## 相關規格文件

- [API 規格](./api-spec.md) — 路由、端點 request/response
- [資料庫規格](./db-spec.md) — Schema、Migration、sqlc
- [JWT 驗證規格](./jwt-auth-spec.md) — Token 結構、Middleware

---

## 技術棧

| 項目 | 技術 | 說明 |
|------|------|------|
| 語言 | Go 1.23 | |
| HTTP Router | `chi` v5 | 輕量、stdlib 相容、Clean Architecture 友善 |
| 資料庫 | PostgreSQL 16 | 主要資料儲存 |
| DB 驅動 | `pgx` v5 | PostgreSQL 官方推薦驅動，支援連線池 |
| ORM | `sqlc` | SQL → 型別安全 Go 程式碼生成 |
| 快取 | Redis 7 | Session JWT 儲存、快取層 |
| Redis 客戶端 | `go-redis` v9 | |
| JWT | `golang-jwt/jwt` v5 | 驗證 BFF 傳入的 JWT |
| 容器化 | Docker + Docker Compose | |

---

## 架構：Clean Architecture

```
┌─────────────────────────────────────────┐
│           Frameworks & Drivers          │  ← chi、pgx、sqlc、go-redis、Docker
├─────────────────────────────────────────┤
│          Interface Adapters             │  ← handlers、repository 實作
├─────────────────────────────────────────┤
│              Use Cases                  │  ← services（商業邏輯）
├─────────────────────────────────────────┤
│               Entities                  │  ← models（純 Go struct）
└─────────────────────────────────────────┘
```

**原則：**
- 內層不依賴外層
- 每一層透過 interface 溝通
- 框架型別（chi、pgx、sqlc 生成碼）只出現在最外層
- `context.Context` 從 handler 貫穿至 repository

---

## 專案結構

```
backend/
├── cmd/
│   └── server/
│       └── main.go               # 進入點：組裝依賴、啟動伺服器
├── internal/
│   ├── handlers/                 # Interface Adapters：處理 HTTP 請求與回應
│   │   ├── user.go
│   │   └── user_test.go
│   ├── services/                 # Use Cases：商業邏輯
│   │   ├── user.go
│   │   └── user_test.go
│   ├── repository/               # Interface Adapters：包裝 sqlc，實作 service 所需的 interface
│   │   ├── user.go
│   │   └── user_test.go
│   └── models/                   # Entities：純業務用資料結構（與 sqlc 生成的 struct 分離）
│       └── user.go
├── db/
│   ├── migrations/               # SQL Migration 檔案
│   │   ├── 000001_create_users.up.sql
│   │   └── 000001_create_users.down.sql
│   ├── queries/                  # sqlc 來源：SQL query 定義檔
│   │   └── user.sql
│   └── sqlc/                     # sqlc 自動生成（禁止手動修改）
│       ├── db.go
│       ├── models.go
│       └── user.sql.go
├── pkg/
│   ├── middleware/               # HTTP Middleware
│   │   ├── jwt.go
│   │   ├── cors.go
│   │   ├── logger.go
│   │   └── timeout.go
│   ├── auth/
│   │   └── jwt.go               # JWT 解析與驗證（純函式）
│   ├── database/
│   │   └── postgres.go          # PostgreSQL 連線池初始化
│   ├── cache/
│   │   └── redis.go             # Redis 連線初始化
│   ├── apperror/
│   │   └── error.go             # 統一錯誤型別
│   └── response/
│       └── response.go          # 統一回應格式
├── sqlc.yaml                     # sqlc 設定檔
├── configs/
│   └── .env.example
├── docs/
├── Dockerfile
└── go.mod
```

---

## API 路由

> 詳見 [API 規格書](./api-spec.md)。

---

## Middleware 規格

### 套用順序

```
Request
  │
  ▼
Logger          ← 記錄所有請求
  │
  ▼
CORS            ← 設定跨域 Header
  │
  ▼
Recoverer       ← 捕捉 panic，回傳 500
  │
  ▼
JWT（群組內）   ← 驗證 JWT，注入 userId / role 至 context
  │
  ▼
Timeout（群組內）← 設定請求最大執行時間
  │
  ▼
Handler
```

### JWT Middleware

```go
func (m *JWTMiddleware) Verify(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := extractBearerToken(r)
        claims, err := auth.ParseJWT(token, m.secret)
        if err != nil {
            response.Error(w, http.StatusUnauthorized, err)
            return
        }
        ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.Sub)
        ctx  = context.WithValue(ctx, ContextKeyRole, claims.Role)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### Timeout Middleware

```go
func Timeout(duration time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), duration)
            defer cancel()
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

---

## Context 規格

### Context Keys

```go
type contextKey string

const (
    ContextKeyUserID contextKey = "userId"
    ContextKeyRole   contextKey = "role"
)
```

### Handler 取用方式

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    userID := r.Context().Value(ContextKeyUserID).(string)
    id     := chi.URLParam(r, "id")
    // ...
}
```

### Context 傳遞規則

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

## 分層規格

### Interface 定義

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

### Dependency Injection（在 main.go 組裝）

```go
// cmd/server/main.go
db    := database.New(cfg.DatabaseURL)
cache := cache.New(cfg.RedisURL)

userRepo    := repository.NewUserRepo(db)
userService := service.NewUserService(userRepo)
userHandler := handlers.NewUserHandler(userService)
```

---

## 統一回應格式

### 成功

```json
{
  "data": { ... }
}
```

### 錯誤

```json
{
  "error": "描述訊息"
}
```

### response 套件

```go
// pkg/response/response.go
func JSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]any{"data": data})
}

func Error(w http.ResponseWriter, status int, err error) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(map[string]any{"error": err.Error()})
}
```

---

## 統一錯誤處理

### AppError 型別

```go
// pkg/apperror/error.go
type AppError struct {
    Code    int
    Message string
}

func (e *AppError) Error() string { return e.Message }

var (
    ErrNotFound     = &AppError{Code: 404, Message: "not found"}
    ErrUnauthorized = &AppError{Code: 401, Message: "unauthorized"}
    ErrBadRequest   = &AppError{Code: 400, Message: "bad request"}
    ErrInternal     = &AppError{Code: 500, Message: "internal server error"}
)
```

### Handler 錯誤處理

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    user, err := h.svc.GetUser(r.Context(), chi.URLParam(r, "id"))
    if err != nil {
        var appErr *apperror.AppError
        if errors.As(err, &appErr) {
            response.Error(w, appErr.Code, appErr)
            return
        }
        response.Error(w, http.StatusInternalServerError, apperror.ErrInternal)
        return
    }
    response.JSON(w, http.StatusOK, user)
}
```

---

## sqlc

> 詳見 [資料庫規格書](./db-spec.md)。

---

## PostgreSQL 規格

### 連線池設定

```go
// pkg/database/postgres.go
type Config struct {
    Host     string
    Port     string
    User     string
    Password string
    DBName   string
}

func New(c Config) *pgxpool.Pool {
    url := fmt.Sprintf(
        "postgres://%s:%s@%s:%s/%s?sslmode=disable",
        c.User, c.Password, c.Host, c.Port, c.DBName,
    )
    cfg, _ := pgxpool.ParseConfig(url)
    cfg.MaxConns = 20
    cfg.MinConns = 5
    cfg.MaxConnLifetime = 1 * time.Hour
    cfg.MaxConnIdleTime = 30 * time.Minute
    pool, _ := pgxpool.NewWithConfig(context.Background(), cfg)
    return pool
}
```

### 環境變數

```
DB_HOST=localhost
DB_PORT=5432
DB_USER=user
DB_PASSWORD=password
DB_NAME=helpzy
```

> Migration 規範詳見 [資料庫規格書](./db-spec.md)。

---

## Redis 規格

### 連線設定

```go
// pkg/cache/redis.go
func New(url string) *redis.Client {
    opt, _ := redis.ParseURL(url)
    return redis.NewClient(opt)
}
```

### 環境變數

```
REDIS_URL=redis://localhost:6379
```

---

## 環境變數

| 變數 | 說明 |
|------|------|
| `PORT` | 伺服器埠號（預設 `8080`） |
| `ENV` | 執行環境（`development` / `production`） |
| `DB_HOST` | PostgreSQL host |
| `DB_PORT` | PostgreSQL port（預設 `5432`） |
| `DB_USER` | PostgreSQL 使用者 |
| `DB_PASSWORD` | PostgreSQL 密碼 |
| `DB_NAME` | PostgreSQL 資料庫名稱 |
| `REDIS_URL` | Redis 連線字串 |
| `JWT_SECRET` | JWT 簽署金鑰（與 BFF 共享，min 32 chars） |
| `ALLOWED_ORIGIN` | 允許的 CORS Origin（BFF 位址） |

---

## Docker

### Dockerfile（Multi-stage Build）

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

FROM alpine:3.20
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

### Docker Compose 服務

```
helpzy-backend  ← Go server（port 8080）
postgres        ← PostgreSQL 16（port 5432）
redis           ← Redis 7（port 6379）
```

---

## TDD 測試策略

### 各層測試方式

| 層 | 測試方式 | 依賴 |
|----|---------|------|
| `pkg/auth` | 單元測試，純函式 | 無 |
| `pkg/middleware` | `httptest` 模擬 HTTP | 無 |
| `internal/handlers` | `httptest` + mock service | mock |
| `internal/services` | 單元測試 + mock repository | mock |
| `internal/repository` | 整合測試，連接真實測試 DB | PostgreSQL（testcontainers） |
| `db/queries` | `sqlc vet` 驗證 SQL 語法 | 無 |

### Mock 方式

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

### 測試指令

```bash
go test ./...                    # 所有測試
go test ./internal/handlers/...  # 只跑 handlers
go test ./... -cover             # 覆蓋率
go test ./internal/repository/... -tags integration  # 整合測試
```

---

## 套件清單

```go
// go.mod 預計引入
require (
    github.com/go-chi/chi/v5          v5.x
    github.com/jackc/pgx/v5           v5.x
    github.com/redis/go-redis/v9      v9.x
    github.com/golang-jwt/jwt/v5      v5.x
    github.com/stretchr/testify        v1.x   // assert
    github.com/testcontainers/testcontainers-go v0.x  // 整合測試
)
```

> 開發工具（sqlc、golang-migrate）安裝詳見 [資料庫規格書](./db-spec.md)。
