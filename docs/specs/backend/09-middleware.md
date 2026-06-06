# 09 — Middleware 規格

## 套用順序

```
Request
  │
  ▼
RequestID       ← 產生 / 沿用 X-Request-ID，注入 context
  │
  ▼
Logger          ← 記錄所有請求（含 request_id）
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

---

## RequestID Middleware

為每個請求產生（或沿用 client 帶入的）`X-Request-ID`，注入至 `context` 與 response header，供下游 logger、錯誤追蹤、跨服務串接使用。

```go
// pkg/middleware/requestid.go
const HeaderRequestID = "X-Request-ID"

func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        rid := r.Header.Get(HeaderRequestID)
        if rid == "" {
            rid = uuid.NewString()
        }
        w.Header().Set(HeaderRequestID, rid)
        ctx := logger.WithRequestID(r.Context(), rid)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

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
- 只允許單一 origin（BFF 位址），不使用萬用字元，降低跨站風險
- `AllowCredentials: true` — BFF 會帶 `Authorization` header
- Preflight `OPTIONS` 請求由套件自動處理

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

JWT 解析規格詳見 [jwt-auth-spec.md](./jwt-auth-spec.md)。

```go
func (m *JWTMiddleware) Verify(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := extractBearerToken(r)
        claims, err := auth.ParseJWT(token, m.secret)
        if err != nil {
            httpx.WriteError(w, r, apperror.Unauthorized("invalid_token", "未授權").Wrap(err))
            return
        }

        // 1) 注入 context 給 handler / service 使用
        ctx := context.WithValue(r.Context(), ContextKeyUserID, claims.Sub)
        ctx  = context.WithValue(ctx, ContextKeyRole, claims.Role)

        // 2) 同步注入 logger，讓後續所有日誌帶上 user_id
        ctx = logger.WithUserID(ctx, claims.Sub)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**注意**：
- 失敗路徑改用 `httpx.WriteError` 而非 `response.Error`，與其它層的錯誤回應格式一致
- `logger.WithUserID` 必須與 `ContextKeyUserID` 一起注入，否則 `user_id` 永遠不會出現在 log

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

## Context Keys

```go
type contextKey string

const (
    ContextKeyUserID contextKey = "userId"
    ContextKeyRole   contextKey = "role"
)
```

Handler 取用方式：

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) error {
    userID := r.Context().Value(ContextKeyUserID).(string)
    // ...
}
```

---

## 測試策略

- 各 middleware 以 `httptest` 模擬 HTTP 請求驗證行為
- JWT middleware：失敗路徑斷言回傳 401；成功路徑斷言 context 注入正確
- Recoverer：製造 panic 斷言回傳 500 且格式符合 AppError shape
