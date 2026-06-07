# 18 — Foundation 階段開發路線與最小 main.go

## 目的

`12-startup.md` 的 `main.go` 是**最終形態**——同時 wire foundation 與政策層（auth service / dispatcher / handlers）。在 foundation 階段尚未實作政策層之前，**該 main.go 無法 compile**，會阻塞 TDD 紅綠循環。

本文件提供：

1. **實作路線**：foundation 17 份規格的展開順序與相依關係
2. **Phase 1 main.go 骨架**：僅依賴 foundation 模組，可獨立 compile / 啟動 / 通過 smoke test
3. **驗收清單**：每個 stage 結束時應該 green 的測試集合

> 政策層（auth / chat / users CRUD 等）整合進 main.go 的時機與順序見 [12-startup.md](./12-startup.md)。

---

## 實作路線（Stage 1 → Stage 6）

依「無外部依賴 → 有 testcontainers → 整合」三段排序，每個 stage 結束都有可執行的 `go test ./...` green 點。

### Stage 1 — 純單元層（無外部依賴）

| 模組 | 規格 | 產出測試 |
|------|------|---------|
| `pkg/apperror` | [04](./04-apperror.md) | `errors.Is` / `Unwrap` / `Wrap` 行為 |
| `pkg/logger` | [03](./03-logger.md) | `From` / `WithRequestID` / `WithUserID` / `parseLevel` |
| `pkg/validator` | [06](./06-validator.md) | `Struct` / `passwordRule` / 自訂規則 |
| `internal/config` | [02](./02-config.md) | `LoadConfig` / `Validate` / `String` 遮蔽 / helpers |
| `pkg/response` + `pkg/httpx` | [05](./05-httpx.md) | `httpx.Func` / `WriteError` / `DecodeJSON` |
| `pkg/auth` | [15](./15-auth-mechanism.md) | `SignAccess` / `ParseAccess` / `HashPassword` / `GenerateOpaqueToken` / `SignedID` |

> Stage 1 完成後可獨立執行 `go test ./pkg/... ./internal/config/...`。**不需要任何 container**。

### Stage 2 — 連線層（純函式部分 + Pinger）

