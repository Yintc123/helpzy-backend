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
SecureHeaders   ← X-Content-Type-Options / Frame-Options / HSTS / CSP 等
  │
  ▼
Logger          ← 記錄所有請求（含 request_id / trace_id）
  │
  ▼
CORS            ← 設定跨域 Header
  │
  ▼
Recoverer       ← 捕捉 panic，回傳 500
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
Timeout（群組內）← 設定請求最大執行時間
  │
  ▼
Handler
```

> **為什麼 RateLimit 放在 JWT 之前**：未驗證路徑（login / register / refresh）一定要在 JWT 之外限流；已驗證路徑可在 JWT 之後另外掛一層 user-based RateLimit。順序由各 route group 自行決定，套用範例見 [12-startup.md](./12-startup.md)。

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

// statusWriter 攔截 WriteHeader 以記錄狀態碼
type statusWriter struct {
    http.ResponseWriter
    status int
}

func (sw *statusWriter) WriteHeader(code int) {
    sw.status = code
    sw.ResponseWriter.WriteHeader(code)
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

使用 `github.com/go-chi/cors`，從 `ALLOWED_ORIGIN` env 讀取允許的 Origin。

```go
// pkg/middleware/cors.go
func CORS(allowedOrigin string) func(http.Handler) http.Handler {
    return cors.Handler(cors.Options{
        AllowedOrigins:   []string{allowedOrigin},
        AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowedHeaders:   []string{"Authorization", "Content-Type"},
        AllowCredentials: true,
        MaxAge:           300,
    })
}
```

**設定理由**：
- 只允許單一 origin（Next.js 前端位址），不使用萬用字元，降低跨站風險
- `AllowCredentials: true` — Next.js client 會帶 `Authorization` header
- Preflight `OPTIONS` 請求由套件自動處理

---

## SecureHeaders Middleware

統一注入安全相關 response headers，由 `cfg.App.Env` 決定 production-only 設定（HSTS）。

```go
// pkg/middleware/secureheaders.go
type SecureHeadersConfig struct {
    HSTSEnabled bool  // 僅 production 啟用（HTTPS 環境）
    HSTSMaxAge  int   // 秒數，建議 31536000（1 年）
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

Backend ↔ Frontend 認證契約見 [../jwt-auth-spec.md](../jwt-auth-spec.md)；簽發 / 驗證 / blacklist 等實作細節見 [15-auth.md](./15-auth.md)。

```go
// 概要：完整實作（含 blacklist 查詢、refresh repo 等）見 15-auth.md
func (m *JWTMiddleware) Verify(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token, err := extractBearerToken(r)
        if err != nil {
            httpx.WriteError(w, r, err)
            return
        }
        claims, err := auth.ParseAccess(token, m.secret)
        if err != nil {
            httpx.WriteError(w, r, apperror.Unauthorized("invalid_token", "未授權").Wrap(err))
            return
        }
        // 查 Redis blacklist（撤銷的 jti）
        if revoked, _ := m.revokeRepo.IsRevoked(r.Context(), claims.Jti); revoked {
            httpx.WriteError(w, r, apperror.Unauthorized("invalid_token", "未授權"))
            return
        }
        // 注入 context（helpers 集中定義於 pkg/middleware/context.go，見下方）
        ctx := WithUserID(r.Context(), claims.Sub)
        ctx  = WithRole(ctx, claims.Role)
        ctx  = WithJTI(ctx, claims.Jti)
        ctx  = logger.WithUserID(ctx, claims.Sub)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**注意**：
- 失敗路徑改用 `httpx.WriteError` 而非 `response.Error`，與其它層的錯誤回應格式一致
- `logger.WithUserID` 必須與 `ContextKeyUserID` 一起注入，否則 `user_id` 永遠不會出現在 log
- JWT middleware 必須查 Redis blacklist 以支援主動撤銷（登出 / 異常處理）

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
    r.Use(middleware.Timeout(10 * time.Second))
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

## Context Keys 與存取 Helper（單一來源）

中間件解析出的請求屬性（`userId` / `role` / `jti`）一律透過 `pkg/middleware/context.go` 集中讀寫。
**任何其他檔案（含 [15-auth.md](./15-auth.md) 的 JWT middleware）一律引用本節定義，禁止重複宣告 `ctxKey` 型別。**

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
