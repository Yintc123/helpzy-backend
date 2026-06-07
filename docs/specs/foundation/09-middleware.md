# 09 — Middleware 規格

## 套用順序

```
Request
  │
  ▼
otelhttp        ← chi router 之外，產 server span + RED metrics（見 16-observability.md）
  │
  ▼
RequestID       ← 優先使用 trace_id 作為 X-Request-ID；無 span 時退而 uuid
  │
  ▼
Logger          ← 記錄所有請求（含 request_id / trace_id）；statusWriter 攔截下游狀態碼
  │
  ▼
Recoverer       ← 捕捉 panic，回傳 500（涵蓋 SecureHeaders / CORS / BodyLimit 以下所有層）
  │
  ▼
SecureHeaders   ← X-Content-Type-Options / Frame-Options / HSTS / CSP 等
  │
  ▼
CORS            ← 設定跨域 Header
  │
  ▼
BodyLimit       ← 限制 request body 大小（防 OOM）
  │
  ▼
RateLimit（群組內）← 公開端點以 IP、已驗證端點以 user_id 計數
  │
  ▼
JWT（群組內）   ← 驗證 JWT，注入 userId / role / familyId 至 context
  │
  ▼
Timeout（群組內，HTTP 路由）← 設定請求最大執行時間；WS / SSE 路由群組不掛此項
  │
  ▼
Handler
```

### 順序設計考量

| 相對位置 | 理由 |
|---------|------|
| `RequestID` 最外 | 後續 middleware（含 Logger / Recoverer）的 log 才能帶上同一 ID |
| `Logger` 在 `Recoverer` 之外 | `Logger` 包 `statusWriter` 傳給下游；`Recoverer` 寫的 500 必須被 `statusWriter` 攔截，否則 access log 會誤記為 200 |
| `Recoverer` 在 `SecureHeaders` / `CORS` 之外（更外層） | 最大化 panic 涵蓋範圍：任何下游 middleware（含套件 bug）panic 都能被接住寫 500 |
| `SecureHeaders` / `CORS` 在 `Recoverer` 之內 | 兩者皆「先 `w.Header().Set`，再 `next.ServeHTTP`」——panic 穿透時 header 已在 `ResponseWriter` 上，Recoverer 寫 500 時會一併送出，CORS header 不會遺失 |
| `RateLimit` 在 `JWT` 之前（公開路由群組） | 未驗證路徑（login / register / refresh）必須在 JWT 外側限流 |
| `RateLimit` 在 `JWT` 之後（已驗證路由群組） | 拿到 `user_id` 才能 per-user 計數 |
| `Timeout` 在群組內最後 | 排除 WS / SSE 路由群組（見 [12-startup.md](./12-startup.md)「streaming 路由 middleware stack」）|

> **WS / SSE 路由排除 `Timeout`**：`middleware.Timeout` 用 `context.WithTimeout` 設 deadline，會直接打死長連線。所有 streaming 路由（`/api/v1/ws/*`、未來的 SSE 端點）必須掛在獨立 chi Group，**不套 `middleware.Timeout`**。同時 handler 內須以 `http.ResponseController.SetWriteDeadline(time.Time{})` 清除 `http.Server.WriteTimeout` 的影響。

---

## RequestID Middleware

為每個請求產生（或沿用 client 帶入的）`X-Request-ID`，注入至 `context` 與 response header，供下游 logger、錯誤追蹤、跨服務串接使用。

