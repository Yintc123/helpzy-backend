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
    "message": "...",
    "request_id": "4bf92f3577b34da6a3ce929d0e0e4736"
  }
}
```

`request_id`（與 `X-Request-ID` response header 共值）為 oncall debug 的入口；缺 RequestID middleware 時欄位省略。

### response 套件

```go
// pkg/response/response.go

// JSON 寫入成功回應。
// 先 marshal 至 buffer、確認成功後才 WriteHeader——若先 WriteHeader 再 encode，
// encode 失敗時 httpx.WriteError 會再寫一次 status，造成 superfluous WriteHeader
// 警告，且 client 收到「200 + error body」的混合回應。
func JSON(w http.ResponseWriter, status int, data any) error {
    buf, err := json.Marshal(map[string]any{"data": data})
    if err != nil {
        return fmt.Errorf("marshal response: %w", err)
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    _, _ = w.Write(buf) // header 已送出；網路寫入失敗無法恢復，刻意忽略
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
    rid := logger.RequestIDFrom(r.Context()) // 缺值時為 ""，仍會 serialize 但無實質害處

    var appErr *apperror.AppError
    if !errors.As(err, &appErr) {
        // 未分類錯誤：500 + 完整 log
        log.Error("unhandled error", "err", err)
        response.Error(w, http.StatusInternalServerError, errorBody{
            Kind:      apperror.ErrInternal.Kind,
            Message:   apperror.ErrInternal.Message,
            RequestID: rid,
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
        Kind:      appErr.Kind,
        Message:   appErr.Message,
        RequestID: rid,
    })
}

type errorBody struct {
    Kind      string `json:"kind"`
    Message   string `json:"message"`
    RequestID string `json:"request_id,omitempty"` // 使用者回報問題時引用，oncall 直接以此 ID 查 trace / log
}
```

> **為什麼回傳 `request_id`**：使用者回報「我遇到錯誤 X」時，最不該發生的事是要求他重現。錯誤 body 帶上 `request_id`（= trace_id，見 [16-observability.md](./16-observability.md)）後，oncall 可直接用該 ID 在 log / trace 兩端定位。`omitempty` 避免 RequestID middleware 未掛時序列化空字串。

---

## DecodeJSON helper

統一處理 JSON 解碼 + body size 超限——配合 `BodyLimit` middleware（見 [09-middleware.md](./09-middleware.md#bodylimit-middleware)）使用。

```go
// pkg/httpx/decode.go
func DecodeJSON(r *http.Request, dst any) error {
    if err := json.NewDecoder(r.Body).Decode(dst); err != nil {
        var maxBytesErr *http.MaxBytesError
        if errors.As(err, &maxBytesErr) {
            return &apperror.AppError{
                Code:    http.StatusRequestEntityTooLarge,
                Kind:    "payload_too_large",
                Message: "請求內容過大",
                Err:     err,
            }
        }
        return apperror.BadRequest("invalid_json", "請求格式錯誤").Wrap(err)
    }
    return nil
}
```

### Handler 使用

```go
func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) error {
    var req UpdateUserRequest
    if err := httpx.DecodeJSON(r, &req); err != nil {
        return err  // 自動處理 413 / 400
    }
    if err := validator.Struct(req); err != nil {
        return err
    }
    // ...
}
```

> 取代各 handler 各自寫 `json.NewDecoder(r.Body).Decode(&req)`——DRY、`MaxBytesError` 一次處理。

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
