# 16 — 可觀測性（Tracing + Metrics + Log Correlation）

> 與 [03-logger.md](./03-logger.md) 合稱「三大可觀測性訊號」（logs / metrics / traces）。本檔規範 OpenTelemetry 整合。

## 目的

- 為每個請求產出 **distributed trace**，跨 service / DB / Redis 在同一條時間軸上看到所有 span
- 暴露 **RED metrics**（rate / errors / duration），供 SLO 監控與告警
- **trace_id ↔ request_id ↔ log** 共用同一識別碼，oncall debug 從一個 ID 就能拉到所有訊號

## 選型

| 元件 | 套件 | 理由 |
|------|------|------|
| Core SDK | `go.opentelemetry.io/otel` | CNCF 標準、廠商中立 |
| Trace exporter | `otlptracehttp` | OTLP/HTTP 經 collector 轉發至 Jaeger / Tempo / Datadog / Honeycomb 等 |
| Metric exporter | `otlpmetrichttp` | 同上 |
| HTTP 自動化 | `contrib/instrumentation/net/http/otelhttp` | 自動產生 server span + RED metrics |
| pgx 自動化 | `github.com/exaring/otelpgx` | pgx v5 官方推薦 tracer |
| Redis 自動化 | `redis/go-redis/extra/redisotel/v9` | go-redis 官方 OTel adapter |

> **為什麼用 OTLP/HTTP 不用 OTLP/gRPC**：HTTP 經得起企業內網 proxy / firewall；資料量大時兩者吞吐差距小，但 HTTP 對接複雜環境的 debug 成本低很多。

---

## 初始化

```go
// pkg/observability/otel.go
package observability

import (
    "context"
    "errors"
    "fmt"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/propagation"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

type Config struct {
    Enabled        bool
    Endpoint       string  // OTLP collector，例：otel-collector:4318
    ServiceName    string  // 例：helpzy-backend
    ServiceVersion string  // build 時注入（commit SHA 或 semver）
    Env            string  // local / staging / production
    SampleRatio    float64 // 0.0–1.0；production 建議 0.05–0.1 控制成本
}

// Init 初始化 tracer + meter provider，回傳 shutdown 函式（main.go 用 defer 呼叫）。
// 若 cfg.Enabled = false，回傳 noop shutdown（不報錯）——所有 OTel API 仍可呼叫但不會送出。
func Init(ctx context.Context, cfg Config) (shutdown func(context.Context) error, err error) {
    if !cfg.Enabled {
        return func(context.Context) error { return nil }, nil
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(cfg.ServiceName),
            semconv.ServiceVersion(cfg.ServiceVersion),
            semconv.DeploymentEnvironment(cfg.Env),
        ),
    )
    if err != nil {
        return nil, fmt.Errorf("otel resource: %w", err)
    }

    // ── Tracer ────────────────────────────────────────────
    traceExp, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint(cfg.Endpoint),
        otlptracehttp.WithInsecure(), // 內網走 collector；對外請改 TLS
    )
    if err != nil {
        return nil, fmt.Errorf("otlp trace exporter: %w", err)
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(traceExp),
        sdktrace.WithResource(res),
        // ParentBased：上游已決定取樣時尊重其決定，否則依比例取樣
        sdktrace.WithSampler(sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(cfg.SampleRatio),
        )),
    )
    otel.SetTracerProvider(tp)

    // 接收 client 傳入的 W3C traceparent，跨服務串接
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    // ── Meter ─────────────────────────────────────────────
    metricExp, err := otlpmetrichttp.New(ctx,
        otlpmetrichttp.WithEndpoint(cfg.Endpoint),
        otlpmetrichttp.WithInsecure(),
    )
    if err != nil {
        return nil, fmt.Errorf("otlp metric exporter: %w", err)
    }

    mp := sdkmetric.NewMeterProvider(
        sdkmetric.WithReader(sdkmetric.NewPeriodicReader(metricExp)),
        sdkmetric.WithResource(res),
    )
    otel.SetMeterProvider(mp)

    // 關閉順序：tracer 先 flush（含未送的 span），meter 再 flush
    return func(ctx context.Context) error {
        var errs []error
        if err := tp.Shutdown(ctx); err != nil {
            errs = append(errs, fmt.Errorf("trace shutdown: %w", err))
        }
        if err := mp.Shutdown(ctx); err != nil {
            errs = append(errs, fmt.Errorf("meter shutdown: %w", err))
        }
        return errors.Join(errs...)
    }, nil
}
```