```go
// pkg/middleware/requestid.go
import "go.opentelemetry.io/otel/trace"

const HeaderRequestID = "X-Request-ID"

func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        rid := r.Header.Get(HeaderRequestID)
        if rid == "" {
            // 若 otelhttp 已啟動 span，沿用 trace_id 讓 log / trace / 錯誤回應共享同一 ID
            if span := trace.SpanFromContext(r.Context()); span.SpanContext().HasTraceID() {
                rid = span.SpanContext().TraceID().String()
            } else {
                rid = uuid.NewString()
            }
        }
        w.Header().Set(HeaderRequestID, rid)
        ctx := logger.WithRequestID(r.Context(), rid)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

> **與 OTel 對接**：詳見 [16-observability.md](./16-observability.md)。`otelhttp.NewHandler` 必須包在 chi router 外面，本 middleware 才能在 ctx 內找到 valid span。

---

## Logger Middleware

使用 `pkg/logger` 取得帶有 `request_id` 的 logger，輸出結構化請求日誌。

```go
// pkg/middleware/logger.go
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        sw    := &statusWriter{ResponseWriter: w, status: http.StatusOK}
        next.ServeHTTP(sw, r)
        logger.From(r.Context()).Info("request",
            "method",      r.Method,
            "path",        r.URL.Path,
            "status",      sw.status,
            "duration_ms", time.Since(start).Milliseconds(),
            "remote",      r.RemoteAddr,
            "user_agent",  r.UserAgent(),
        )
    })
}

// statusWriter 攔截 WriteHeader 以記錄狀態碼。
//
// 必須實作 Unwrap()：Go 1.20+ 的 http.ResponseController（WebSocket Hijack、
// SSE Flush、SetWriteDeadline 等都走它）透過 Unwrap 鏈找到具備能力的底層 writer，
// 不會穿透 embed promotion。沒有這個方法 → 任何下游 handler 想 Hijack/Flush 都會
// 拿到 "feature not supported"，WS 升級會直接失敗。
// 詳細討論見 [decisions/responsewriter-unwrap.md](../../decisions/responsewriter-unwrap.md)。
type statusWriter struct {
    http.ResponseWriter
    status int
}

func (sw *statusWriter) WriteHeader(code int) {
    sw.status = code
    sw.ResponseWriter.WriteHeader(code)
}

func (sw *statusWriter) Unwrap() http.ResponseWriter {
    return sw.ResponseWriter
}
```

**欄位**：

| 欄位 | 說明 |
|------|------|
| `request_id` | 由 logger 自動帶入（從 context） |
| `method` | HTTP method |
| `path` | Request path |
| `status` | HTTP status code |
| `duration_ms` | 處理時間（毫秒） |
| `remote` | Client 位址 |
| `user_agent` | User-Agent |

---

## CORS Middleware

使用 `github.com/go-chi/cors`；`allowedOrigins` 來自 `ALLOWED_ORIGINS`、`maxAge` 來自 `CORS_MAX_AGE`。

```go
// pkg/middleware/cors.go
func CORS(allowedOrigins []string, maxAge int) func(http.Handler) http.Handler {
    return cors.Handler(cors.Options{
        AllowedOrigins:   allowedOrigins,
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"Authorization", "Content-Type", "X-CSRF-Token", "X-Request-ID"},
        ExposedHeaders:   []string{"X-Request-ID"},
        AllowCredentials: true,
        MaxAge:           maxAge,
    })
}
```

**設定理由**：
- 採白名單（精確比對），不使用萬用字元，降低跨站風險；多前端站點（marketing / app / admin）以逗號分隔列入
- `AllowCredentials: true` — Next.js client 會帶 `Authorization` header
- `AllowedHeaders` 含 `X-CSRF-Token`（BFF 端會送）與 `X-Request-ID`（trace 串接）
- `ExposedHeaders` 含 `X-Request-ID`，讓 client 在 debug 時能讀到 server 端的 request id
- `MaxAge` 抽至 config（[02-config.md](./02-config.md) `CORS_MAX_AGE`），便於不同瀏覽器 / CDN 策略調校
- Preflight `OPTIONS` 請求由套件自動處理

---

## SecureHeaders Middleware

統一注入安全相關 response headers，由 `cfg.App.Env` 決定 production-only 設定（HSTS）。

```go
// pkg/middleware/secureheaders.go
type SecureHeadersConfig struct {
    HSTSEnabled bool  // 僅 production 啟用（HTTPS 環境）
    HSTSMaxAge  int   // 秒數，來自 cfg.App.HSTSMaxAge（首次上線建議 300 → 86400 → 31536000 階段性調高）
}

