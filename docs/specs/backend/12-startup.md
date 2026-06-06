# 12 — 啟動與優雅關閉

## 為什麼必要

部署滾動更新時 k8s / load balancer 會送 `SIGTERM`，若 server 直接 `os.Exit` 會中斷 in-flight 請求、連線池與快取連線未正確收尾。優雅關閉確保：

1. 不再接收新請求（從 LB 摘掉）
2. 等待現有請求完成（在 `ShutdownTimeout` 內）
3. 關閉 DB、Redis 連線

---

## main.go 完整骨架

```go
// cmd/server/main.go
func main() {
    cfg, err := config.LoadConfig(os.Getenv("ENV"))
    if err != nil {
        slog.Error("config load failed", "err", err)
        os.Exit(1)
    }

    logger.Init(cfg.App.Env, cfg.App.LogLevel)
    validator.Init()

    pool, err := database.New(cfg.DB)
    if err != nil {
        slog.Error("db init failed", "err", err)
        os.Exit(1)
    }
    defer pool.Close()
    txMgr := database.NewTxManager(pool)

    rdb, err := cache.New(cfg.Redis.URL)
    if err != nil {
        slog.Error("redis init failed", "err", err)
        os.Exit(1)
    }
    defer rdb.Close()

    // DI 組裝
    userRepo      := repository.NewUserRepo(pool)
    userService   := service.NewUserService(userRepo, txMgr)
    userHandler   := handlers.NewUserHandler(userService)
    healthHandler := handlers.NewHealthHandler(pool, cache.RedisPinger{Client: rdb})

    // 路由
    r := chi.NewRouter()
    r.Use(middleware.RequestID, middleware.Logger,
          middleware.CORS(cfg.App.AllowedOrigin), middleware.Recoverer)
    httpx.Get(r, "/healthz", healthHandler.Live)
    httpx.Get(r, "/readyz",  healthHandler.Ready)
    r.Group(func(r chi.Router) {
        r.Use(jwtMw.Verify, middleware.Timeout(10*time.Second))
        httpx.Get(r, "/api/users/{id}", userHandler.GetUser)
        httpx.Put(r, "/api/users/{id}", userHandler.UpdateUser)
    })

    srv := &http.Server{
        Addr:              fmt.Sprintf(":%d", cfg.App.Port),
        Handler:           r,
        ReadHeaderTimeout: 5 * time.Second,
    }

    // 1) 在 goroutine 啟動 server
    serverErr := make(chan error, 1)
    go func() {
        slog.Info("server started", "port", cfg.App.Port, "env", cfg.App.Env)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            serverErr <- err
        }
    }()

    // 2) 等待 SIGTERM / SIGINT 或 server 自爆
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    select {
    case sig := <-quit:
        slog.Info("shutdown signal received", "signal", sig.String())
    case err := <-serverErr:
        slog.Error("server crashed", "err", err)
        os.Exit(1)
    }

    // 3) 優雅關閉：在 ShutdownTimeout 內等 in-flight 請求結束
    ctx, cancel := context.WithTimeout(context.Background(), cfg.App.ShutdownTimeout)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        slog.Error("server shutdown failed", "err", err)
    }
    slog.Info("server stopped")
}
```

---

## 關閉順序

```
SIGTERM
   │
   ▼
srv.Shutdown(ctx)          ← 停止接受新連線，等 in-flight 請求完成
   │
   ▼
defer rdb.Close()          ← Redis 連線關閉
   │
   ▼
defer pool.Close()         ← Postgres 連線池關閉
```

**注意**：使用 `defer` 順序（LIFO）確保 server 完全停止後再關 DB / Redis；若反向順序，仍在處理的請求會打到關閉的連線。

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

## 環境變數

| 變數 | 預設 | 說明 |
|------|------|------|
| `SHUTDOWN_TIMEOUT` | `15s` | 等待 in-flight 請求完成的最長時間；超過則強制中斷 |
