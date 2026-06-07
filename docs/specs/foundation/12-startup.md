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

    // OTel：必須在 pool / rdb 建立前初始化，pgx / redis instrumentation 才能掛上
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

    pool, err := database.New(cfg.DB) // 內部已掛 otelpgx.NewTracer
    if err != nil {
        slog.Error("db init failed", "err", err)
        os.Exit(1)
    }
    defer pool.Close()
    txMgr := database.NewTxManager(pool)

    rdb, err := cache.New(cfg.Redis.URL) // 內部已掛 redisotel
    if err != nil {
        slog.Error("redis init failed", "err", err)
        os.Exit(1)
    }
    defer rdb.Close()

    // ── 機制層（foundation）：authstore + JWT middleware ────────────────
    refreshStore  := authstore.NewRefreshStore(rdb, cfg.JWT.RefreshTTL)
    familyRevoker := authstore.NewFamilyRevoker(rdb, cfg.JWT.RefreshTTL)
    ticketStore   := authstore.NewTicketStore(rdb, cfg.WSTicket.TTL)

    // ── 政策層：repos + services ─────────────────────────────────
    userRepo         := repository.NewUserRepo(pool)
    conversationRepo := repository.NewConversationRepo(pool) // 實作 ConversationLinker、ChatRepo 等多個 interface

    userService := service.NewUserService(userRepo, txMgr)
    authService := authservice.New(
        userRepo, conversationRepo, // conversationRepo 滿足 ConversationLinker
        refreshStore, familyRevoker, ticketStore,
        cfg.JWT, cfg.WSTicket, cfg.Visitor, cfg.BcryptCost,
    )

    // LLM Provider Registry
    llmReg := llm.NewRegistry()
    if cfg.LLM.ClaudeAPIKey != "" {
        llmReg.Register(claude.New(cfg.LLM.ClaudeAPIKey, cfg.LLM.ClaudeDefaultModel, cfg.LLM.ClaudeAPIBaseURL))
    }
    llmReg.SetFallback(cfg.LLM.Provider)

    // Chat dispatcher（per-conversation goroutine）
    dispatcher := dispatcher.New(llmReg, conversationRepo, cfg.Chat)

    userHandler   := handlers.NewUserHandler(userService)
    authHandler   := handlers.NewAuthHandler(authService, *cfg)
    wsHandler     := handlers.NewWSHandler(authService, dispatcher, cfg.WS)
    healthHandler := handlers.NewHealthHandler(pool, cache.RedisPinger{Client: rdb}, cfg.App.HealthPingTimeout)

    jwtMw   := middleware.NewJWTMiddleware(cfg.JWT.Secret, familyRevoker)
    limiter := middleware.NewLimiter(rdb)

    // 路由
    //
    // Middleware 套用順序對齊 [09-middleware.md](./09-middleware.md#套用順序)：
    //   RequestID → Logger → Recoverer → SecureHeaders → CORS → BodyLimit
    // 順序變更前請先閱讀 09-middleware 的「順序設計考量」表，每個位置都有原因。
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

    // 公開：註冊 / 登入 / 刷新（無需 JWT；以 IP 限流）
    r.Route("/api/v1/auth", func(r chi.Router) {
        r.Use(middleware.Timeout(cfg.App.AuthRequestTimeout))
        r.With(limiter.PerIP(redis_rate.PerMinute(cfg.RateLimit.LoginPerMinute), "login")).
            Method("POST", "/login", httpx.Func(authHandler.Login))
        r.With(limiter.PerIP(redis_rate.PerHour(cfg.RateLimit.RegisterPerHour), "register")).
            Method("POST", "/register", httpx.Func(authHandler.Register))
        r.With(limiter.PerIP(redis_rate.PerMinute(cfg.RateLimit.RefreshPerMinute), "refresh")).
            Method("POST", "/refresh", httpx.Func(authHandler.Refresh))
    })

    // 匿名訪客：從 visitor cookie 申請 anonymous WS ticket（不掛 JWT）
    r.Route("/api/v1/anonymous", func(r chi.Router) {
        r.Use(
            middleware.Timeout(cfg.App.AnonymousRequestTimeout),
            limiter.PerIP(redis_rate.PerMinute(cfg.RateLimit.AnonymousTicketPerMinute), "anon_ticket"),
        )
        r.Method("POST", "/ws-ticket", httpx.Func(authHandler.AnonymousWSTicket))
    })

    // 受保護：所有 /api/v1/* 其它路徑（JWT + user-based 限流）
    r.Group(func(r chi.Router) {
        r.Use(
            jwtMw.Verify,
            limiter.PerUser(redis_rate.PerMinute(cfg.RateLimit.AuthedPerMinute), "authed"),
            middleware.Timeout(cfg.App.RequestTimeout),
        )
        httpx.Post(r, "/api/v1/auth/logout",       authHandler.Logout)
        httpx.Post(r, "/api/v1/auth/ws-ticket",    authHandler.WSTicket)
        httpx.Post(r, "/api/v1/auth/link-visitor", authHandler.LinkVisitor)
        httpx.Get(r,  "/api/v1/users/{id}",        userHandler.GetUser)
        httpx.Put(r,  "/api/v1/users/{id}",        userHandler.UpdateUser)
    })

    // WebSocket 升級：以 ticket 驗證，不掛 JWT，不套 RequestTimeout（長連線）
    r.Group(func(r chi.Router) {
        // RequestID / Logger / Recoverer 仍套用；不用 BodyLimit（WS 升級無 body）
        r.Get("/api/v1/ws/customer", wsHandler.Customer)
        r.Get("/api/v1/ws/agent",    wsHandler.Agent)
    })

    // otelhttp 包在 chi 外面：下游 middleware（含 RequestID）可從 ctx 取得 span
    rootHandler := otelhttp.NewHandler(r, "http.server",
        otelhttp.WithFilter(func(r *http.Request) bool {
            return r.URL.Path != "/healthz" && r.URL.Path != "/readyz"
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
    // OTel：在 server 停止後 flush 殘留 span / metric，再讓 defer 關 pool / rdb
    if err := otelShutdown(ctx); err != nil {
        slog.Error("otel shutdown failed", "err", err)
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
otelShutdown(ctx)          ← flush 殘留 trace span / metric 至 collector
   │
   ▼
defer rdb.Close()          ← Redis 連線關閉
   │
   ▼
defer pool.Close()         ← Postgres 連線池關閉
```

**注意**：使用 `defer` 順序（LIFO）確保 server 完全停止後再關 DB / Redis；若反向順序，仍在處理的請求會打到關閉的連線。OTel 在 server 停止後、DB / Redis 關閉前 flush——確保最後一批 span 能送出。

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

### docker-compose.yml

```yaml
# backend/docker-compose.yml
services:
  backend:
    build: .
    container_name: helpzy-backend
    ports:
      - "8080:8080"
    env_file:
      - .env.local
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/healthz"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres:
    image: postgres:16-alpine
    container_name: helpzy-postgres
    environment:
      POSTGRES_USER:     ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB:       ${DB_NAME}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: helpzy-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
```

**設計要點**：

- `depends_on: condition: service_healthy` — backend 等 DB / Redis healthcheck 通過才啟動，避免冷啟動連線失敗
- `volumes: postgres-data` — 命名 volume 持久化資料；`docker compose down` 不會清空
- `env_file: .env.local` — local 開發從 `.env.local` 注入；production 由 orchestrator 注入
- backend 的 healthcheck 走 `/healthz`（liveness），不檢查外部依賴；details 見 [10-health.md](./10-health.md)

---

## 環境變數

| 變數 | 預設 | 說明 |
|------|------|------|
| `SHUTDOWN_TIMEOUT` | `15` | 等待 in-flight 請求完成的最長時間（秒）；超過則強制中斷 |
| `READ_HEADER_TIMEOUT` | `5` | `http.Server` 讀取請求 header 的最長時間（秒，Slowloris 防護） |
| `READ_TIMEOUT` | `30` | `http.Server` 讀取整個 request 的最長時間（秒） |
| `WRITE_TIMEOUT` | `30` | `http.Server` 寫入 response 的最長時間（秒） |
| `IDLE_TIMEOUT` | `120` | `http.Server` keep-alive 閒置上限（秒） |
| `REQUEST_TIMEOUT` | `10` | 單一請求 handler 最長執行時間（秒） |
| `MAX_BODY_BYTES` | `1048576` | request body 大小上限（bytes，1 MiB） |