func SecureHeaders(cfg SecureHeadersConfig) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            h := w.Header()
            // 防止 MIME sniff 攻擊
            h.Set("X-Content-Type-Options", "nosniff")
            // 後端為純 API，禁止任何頁面嵌入
            h.Set("X-Frame-Options", "DENY")
            // 不外洩 referrer
            h.Set("Referrer-Policy", "no-referrer")
            // Cross-Origin 隔離
            h.Set("Cross-Origin-Opener-Policy",   "same-origin")
            h.Set("Cross-Origin-Resource-Policy", "same-site")
            // API 只回 JSON，最嚴格 CSP——零內嵌資源
            h.Set("Content-Security-Policy",
                "default-src 'none'; frame-ancestors 'none'")
            if cfg.HSTSEnabled {
                h.Set("Strict-Transport-Security",
                    fmt.Sprintf("max-age=%d; includeSubDomains", cfg.HSTSMaxAge))
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

**設計理由**：

| Header | 為什麼 |
|--------|--------|
| `X-Content-Type-Options: nosniff` | 阻止瀏覽器猜測 content-type；單純 JSON API 也必設 |
| `X-Frame-Options: DENY` | 後端不該被任何頁面 `<iframe>`，含 attacker 控制的網站 |
| `Referrer-Policy: no-referrer` | API 回應不需要 referrer；外洩來源資訊只有壞處 |
| `Cross-Origin-*` | 即使是 JSON 也設，阻止 Spectre 類旁路攻擊 |
| `Content-Security-Policy: default-src 'none'` | API 不返回任何嵌入式資源；如未來改回 HTML 再放寬 |
| `Strict-Transport-Security` | 僅 production 開啟（local 是 HTTP），瀏覽器強制走 HTTPS |

> **HSTS 守則**：`max-age` 一旦設定後客戶端會記住，誤開啟後切回 HTTP 會讓使用者 lockout。所以 `HSTSEnabled` 嚴格依 production 環境決定，由 [02-config.md](./02-config.md) 控制。

---

## Recoverer Middleware

捕捉下游 handler / middleware 未處理的 panic，記錄 stack trace 後回傳 500。

```go
// pkg/middleware/recoverer.go
func Recoverer(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                logger.From(r.Context()).Error("panic recovered",
                    "panic", rec,
                    "stack", string(debug.Stack()),
                    "path",  r.URL.Path,
                )
                // 將 panic 轉成 error 丟給 httpx.WriteError，
                // 確保 500 回應格式與一般業務錯誤一致（kind + message）。
                httpx.WriteError(w, r, apperror.Internal(fmt.Errorf("panic: %v", rec)))
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**注意**：
- 依套用順序，Recoverer 在 Logger 之內。panic 被 Recoverer 接走後寫入 `500`，Logger 仍透過 `statusWriter` 捕捉到正確狀態碼
- Recoverer 與 httpx 透過 `httpx.WriteError` 共用同一個 500 回應格式，client 不會看到兩種 shape

**職責邊界**：
- `Recoverer` 只處理「未預期的 panic」（nil pointer、index out of range、第三方套件 panic）
- 「業務錯誤」由 handler 以 `return err` 拋出，交給 `pkg/httpx` 中央寫入器處理
- 兩者互不重疊；handler 內禁止用 `panic` 表達業務錯誤

---

## JWT Middleware

驗證 access token、查 family 撤銷狀態，並注入 `userId` / `role` / `jti` / `familyId` 至 context。

**完整實作**（`JWTMiddleware` struct、`Verify` 方法、`extractBearerToken` helper、錯誤 kind 對應表）見 [15-auth-mechanism.md](./15-auth-mechanism.md#pkgmiddlewarejwtgo)。本檔只規範它在 middleware stack 內的位置與職責邊界：

| 項目 | 規範 |
|------|------|
| 套用順序 | 群組內、`RateLimit`（per-user）之前、`Timeout` 之前；見上方順序圖 |
| 依賴 | `authstore.FamilyRevoker` interface（機制層），不直接認 Redis client |
| 失敗回應 | 一律走 `httpx.WriteError`，與其它層的 AppError shape 一致 |
| context 注入 | 必須同時呼叫 `logger.WithUserID(...)`，否則 access log 不會帶 user_id |
| 撤銷檢查 | 驗章後查 family 撤銷，支援登出 / reuse detected / 主動撤銷整條鏈失效 |

Backend ↔ Frontend 認證契約見 [../jwt-auth-spec.md](../jwt-auth-spec.md)。

---

## BodyLimit Middleware

限制每個請求的 body 大小（`MAX_BODY_BYTES`，預設 1 MiB），避免攻擊者送超大 body 把記憶體 / DB 撐爆。

```go
// pkg/middleware/bodylimit.go
func BodyLimit(maxBytes int64) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
            next.ServeHTTP(w, r)
        })
    }
}
```

**運作原理**：
- `http.MaxBytesReader` 是 **lazy**——只在實際讀取 body 時才檢查
- 超過上限時，下游 reader（`json.Decode` 等）會收到 `*http.MaxBytesError`
- 這個錯誤由 [`httpx.DecodeJSON`](./05-httpx.md#decodejson-helper) 統一映射為 `413 Payload Too Large`

**套用位置**：放在 `Recoverer` 之內、`JWT` 之外——讓 panic 仍能被捕捉，且未驗證的請求也受體積保護。

---

## RateLimit Middleware

### 目的

防止暴力破解（login）、token 探測（refresh）、註冊洪水（register）、單一 user 濫用 API。
未限流的 auth 端點在生產環境上線一週內幾乎必定被掃。

### 選型：`redis_rate/v10`

`github.com/go-redis/redis_rate/v10` 採用 GCRA（Generic Cell Rate Algorithm），go-redis 官方副專案，特性：

- **單一 Redis Lua script** 完成 check + decrement，原子操作無 race
- 內建標準 RateLimit headers 回傳值（`Limit`、`Remaining`、`ResetAfter`、`RetryAfter`）
- 跨實例共享狀態：水平擴展不會出現「每個 pod 各自 5 次 → 總共 N×5」的破洞

### 實作

```go
// pkg/middleware/ratelimit.go
package middleware