---

## 自動化儀器（Instrumentation）

### HTTP 入口

`otelhttp.NewHandler` 包在 chi router **外面**——使下游 middleware（含 RequestID）都能從 ctx 取得 span：

```go
// cmd/server/main.go（節錄）
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

srv := &http.Server{
    // ...
    Handler: otelhttp.NewHandler(r, "http.server",
        // 探針端點不收 trace 與 metric，避免 baseline 噪音
        otelhttp.WithFilter(func(r *http.Request) bool {
            return r.URL.Path != "/healthz" && r.URL.Path != "/readyz"
        }),
    ),
}
```

自動產生：
- Server-side span（含 method、route、status_code、duration）
- 標準 metrics：`http.server.duration`、`http.server.request.body.size`、`http.server.active_requests`
- 從 incoming `traceparent` header 萃取 parent context

### pgx

```go
// pkg/database/postgres.go（補強）
import "github.com/exaring/otelpgx"

func New(c config.DBConfig) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(buildDSN(c))
    if err != nil { return nil, fmt.Errorf("parse db url: %w", err) }
    cfg.MaxConns        = int32(c.MaxConns)
    cfg.MinConns        = int32(c.MinConns)
    cfg.MaxConnLifetime = c.MaxConnLifetime
    cfg.MaxConnIdleTime = c.MaxConnIdleTime
    cfg.ConnConfig.Tracer = otelpgx.NewTracer() // ← 新增

    pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
    if err != nil { return nil, fmt.Errorf("create pool: %w", err) }
    return pool, nil
}
```

每個 SQL 都會帶 span（含 query text、duration、rows affected）。

### Redis

```go
// pkg/cache/redis.go（補強）
import "github.com/redis/go-redis/extra/redisotel/v9"

func New(url string) (*redis.Client, error) {
    opt, err := redis.ParseURL(url)
    if err != nil { return nil, fmt.Errorf("parse redis url: %w", err) }
    rdb := redis.NewClient(opt)

    if err := redisotel.InstrumentTracing(rdb); err != nil {
        return nil, fmt.Errorf("redis otel tracing: %w", err)
    }
    if err := redisotel.InstrumentMetrics(rdb); err != nil {
        return nil, fmt.Errorf("redis otel metrics: %w", err)
    }
    return rdb, nil
}
```

---

## trace_id ↔ request_id 統一

[09-middleware.md](./09-middleware.md) 的 `RequestID` middleware 優先使用 trace_id 作為 request_id，使 log / trace / 錯誤回應共用同一識別碼。完整實作見該檔的 RequestID 段落。

> **為什麼這樣設計**：使用者向 oncall 回報「請求 ID `abc123` 出錯」時，可直接用該 ID 在 Tempo / Jaeger 找 trace，在 Loki / ELK 找 log，在 Grafana 看當下指標——無需任何 join。

---

## Log correlation

`pkg/logger.From(ctx)` 取出 logger 時，自動補上 trace context：

```go
// pkg/logger/logger.go（補強，完整版見 03-logger.md）
import "go.opentelemetry.io/otel/trace"

func From(ctx context.Context) *slog.Logger {
    l := slog.Default()
    if stored, ok := ctx.Value(ctxKey{}).(*slog.Logger); ok {
        l = stored
    }
    if span := trace.SpanFromContext(ctx); span.SpanContext().IsValid() {
        sc := span.SpanContext()
        l = l.With(
            "trace_id", sc.TraceID().String(),
            "span_id",  sc.SpanID().String(),
        )
    }
    return l
}
```

每筆 log 都會附帶 `trace_id` / `span_id`，可在 trace UI 中以 log 連結直接跳轉。

