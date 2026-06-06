# 13 — TDD 測試策略

## 各層測試方式

| 層 | 測試方式 | 依賴 |
|----|---------|------|
| `pkg/auth` | 單元測試，純函式 | 無 |
| `internal/config` | `t.Setenv` 模擬環境，覆蓋 required / default / 邊界 helper | 無 |
| `pkg/logger` | 單元測試（buffer handler 斷言 JSON 欄位、含 OTel trace span 補欄位） | 無 |
| `pkg/validator` | 單元測試（table-driven 涵蓋自訂規則） | 無 |
| `pkg/apperror` | 單元測試（`errors.Is` / `Unwrap` / `Wrap` 行為） | 無 |
| `pkg/httpx` | `httptest` 驗證錯誤 → HTTP 回應映射（含 `request_id` 欄位） | 無 |
| `pkg/database/tx` | 整合測試（rollback / commit / 巢狀 / fallback） | PostgreSQL（testcontainers） |
| `pkg/middleware` | `httptest` 模擬 HTTP | 無 |
| `pkg/middleware/ratelimit` | 整合測試（允許 → 拒絕 → 重置；Redis 故障 fail-open） | Redis（testcontainers） |
| `pkg/middleware/secureheaders` | `httptest` 斷言 response headers；HSTS production-only | 無 |
| `pkg/observability` | OTel `tracetest.InMemoryExporter` 斷言 span；metric testkit 斷言 counter | 無 |
| `internal/handlers/health` | mock `Pinger` interface 斷言 200/500 | mock |
| `internal/handlers` | `httptest` + mock service | mock |
| `internal/services` | 單元測試 + mock repository / mock TxManager | mock |
| `internal/repository` | 整合測試，連接真實測試 DB | PostgreSQL（testcontainers） |
| `internal/repository/refresh` | 整合測試（Rotate 原子性 + reuse detection） | Redis（testcontainers） |
| `internal/repository/revoke` | 整合測試（RevokeFamily / IsFamilyRevoked / TTL） | Redis（testcontainers） |
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

---

## 整合測試 Scaffold（testcontainers）

`pkg/database/tx_test.go`、`internal/repository/*_test.go`、`pkg/middleware/jwt_test.go`（blacklist 部分）等都需要真實 Postgres / Redis。共用 helper 集中於 `internal/testutil/`，避免每個整合測試各自寫 testcontainers 啟動樣板。

### 設計原則

| 原則 | 理由 |
|------|------|
| **Process-wide singleton container** | 整個 `go test` 共用一個 Postgres / Redis container；首次呼叫啟動（~3 秒），後續複用近乎零延遲 |
| **Per-test truncate** | `t.Cleanup` 清空 user tables（Postgres）/ `FLUSHDB`（Redis）→ 測試間隔離 |
| **首次啟動時套用 migration** | 用 `golang-migrate` 一次套用 `db/migrations/`，repository 測試直接面對 production schema |
| **`//go:build integration` 隔離** | 純單元測試（`go test ./...`）不會觸發 container 啟動 |
| **基礎建設錯誤 panic** | container 啟動 / migration 失敗屬環境問題，panic 讓 `go test` 直接終止比 `t.Fatal` 更醒目 |
| **自動定位專案根目錄** | 走訪父目錄找 `go.mod`，不依賴 `testutil` 檔案位置或測試執行目錄 |

### `internal/testutil/postgres.go`