import (
    "net"
    "net/http"
    "strconv"
    "strings"

    "github.com/go-redis/redis_rate/v10"
    "github.com/redis/go-redis/v9"

    "helpzy/pkg/apperror"
    "helpzy/pkg/httpx"
    "helpzy/pkg/logger"
    "helpzy/pkg/observability"
)

type Limiter struct {
    limiter *redis_rate.Limiter
}

func NewLimiter(rdb *redis.Client) *Limiter {
    return &Limiter{limiter: redis_rate.NewLimiter(rdb)}
}

// PerIP 以 client IP 計數，scope 區隔不同端點群（例：login / register）。
func (l *Limiter) PerIP(rate redis_rate.Limit, scope string) func(http.Handler) http.Handler {
    return l.build(rate, scope, ipKey)
}

// PerUser 以 user_id 計數；必須掛在 JWTMiddleware 之後。
// 缺 user_id 時 fallback 至 IP，避免漏限流。
func (l *Limiter) PerUser(rate redis_rate.Limit, scope string) func(http.Handler) http.Handler {
    return l.build(rate, scope, userKey)
}

func (l *Limiter) build(rate redis_rate.Limit, scope string, keyFn func(*http.Request) string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := "rl:" + scope + ":" + keyFn(r)
            res, err := l.limiter.Allow(r.Context(), key, rate)
            if err != nil {
                // Redis 異常：fail-open 不阻擋業務，但記 warn + 計入指標讓 oncall 發現
                logger.From(r.Context()).Warn("rate limiter degraded",
                    "err", err, "scope", scope)
                observability.RateLimitErrors.Add(r.Context(), 1)
                next.ServeHTTP(w, r)
                return
            }

            h := w.Header()
            h.Set("RateLimit-Limit",     strconv.Itoa(res.Limit.Rate))
            h.Set("RateLimit-Remaining", strconv.Itoa(res.Remaining))
            h.Set("RateLimit-Reset",     strconv.Itoa(int(res.ResetAfter.Seconds())))

            if res.Allowed == 0 {
                h.Set("Retry-After", strconv.Itoa(int(res.RetryAfter.Seconds())))
                observability.RateLimitBlocked.Add(r.Context(), 1)
                httpx.WriteError(w, r,
                    apperror.TooManyRequests("rate_limited", "請求過於頻繁，請稍後再試"))
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// ipKey 取 client IP；信任 RemoteAddr，由反向代理（若有）負責改寫。
// 詳見下方「IP 來源信任邊界」。
func ipKey(r *http.Request) string {
    host, _, err := net.SplitHostPort(r.RemoteAddr)
    if err != nil {
        return strings.TrimSpace(r.RemoteAddr)
    }
    return host
}

// userKey 優先以已驗證 user_id 計數；無 user_id 時 fallback 至 IP，避免遺漏。
func userKey(r *http.Request) string {
    if uid, ok := UserIDFrom(r.Context()); ok {
        return "u:" + uid
    }
    return "ip:" + ipKey(r)
}
```

### IP 來源信任邊界

| 部署方式 | 處理 |
|---------|------|
| Backend 直接面對 client | `RemoteAddr` 即為真實 IP，無需處理 X-Forwarded-For |
| Backend 在 nginx / ELB / Cloudflare 後 | 在 RequestID **之前**掛 `chi/middleware.RealIP`，從可信代理的 `X-Forwarded-For` / `X-Real-IP` 改寫 `RemoteAddr` |

> **絕對禁止**：本 middleware 不直接讀 `X-Forwarded-For`——若 backend 沒有代理在前，攻擊者可任意偽造該 header 繞過限流。信任邊界由部署架構決定，集中在 RealIP middleware 配置一次。

### Fail-open vs Fail-closed

本實作**預設 fail-open**（Redis 故障時放行）——理由：

| 選項 | 優點 | 缺點 |
|------|------|------|
| fail-open | Redis 暫時故障不會擴大影響到全站登入 | Redis 故障 + 攻擊同時發生時失去防護 |
| fail-closed | 全程強制限流 | Redis 一抖動所有 auth 直接 down，DoS 攻擊面變大 |

業界主流選 fail-open + 監控 `ratelimit.errors_total` 警報——一旦 Redis 異常，oncall 應該已經被通知，而不是把問題傳染給使用者。

### 套用範例

```go
// cmd/server/main.go（節錄；完整範例見 12-startup.md）
rl := middleware.NewLimiter(rdb)

// 公開 auth 路由：以 IP 限流
r.Route("/api/v1/auth", func(r chi.Router) {
    r.Use(middleware.Timeout(cfg.App.AuthRequestTimeout))
    r.With(rl.PerIP(redis_rate.PerMinute(cfg.RateLimit.LoginPerMinute), "login")).
        Method("POST", "/login", httpx.Func(authHandler.Login))
    r.With(rl.PerIP(redis_rate.PerHour(cfg.RateLimit.RegisterPerHour), "register")).
        Method("POST", "/register", httpx.Func(authHandler.Register))
    r.With(rl.PerIP(redis_rate.PerMinute(cfg.RateLimit.RefreshPerMinute), "refresh")).
        Method("POST", "/refresh", httpx.Func(authHandler.Refresh))
})

// 受保護路由：以 user_id 限流
r.Group(func(r chi.Router) {
    r.Use(
        jwtMw.Verify,
        rl.PerUser(redis_rate.PerMinute(cfg.RateLimit.AuthedPerMinute), "authed"),
        middleware.Timeout(cfg.App.RequestTimeout),
    )
    // ...
})
```

### 守則

- `scope` 必須能區隔語義——`login` / `register` 算同一 IP 但用不同 quota
- 業務上需要「按 email 限流」（如註冊洪水）時，handler 內呼叫 `limiter.Allow(ctx, "register-email:"+email, rate)`；middleware 處理不到 body
- RateLimit middleware 內**不**做 ban / lockout 邏輯——這屬於 auth service 的職責（多次密碼錯誤 → 暫時鎖帳號），各自獨立

---

## Timeout Middleware

僅做 `context.WithTimeout` 把 deadline 推下去是不夠的——若 handler 不檢查 `ctx.Err()`、或 handler 已開始送 response，client 仍會收到「半個 200」+ 連線斷。本實作做三件事：

1. **推 ctx deadline**：DB / Redis / outbound HTTP 自動取消 in-flight 操作
2. **超時主動寫 504 + 統一 JSON shape**（與 `httpx.WriteError` 一致）
3. **後續寫入轉 no-op**：handler 即使繼續跑完，對 `ResponseWriter` 的寫入不再生效，避免「504 + 半個 body」

> **不適用於 streaming 路由**（WS / SSE）。`12-startup.md` 已將這些路由放在不套 `Timeout` 的群組。

```go
// pkg/middleware/timeout.go
package middleware

import (
    "context"
    "io"
    "net/http"
    "sync"
    "time"
)

const timeoutBody = `{"error":{"kind":"request_timeout","message":"請求逾時"}}`

func Timeout(d time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), d)
            defer cancel()

            tw := &timeoutWriter{ResponseWriter: w}
            done := make(chan struct{})
            var panicVal any

            go func() {
                defer func() {
                    panicVal = recover()
                    close(done)
                }()
                next.ServeHTTP(tw, r.WithContext(ctx))
            }()

            select {
            case <-done:
                if panicVal != nil {
                    panic(panicVal) // 推回 Recoverer
                }
            case <-ctx.Done():
                tw.writeTimeout()
            }
        })
    }
}

