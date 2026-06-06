# 05 — 統一回應格式與 HTTP 錯誤處理層（httpx）

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
  "error": {
    "kind": "...",
    "message": "..."
  }
}
```

### response 套件

```go
// pkg/response/response.go

// JSON 回傳成功回應；encode 失敗時回傳 error 供 httpx.Func 上拋
func JSON(w http.ResponseWriter, status int, data any) error {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    if err := json.NewEncoder(w).Encode(map[string]any{"data": data}); err != nil {
        return fmt.Errorf("encode response: %w", err)
    }
    return nil
}

// Error 由 pkg/httpx 內部呼叫，handler 不直接使用（一律以 return err 拋出）
func Error(w http.ResponseWriter, status int, body any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(map[string]any{"error": body})
}
```

**呼叫慣例**：
- Handler 成功路徑 → `return response.JSON(w, status, data)`
- Handler 錯誤路徑 → `return err`（讓 `pkg/httpx` 處理）
- `response.Error` 屬於底層 API，**只有 `pkg/httpx.WriteError` 呼叫**；其他層（包含 Recoverer）一律走 `httpx.WriteError` 確保格式一致

---

## 為什麼不能用一般 middleware

Go 的 `http.Handler.ServeHTTP` 沒有 error 回傳值，真正的 middleware 無法攔截下游 handler 的 error。業界（Mat Ryer、go-kit、Stripe 內部）的標準做法是定義一個**自訂 HandlerFunc 型別**，handler 改回傳 `error`，由型別的 `ServeHTTP` 統一寫入。效果與 middleware 相同，但完全符合 `http.Handler` 契約，與 chi 路由相容。

---

## httpx 設計

```go
// pkg/httpx/handler.go

// Func 是專案統一的 handler 簽章
type Func func(w http.ResponseWriter, r *http.Request) error

// 實作 http.Handler — 直接註冊到 chi 路由
func (f Func) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := f(w, r); err != nil {
        WriteError(w, r, err)
    }
}

// WriteError 是整個系統錯誤回應的唯一出口
func WriteError(w http.ResponseWriter, r *http.Request, err error) {
    log := logger.From(r.Context())

    var appErr *apperror.AppError
    if !errors.As(err, &appErr) {
        // 未分類錯誤：500 + 完整 log
        log.Error("unhandled error", "err", err)
        response.Error(w, http.StatusInternalServerError, errorBody{
            Kind:    apperror.ErrInternal.Kind,
            Message: apperror.ErrInternal.Message,
        })
        return
    }

    // 5xx 視為 server-side 問題，需要 oncall 關注
    if appErr.Code >= 500 {
        log.Error("server error",
            "kind", appErr.Kind,
            "err",  appErr.Unwrap(),
        )
    } else {
        // 4xx 為 client 錯誤，用 Warn 避免噪音
        log.Warn("client error",
            "kind", appErr.Kind,
            "code", appErr.Code,
        )
    }

    response.Error(w, appErr.Code, errorBody{
        Kind:    appErr.Kind,
        Message: appErr.Message,
    })
}

type errorBody struct {
    Kind    string `json:"kind"`
    Message string `json:"message"`
}
```

---

## 路由註冊

```go
// cmd/server/main.go
r := chi.NewRouter()
r.Method("GET",  "/api/users/{id}", httpx.Func(userHandler.GetUser))
r.Method("PUT",  "/api/users/{id}", httpx.Func(userHandler.UpdateUser))
r.Method("POST", "/api/users",      httpx.Func(userHandler.CreateUser))
```

可選的 helper（減少視覺噪音）：

```go
// pkg/httpx/handler.go
func Get(r chi.Router, pattern string, f Func)    { r.Method("GET",    pattern, f) }
func Post(r chi.Router, pattern string, f Func)   { r.Method("POST",   pattern, f) }
func Put(r chi.Router, pattern string, f Func)    { r.Method("PUT",    pattern, f) }
func Delete(r chi.Router, pattern string, f Func) { r.Method("DELETE", pattern, f) }

// 使用
httpx.Get(r, "/api/users/{id}", userHandler.GetUser)
```

---

## 錯誤流向總覽

```
Service / Repository
   │
   │  return apperror.NotFound(...) / apperror.Internal(err)
   ▼
Handler (httpx.Func)
   │
   │  return err   ← 不做 errors.As、不寫 response
   ▼
Func.ServeHTTP
   │
   │  WriteError(w, r, err)
   ▼
errors.As → *AppError
   │           │
   │           ├─ code >= 500 → log.Error + 寫入 500 body
   │           └─ code <  500 → log.Warn  + 寫入對應 status + body
   │
   └─ 非 AppError → log.Error + 500 fallback
```

---

## 與 Recoverer 的職責分工

| 失敗類型 | 來源 | 處理者 |
|---------|------|--------|
| 業務錯誤 / 預期失敗 | handler `return err` | `httpx.Func` / `WriteError` |
| 未預期 panic | nil pointer、第三方 panic | `Recoverer` middleware |
| 請求參數驗證失敗 | `validator.Struct` | 回傳 `*AppError` → `httpx.Func` |

---

## 測試策略

```go
func TestFunc_AppError_寫入對應狀態碼(t *testing.T) {
    h := httpx.Func(func(w http.ResponseWriter, r *http.Request) error {
        return apperror.NotFound("user_not_found", "找不到使用者")
    })

    req := httptest.NewRequest("GET", "/", nil)
    rec := httptest.NewRecorder()
    h.ServeHTTP(rec, req)

    require.Equal(t, 404, rec.Code)
    require.JSONEq(t, `{"error":{"kind":"user_not_found","message":"找不到使用者"}}`, rec.Body.String())
}

func TestFunc_未分類錯誤_回傳500(t *testing.T) {
    h := httpx.Func(func(w http.ResponseWriter, r *http.Request) error {
        return errors.New("boom")
    })
    // ... 驗證 500 + 預設 internal body
}
```