| 模組 | 規格 | 產出測試 |
|------|------|---------|
| `pkg/database` (`buildDSN`) | [07](./07-database.md) | percent-encoding round-trip |
| `pkg/cache` (`New` / `RedisPinger`) | [08](./08-redis.md) | URL 解析錯誤路徑 |
| `pkg/middleware/context.go` | [09](./09-middleware.md#context-keys) | `With*` / `*From` 雙向 |
| `pkg/middleware`（非整合部分） | [09](./09-middleware.md) | `RequestID` / `Logger` / `CORS` / `SecureHeaders` / `Recoverer` / `BodyLimit` / `Timeout` / `extractBearerToken` |

> 這個 stage 的 middleware 不掛 Redis / DB 的依賴項（rate limit、JWT 撤銷檢查留到 Stage 3）。

### Stage 3 — 整合層（testcontainers Postgres / Redis）

| 模組 | 規格 | 產出測試 |
|------|------|---------|
| `internal/testutil` | [13](./13-testing.md) | scaffold 本身 |
| `pkg/database/tx.go` | [07](./07-database.md) | rollback / commit / 巢狀 / pool fallback |
| `pkg/authstore/{refresh,revoke,ticket}` | [15](./15-auth-mechanism.md) | 原子 rotation、reuse detection、family revoke、GETDEL |
| `pkg/middleware/ratelimit.go` | [09](./09-middleware.md#ratelimit-middleware) | 允許 → 拒絕 → 重置；fail-open |
| `pkg/middleware/jwt.go` | [15](./15-auth-mechanism.md#pkgmiddlewarejwtgo) | 完整 `Verify` |

> 從這裡開始 `go test ./... -tags integration` 會啟 container。CI 上應分兩段（單元 / 整合）報告。

### Stage 4 — 最小 schema + 第一個 repo

| 模組 | 規格 | 產出測試 |
|------|------|---------|
| `db/migrations/000001_create_users.{up,down}.sql` | [features/auth-feature-spec.md §1](../features/auth-feature-spec.md) | migration up / down round-trip |
| `db/queries/user.sql` + `db/sqlc/` 生成 | 同上 | `sqlc vet` |
| `internal/repository/user.go` | [14](./14-sqlc-and-migration.md) | `FindByID` / `FindByEmail` / `Create` 整合測試 |
| `internal/handlers/health.go` | [10](./10-health.md) | `Live` / `Ready` |

> Stage 4 結束時，**Phase 1 main.go**（見下）可實際啟動並通過 `/healthz` / `/readyz` smoke test。

### Stage 5 — 可觀測性

| 模組 | 規格 | 產出測試 |
|------|------|---------|
| `pkg/observability/otel.go` | [16](./16-observability.md) | `Init` 成功 / `Enabled=false` 為 noop |
| `pkg/observability/metrics.go` | [16](./16-observability.md#業務指標範例) | counter 增量（`metricdata` testkit） |
| 把 `otelpgx` / `redisotel` 接入 `pkg/database` / `pkg/cache` | [16](./16-observability.md#自動化儀器instrumentation) | 整合測試斷言 span |

### Stage 6 — 接政策層

`12-startup.md` 的完整 main.go 出場——`authservice` / `dispatcher` / `userHandler` / `wsHandler` 由 feature spec 規範，**已超出 foundation 範圍**。

---

## Phase 1 main.go（foundation-only）

下列骨架**只依賴 foundation 模組**，可在 Stage 4 結束時直接 compile + run。功能限制：只提供 `/healthz`、`/readyz` 與一個 echo-only `/api/v1/ping`（驗證 middleware 串通），所有業務路由留待 Phase 2。

> 此檔案位於 `cmd/server/main.go`，**在 Stage 6 時將被 [12-startup.md](./12-startup.md) 的完整版本替換**。中間階段保留此版本不會被誤刪——`12-startup.md` 是擴充而非分支。

```go
// cmd/server/main.go（Phase 1 — foundation only）
package main

import (
    "context"
    "errors"
    "fmt"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"

    "github.com/go-chi/chi/v5"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

    "helpzy/internal/config"
    "helpzy/internal/handlers"
    "helpzy/pkg/cache"
    "helpzy/pkg/database"
    "helpzy/pkg/httpx"
    "helpzy/pkg/logger"
    "helpzy/pkg/middleware"
    "helpzy/pkg/observability"
    "helpzy/pkg/response"
    "helpzy/pkg/validator"
)

func main() {
    cfg, err := config.LoadConfig(os.Getenv("ENV"))
    if err != nil {
        slog.Error("config load failed", "err", err)
        os.Exit(1)
    }

    logger.Init(cfg.App.Env, cfg.App.LogLevel)
    validator.Init()

    otelShutdown, err := observability.Init(context.Background(), observability.Config{
        Enabled:        cfg.OTel.Enabled,
        Endpoint:       cfg.OTel.Endpoint,
        ServiceName:    cfg.OTel.ServiceName,
        ServiceVersion: cfg.OTel.ServiceVersion,
        Env:            cfg.App.Env,
        SampleRatio:    cfg.OTel.SampleRatio,
    })
    if err != nil {
        slog.Error("otel init failed", "err", err)
        os.Exit(1)
    }

    pool, err := database.New(cfg.DB)
    if err != nil {
        slog.Error("db init failed", "err", err)
        os.Exit(1)
    }
    defer pool.Close()

    rdb, err := cache.New(cfg.Redis.URL)
    if err != nil {
        slog.Error("redis init failed", "err", err)
        os.Exit(1)
    }
    defer rdb.Close()

    healthHandler := handlers.NewHealthHandler(
        pool,
        cache.RedisPinger{Client: rdb},
        cfg.App.HealthPingTimeout,
    )

    // ── 路由：僅 health + ping（無業務 handler） ───────────
    r := chi.NewRouter()
    r.Use(
        middleware.RequestID,
        middleware.Logger,
        middleware.Recoverer,
        middleware.SecureHeaders(middleware.SecureHeadersConfig{
            HSTSEnabled: cfg.App.Env == "production",
            HSTSMaxAge:  cfg.App.HSTSMaxAge,
        }),
        middleware.CORS(cfg.App.AllowedOrigins, cfg.App.CORSMaxAge),
        middleware.BodyLimit(cfg.App.MaxBodyBytes),
    )
    httpx.Get(r, "/healthz", healthHandler.Live)
    httpx.Get(r, "/readyz",  healthHandler.Ready)

    // Smoke test 用：確認 middleware 串通，含 ctx / logger / 錯誤回應
    httpx.Get(r, "/api/v1/ping", func(w http.ResponseWriter, req *http.Request) error {
        logger.From(req.Context()).Info("ping")
        return response.JSON(w, http.StatusOK, map[string]string{"pong": "ok"})
    })

    rootHandler := otelhttp.NewHandler(r, "http.server",
        otelhttp.WithFilter(func(req *http.Request) bool {
            return req.URL.Path != "/healthz" && req.URL.Path != "/readyz"
        }),
    )

    srv := &http.Server{
        Addr:              fmt.Sprintf(":%d", cfg.App.Port),
        Handler:           rootHandler,
        ReadHeaderTimeout: cfg.App.ReadHeaderTimeout,
        ReadTimeout:       cfg.App.ReadTimeout,
        WriteTimeout:      cfg.App.WriteTimeout,
        IdleTimeout:       cfg.App.IdleTimeout,
    }

    serverErr := make(chan error, 1)
    go func() {
        slog.Info("server started", "port", cfg.App.Port, "env", cfg.App.Env, "phase", "foundation-only")
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            serverErr <- err
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    select {
    case sig := <-quit:
        slog.Info("shutdown signal received", "signal", sig.String())
    case err := <-serverErr:
        slog.Error("server crashed", "err", err)
        os.Exit(1)
    }

    ctx, cancel := context.WithTimeout(context.Background(), cfg.App.ShutdownTimeout)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        slog.Error("server shutdown failed", "err", err)
    }
    if err := otelShutdown(ctx); err != nil {
        slog.Error("otel shutdown failed", "err", err)
    }
    slog.Info("server stopped")
}
```

### 與 [12-startup.md](./12-startup.md) 完整版的差別

| 項目 | Phase 1 | Phase 2 / 完整版 |
|------|---------|------------------|
| `authstore.*` / `JWT middleware` / `RateLimiter` | 不 wire | 全部 wire |
| `userRepo` / `conversationRepo` | 不 wire（首版 main.go 沒 sqlc 依賴也能跑） | wire |
| `authService` / `dispatcher` / `llm.Registry` | 不 wire | wire |
| `userHandler` / `authHandler` / `wsHandler` | 不 wire | wire |
| 路由群組 | 只有 `/healthz` / `/readyz` / `/api/v1/ping` | `/api/v1/auth/*` / `/api/v1/anonymous/*` / `/api/v1/ws/*` / 受保護群組 |
| middleware 堆疊 | RequestID → Logger → Recoverer → SecureHeaders → CORS → BodyLimit | 同 + JWT + RateLimit + Timeout（依路由群組） |

> Phase 1 完全不引入 `pkg/authstore` / `internal/services` 套件路徑，可獨立 `go build ./cmd/server` 通過。

---

## Smoke test（Stage 4 結束時應通過）

啟動後手動驗證：

```bash
# 1. /healthz 應該無依賴永遠 200
curl -i http://localhost:8080/healthz
# Expect: 200 + {"data":{"status":"ok"}} + X-Request-ID

# 2. /readyz 應該檢查 DB / Redis
curl -i http://localhost:8080/readyz
# Expect: 200 + {"data":{"status":"ready"}}

# 3. /api/v1/ping 驗證 middleware 串通
curl -i http://localhost:8080/api/v1/ping
# Expect: 200 + {"data":{"pong":"ok"}} + X-Request-ID + 結構化 access log

# 4. Recoverer 驗證（暫時加一個 panic handler 後移除）
# 應收 500 + AppError shape，X-Request-ID 仍存在

# 5. BodyLimit
curl -X POST -H 'Content-Type: application/json' \
  --data-binary "@$(head -c 2M /dev/urandom | base64)" \
  http://localhost:8080/api/v1/ping
# Expect: 413 + kind=payload_too_large
```

容器化 smoke：

```bash
docker compose up -d postgres redis
ENV=local go run ./cmd/server
```

---

## 驗收清單（每個 stage 結束時 green）

### Stage 1 完成
- [ ] `go test ./pkg/apperror/... ./pkg/logger/... ./pkg/validator/... ./internal/config/... ./pkg/response/... ./pkg/httpx/... ./pkg/auth/...`
- [ ] 覆蓋率 ≥ 80%（純函式應接近 100%）

### Stage 2 完成
- [ ] `go test ./pkg/database/... ./pkg/cache/... ./pkg/middleware/...`
- [ ] `buildDSN` percent-encoding round-trip 通過

### Stage 3 完成
- [ ] `go test ./... -tags integration` 啟動 testcontainers 通過
- [ ] `authstore` 三件 + ratelimit + jwt middleware 全綠

### Stage 4 完成
- [ ] `migrate up` / `migrate down 1` / 再 `up` 連續成功
- [ ] `sqlc generate` 產生 `db/sqlc/user.sql.go`
- [ ] `internal/repository/user_test.go`（整合）綠
- [ ] **Phase 1 main.go 可啟動**，五項 smoke test 全通過

### Stage 5 完成
- [ ] OTel disabled 路徑無 panic
- [ ] OTel enabled + 本機 collector 能看到 server span、pgx span、redis span
- [ ] `auth.login.attempts` 等 counter 在 collector 出現

### Stage 6（已屬政策層）
- 見各 feature spec 的驗收清單

---

## 測試策略

本文件不引入新模組，**無對應測試**。Phase 1 main.go 的「能否啟動 + smoke test 是否通過」即為驗收手段（自動化可寫成 e2e bash 腳本放 CI nightly）。

---

## 與其它規格的關係

| 規格 | 關聯 |
|------|------|
| [12-startup.md](./12-startup.md) | 完整 main.go（Phase 2 + 政策層） |
| [13-testing.md](./13-testing.md) | testcontainers scaffold 在 Stage 3 引入 |
| [14-sqlc-and-migration.md](./14-sqlc-and-migration.md) | Stage 4 首次套用 migration + sqlc |
| [features/auth-feature-spec.md](../features/auth-feature-spec.md) | Stage 4 的 `000001_create_users` 由此 feature 規範 |