// timeoutWriter 在超時後對所有寫入轉 no-op，避免 handler goroutine 後續寫出污染 504 回應。
// 必須實作 Unwrap()，理由同 statusWriter（見 decisions/responsewriter-unwrap.md）。
type timeoutWriter struct {
    http.ResponseWriter
    mu          sync.Mutex
    timedOut    bool
    wroteHeader bool
}

func (tw *timeoutWriter) Header() http.Header {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    if tw.timedOut {
        return http.Header{}
    }
    return tw.ResponseWriter.Header()
}

func (tw *timeoutWriter) Write(p []byte) (int, error) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    if tw.timedOut {
        return len(p), nil
    }
    tw.wroteHeader = true
    return tw.ResponseWriter.Write(p)
}

func (tw *timeoutWriter) WriteHeader(code int) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    if tw.timedOut || tw.wroteHeader {
        return
    }
    tw.wroteHeader = true
    tw.ResponseWriter.WriteHeader(code)
}

func (tw *timeoutWriter) Unwrap() http.ResponseWriter {
    return tw.ResponseWriter
}

// writeTimeout 標記 timedOut 並寫入 504。
// 若 handler 已先送 header（極罕見的競態），本次寫入退回 no-op；
// client 端表現為「半截回應 + 連線斷」，但至少不會看到「504 + handler 的部分 body」這種混合。
func (tw *timeoutWriter) writeTimeout() {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    if tw.wroteHeader {
        tw.timedOut = true
        return
    }
    tw.timedOut = true
    tw.ResponseWriter.Header().Set("Content-Type", "application/json; charset=utf-8")
    tw.ResponseWriter.WriteHeader(http.StatusGatewayTimeout)
    _, _ = io.WriteString(tw.ResponseWriter, timeoutBody)
}
```

### 設計決策

| 決策 | 理由 |
|------|------|
| 用自己的 `timeoutWriter` 而非 `http.TimeoutHandler` | stdlib 版本內部 buffer 整個 response，與 streaming / Hijack 不相容；自訂版本只攔截「超時後寫入」 |
| 504 而非 408 | 408 語意是「client 沒把 request 送完」；超時發生在我們這側未完成處理 → 504 Gateway Timeout 較準確 |
| handler panic 推回 Recoverer | 在 goroutine 內 `recover()` 後重新 `panic`，讓上游 Recoverer middleware 走原本的 500 路徑、不繞過 |
| 鎖到 504 寫入結束 | handler goroutine 若同時寫，兩者都會走 `tw.mu` 序列化，避免雙重寫入 |
| 實作 `Unwrap()` | 與 `statusWriter` 同——讓 `http.ResponseController` 能穿透這層 wrapper |

---

## Context Keys 與存取 Helper（單一來源）

中間件解析出的請求屬性（`userId` / `role` / `jti`）一律透過 `pkg/middleware/context.go` 集中讀寫。
**任何其他檔案（含 [15-auth-mechanism.md](./15-auth-mechanism.md) 的 JWT middleware）一律引用本節定義，禁止重複宣告 `ctxKey` 型別。**

### 為什麼這樣設計

| 問題 | 對策 |
|------|------|
| 兩處宣告 `type ctxKey` → 同套件編譯衝突 | 集中於本檔案唯一定義 |
| Handler 用 `.(string)` 強制斷言 → 缺值即 panic | Getter 回傳 `(value, ok)` |
| 外部套件用同字串字面量撞 key | Key 型別未匯出，跨套件型別系統強制隔離 |
| 字面量字串拼錯不會被編譯器抓到 | Key 是常數，typo 直接編譯失敗 |

### 實作

```go
// pkg/middleware/context.go
package middleware