```go
//go:build integration

package testutil

import (
    "context"
    "fmt"
    "os"
    "path/filepath"
    "strings"
    "sync"
    "testing"
    "time"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

var (
    pgOnce sync.Once
    pgPool *pgxpool.Pool
)

// NewPostgres 回傳指向共享測試 container 的連線池。
// 首次呼叫啟動 container 並套用 migration；後續呼叫直接複用。
// 每個 test 結束時自動 truncate 所有 user tables。
func NewPostgres(t *testing.T) *pgxpool.Pool {
    t.Helper()
    pgOnce.Do(startPostgres)
    t.Cleanup(func() { truncateAll(t, pgPool) })
    return pgPool
}

func startPostgres() {
    ctx := context.Background()
    container, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("helpzy_test"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        panic(fmt.Errorf("start postgres: %w", err))
    }

    dsn, err := container.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        panic(fmt.Errorf("get dsn: %w", err))
    }

    migrationsURL := "file://" + filepath.Join(findProjectRoot(), "db", "migrations")
    m, err := migrate.New(migrationsURL, dsn)
    if err != nil {
        panic(fmt.Errorf("init migrate: %w", err))
    }
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        panic(fmt.Errorf("apply migrations: %w", err))
    }

    pool, err := pgxpool.New(ctx, dsn)
    if err != nil {
        panic(fmt.Errorf("create pool: %w", err))
    }
    pgPool = pool
}

// truncateAll 清空所有 user table（不含 schema_migrations）。
// 用 TRUNCATE ... CASCADE 一次清掉，避免 FK 順序問題。
func truncateAll(t *testing.T, pool *pgxpool.Pool) {
    t.Helper()
    ctx := context.Background()
    rows, err := pool.Query(ctx, `
        SELECT tablename FROM pg_tables
        WHERE schemaname = 'public' AND tablename != 'schema_migrations'
    `)
    if err != nil {
        t.Fatalf("list tables: %v", err)
    }
    var tables []string
    for rows.Next() {
        var name string
        if err := rows.Scan(&name); err != nil {
            rows.Close()
            t.Fatalf("scan: %v", err)
        }
        tables = append(tables, name)
    }
    rows.Close()
    if len(tables) == 0 {
        return
    }
    if _, err := pool.Exec(ctx,
        fmt.Sprintf("TRUNCATE %s RESTART IDENTITY CASCADE",
            strings.Join(tables, ", "))); err != nil {
        t.Fatalf("truncate: %v", err)
    }
}

// findProjectRoot 從當前 working directory 往上找 go.mod，
// 讓 migration path 不依賴 test 的執行位置。
func findProjectRoot() string {
    dir, err := os.Getwd()
    if err != nil {
        panic(err)
    }
    for {
        if _, err := os.Stat(filepath.Join(dir, "go.mod")); err == nil {
            return dir
        }
        parent := filepath.Dir(dir)
        if parent == dir {
            panic("go.mod not found from " + dir)
        }
        dir = parent
    }
}
```

### `internal/testutil/redis.go`

```go
//go:build integration

package testutil

import (
    "context"
    "fmt"
    "sync"
    "testing"

    redislib "github.com/redis/go-redis/v9"
    tcredis "github.com/testcontainers/testcontainers-go/modules/redis"
)

var (
    redisOnce   sync.Once
    redisClient *redislib.Client
)

// NewRedis 回傳指向共享測試 container 的 Redis client。
// 首次呼叫啟動 container；後續複用。每個 test 結束自動 FLUSHDB。
func NewRedis(t *testing.T) *redislib.Client {
    t.Helper()
    redisOnce.Do(startRedis)
    t.Cleanup(func() {
        if err := redisClient.FlushDB(context.Background()).Err(); err != nil {
            t.Fatalf("flushdb: %v", err)
        }
    })
    return redisClient
}

func startRedis() {
    ctx := context.Background()
    container, err := tcredis.Run(ctx, "redis:7-alpine")
    if err != nil {
        panic(fmt.Errorf("start redis: %w", err))
    }
    endpoint, err := container.ConnectionString(ctx)
    if err != nil {
        panic(fmt.Errorf("get endpoint: %w", err))
    }
    opt, err := redislib.ParseURL(endpoint)
    if err != nil {
        panic(fmt.Errorf("parse url: %w", err))
    }
    redisClient = redislib.NewClient(opt)
}
```

### 使用範例

```go
//go:build integration

package repository_test

import (
    "context"
    "testing"

    "helpzy/internal/repository"
    "helpzy/internal/testutil"
)

func TestUserRepo_FindByID_WhenExists_ReturnsUser(t *testing.T) {
    pool := testutil.NewPostgres(t) // 取得共享 pool；test 結束時自動 truncate
    repo := repository.NewUserRepo(pool)

    // ... 寫入測試資料、呼叫 repo、斷言
    _ = context.Background()
}
```

### 執行

```bash
go test ./...                          # 純單元測試（不啟 container，快）
go test ./... -tags integration        # 含整合測試（啟 container，較慢但完整）
```

### 限制與權衡

- 因為 container 是 process-wide singleton + truncate，**同 package 內的整合測試不能 `t.Parallel()`**——會撞 truncate。若日後需要平行化，再改為「每 test 自己的 schema」模式即可。
- 首次呼叫 `NewPostgres` / `NewRedis` 約需 3 秒啟動 container；後續複用無延遲。`sync.Once` 確保多 test 並發呼叫只啟動一次。
- 若 `startPostgres` panic，`sync.Once` 不會重試；後續 test 會 nil-deref 連鎖失敗——這是刻意的，避免遮蔽環境問題。