---

## 業務指標（範例）

`otelhttp` 已涵蓋 HTTP-level RED metrics；應用層自訂指標統一於 `pkg/observability/metrics.go`：

```go
// pkg/observability/metrics.go
package observability

import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/metric"
)

var (
    meter = otel.Meter("helpzy/business")

    LoginAttempts, _ = meter.Int64Counter("auth.login.attempts",
        metric.WithDescription("Total login attempts"))
    LoginFailures, _ = meter.Int64Counter("auth.login.failures",
        metric.WithDescription("Failed login attempts"))
    TokenReusedTotal, _ = meter.Int64Counter("auth.refresh.reused_total",
        metric.WithDescription("Detected refresh token reuse — potential theft"))
    RateLimitBlocked, _ = meter.Int64Counter("ratelimit.blocked_total",
        metric.WithDescription("Requests rejected by rate limiter"))
    RateLimitErrors, _ = meter.Int64Counter("ratelimit.errors_total",
        metric.WithDescription("Rate limiter backend errors (e.g., Redis down)"))
)
```

Service 層使用：

```go
import "go.opentelemetry.io/otel/attribute"

func (s *AuthService) Login(ctx context.Context, in LoginInput) (...) {
    observability.LoginAttempts.Add(ctx, 1)
    // ...
    if errors.Is(err, apperror.ErrUnauthorized) {
        observability.LoginFailures.Add(ctx, 1, metric.WithAttributes(
            attribute.String("reason", "invalid_credentials"),
        ))
    }
}
```

---

## 採樣策略

| 環境 | 建議 SampleRatio | 理由 |
|------|------------------|------|
| local | `1.0` | 開發 debug，全收 |
| staging | `1.0` | 量小，方便 QA 追問題 |
| production | `0.05–0.1` | 控制 collector 流量與儲存成本；錯誤路徑請以「tail sampling」在 collector 端 100% 保留 |

`ParentBased`：若上游服務已決定取樣（`traceparent` 的 `sampled` flag），則沿用其決定——確保跨服務一條 trace 不會被部分取樣切斷。

---

## 環境變數

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `OTEL_ENABLED` | ✗ | `false` | 是否啟用 OTel；local 通常關閉 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `OTEL_ENABLED=true` 時必填 | — | OTLP collector 位址，例：`otel-collector:4318` |
| `OTEL_SERVICE_NAME` | ✗ | `helpzy-backend` | `service.name` resource attribute |
| `OTEL_SERVICE_VERSION` | ✗ | `dev` | 通常 build 時以 `-ldflags` 注入 commit SHA |
| `OTEL_SAMPLE_RATIO` | ✗ | `1.0` | trace 取樣率 [0.0, 1.0] |

---

## 與其它規格的關係

| 規格 | 關聯 |
|------|------|
| [03-logger.md](./03-logger.md) | `From()` 自動帶 `trace_id` / `span_id` 欄位 |
| [05-httpx.md](./05-httpx.md) | 錯誤回應的 `request_id` = trace_id（如有） |
| [07-database.md](./07-database.md) | pgx tracer 注入 |
| [08-redis.md](./08-redis.md) | go-redis OTel instrument |
| [09-middleware.md](./09-middleware.md) | RequestID 優先採 trace_id；otelhttp 在最外層 |
| [12-startup.md](./12-startup.md) | OTel init / shutdown 整合至 main.go |

---

## 測試策略

- `observability.Init`：以 OTel 內建 `tracetest.InMemoryExporter` 替換 OTLP exporter，斷言 span 結構與 attributes
- `RequestID` middleware：模擬有 / 無 incoming `traceparent`，斷言 `X-Request-ID` 來源（trace_id vs uuid）
- `logger.From`：模擬 ctx 內含 valid span，斷言輸出含 `trace_id` / `span_id`
- 業務 counters：以 `metric/metricdata` testkit 斷言增量
- Init 在 `Enabled=false` 時：所有 OTel API 仍可呼叫不 panic、shutdown 為 no-op