import "context"

// ctxKey 為未匯出型別——其他套件即使定義同名 ctxKey 也是不同型別，
// 物理上不可能撞 key。
type ctxKey int

const (
    userIDKey ctxKey = iota + 1 // 從 1 起跳，留下 0 作為「未設值」哨兵
    roleKey
    jtiKey
    familyIDKey
)

// ── 寫入：middleware 用 ────────────────────────────
func WithUserID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, userIDKey, id)
}

func WithRole(ctx context.Context, role string) context.Context {
    return context.WithValue(ctx, roleKey, role)
}

func WithJTI(ctx context.Context, jti string) context.Context {
    return context.WithValue(ctx, jtiKey, jti)
}

func WithFamilyID(ctx context.Context, fid string) context.Context {
    return context.WithValue(ctx, familyIDKey, fid)
}

// ── 讀取：handler / service 用 ─────────────────────
// 缺值時 ok = false；handler 不會因為斷言失敗 panic。
func UserIDFrom(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(userIDKey).(string)
    return id, ok
}

func RoleFrom(ctx context.Context) (string, bool) {
    role, ok := ctx.Value(roleKey).(string)
    return role, ok
}

func JTIFrom(ctx context.Context) (string, bool) {
    jti, ok := ctx.Value(jtiKey).(string)
    return jti, ok
}

func FamilyIDFrom(ctx context.Context) (string, bool) {
    fid, ok := ctx.Value(familyIDKey).(string)
    return fid, ok
}
```

### 設計重點

| 設計 | 理由 |
|------|------|
| `ctxKey` 未匯出 | 跨套件型別不同，撞 key 在編譯期被阻擋 |
| 用 `int + iota` 而非 `string` | 比較更快、無字面量重複風險 |
| Key 常數未匯出（`userIDKey`） | 外部只能走 `With*` / `*From`，封閉性最高 |
| Getter 回傳 `(value, ok)` | handler 缺值時可優雅回 401，不 panic |
| Setter 命名 `WithXxx` | 與 stdlib `context.WithValue` 風格一致 |

### Handler 取用範例

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) error {
    userID, ok := middleware.UserIDFrom(r.Context())
    if !ok {
        // 路由理應掛 JWT middleware；缺值代表路由配置漏掛 → 401
        return apperror.Unauthorized("missing_user_id", "未授權")
    }
    // ...
}
```

---

## 測試策略

- 各 middleware 以 `httptest` 模擬 HTTP 請求驗證行為
- JWT middleware：失敗路徑斷言回傳 401；成功路徑斷言 context 注入正確
- Recoverer：製造 panic 斷言回傳 500 且格式符合 AppError shape
