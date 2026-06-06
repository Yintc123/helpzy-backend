# Backend 基礎架構規格書

## 相關規格文件

- [API 規格](./api-spec.md) — 路由、端點 request/response
- [資料庫規格](./db-spec.md) — Schema、Migration、sqlc
- [JWT 驗證規格](./jwt-auth-spec.md) — Token 結構、Middleware

---

## 技術棧

| 項目 | 技術 | 說明 |
|------|------|------|
| 語言 | Go 1.23 | |
| HTTP Router | `chi` v5 | 輕量、stdlib 相容、Clean Architecture 友善 |
| 資料庫 | PostgreSQL 16 | 主要資料儲存 |
| DB 驅動 | `pgx` v5 | PostgreSQL 官方推薦驅動，支援連線池 |
| ORM | `sqlc` | SQL → 型別安全 Go 程式碼生成 |
| 快取 | Redis 7 | Session JWT 儲存、快取層 |
| Redis 客戶端 | `go-redis` v9 | |
| JWT | `golang-jwt/jwt` v5 | 驗證 BFF 傳入的 JWT |
| 容器化 | Docker + Docker Compose | |

---

## 架構：Clean Architecture

```
┌─────────────────────────────────────────┐
│           Frameworks & Drivers          │  ← chi、pgx、sqlc、go-redis、Docker
├─────────────────────────────────────────┤
│          Interface Adapters             │  ← handlers、repository 實作
├─────────────────────────────────────────┤
│              Use Cases                  │  ← services（商業邏輯）
├─────────────────────────────────────────┤
│               Entities                  │  ← models（純 Go struct）
└─────────────────────────────────────────┘
```

**原則：**
- 內層不依賴外層
- 每一層透過 interface 溝通
- 框架型別（chi、pgx、sqlc 生成碼）只出現在最外層
- `context.Context` 從 handler 貫穿至 repository

---

## 專案結構

```
backend/
├── cmd/
│   └── server/
│       └── main.go               # 進入點：組裝依賴、啟動伺服器、優雅關閉
├── internal/
│   ├── config/                   # 設定管理：env 載入、helper、驗證
│   │   ├── config.go
│   │   └── config_test.go
│   ├── handlers/                 # Interface Adapters：處理 HTTP 請求與回應
│   │   ├── user.go
│   │   ├── user_test.go
│   │   ├── health.go             # /healthz、/readyz 端點
│   │   └── health_test.go
│   ├── services/                 # Use Cases：商業邏輯
│   │   ├── user.go
│   │   └── user_test.go
│   ├── repository/               # Interface Adapters：包裝 sqlc，實作 service 所需的 interface
│   │   ├── user.go
│   │   └── user_test.go
│   └── models/                   # Entities：純業務用資料結構（與 sqlc 生成的 struct 分離）
│       └── user.go
├── db/
│   ├── migrations/               # SQL Migration 檔案
│   │   ├── 000001_create_users.up.sql
│   │   └── 000001_create_users.down.sql
│   ├── queries/                  # sqlc 來源：SQL query 定義檔
│   │   └── user.sql
│   └── sqlc/                     # sqlc 自動生成（禁止手動修改）
│       ├── db.go
│       ├── models.go
│       └── user.sql.go
├── pkg/
│   ├── middleware/               # HTTP Middleware
│   │   ├── jwt.go
│   │   ├── cors.go
│   │   ├── logger.go
│   │   ├── requestid.go
│   │   ├── recoverer.go
│   │   └── timeout.go
│   ├── auth/
│   │   └── jwt.go               # JWT 解析與驗證（純函式）
│   ├── database/
│   │   ├── postgres.go          # PostgreSQL 連線池初始化
│   │   ├── tx.go                # TxManager：跨 repository 共用交易
│   │   └── tx_test.go
│   ├── cache/
│   │   └── redis.go             # Redis 連線初始化
│   ├── logger/
│   │   ├── logger.go            # 結構化 logger 初始化與 context 注入
│   │   └── logger_test.go
│   ├── validator/
│   │   ├── validator.go         # validator/v10 封裝與錯誤轉換
│   │   └── validator_test.go
│   ├── apperror/
│   │   ├── error.go             # 統一錯誤型別與 constructors
│   │   └── error_test.go
│   ├── httpx/
│   │   ├── handler.go           # 自訂 HandlerFunc：回傳 error → 中央寫入器
│   │   └── handler_test.go
│   └── response/
│       └── response.go          # 統一回應格式
├── sqlc.yaml                     # sqlc 設定檔
├── configs/
│   └── .env.example
├── docs/
├── Dockerfile
└── go.mod
```

---

## API 路由

> 詳見 [API 規格書](./api-spec.md)。

---

## Middleware 規格

### 套用順序

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

### RequestID Middleware

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

### Logger Middleware

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

### CORS Middleware

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

### Recoverer Middleware

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
- 「業務錯誤」由 handler 以 `return err` 拋出，交給 [`pkg/httpx`](#http-錯誤處理層httpx) 中央寫入器處理
- 兩者互不重疊；handler 內禁止用 `panic` 表達業務錯誤

---

### JWT Middleware

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
- `logger.WithUserID` 必須與 `ContextKeyUserID` 一起注入，否則 [Logger 共通欄位](#共通欄位) 表寫的 `user_id` 永遠不會出現

### Timeout Middleware

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

## Logger 可觀測性模組

### 目的

提供應用層統一的結構化日誌入口，並支援以 `context.Context` 傳遞 `request_id`、`user_id` 等追蹤欄位，使 handler、service、repository 各層輸出的日誌能串成同一條請求鏈。

### 設計原則

- 底層採用標準庫 `log/slog`，輸出 JSON
- Logger 透過 `context` 傳遞，不使用 package-level global（除了 fallback default）
- `request_id` 由 RequestID middleware 注入，logger 自動帶入每筆 log
- 任意層只需 `logger.From(ctx)` 取得 logger，不需重複組裝欄位

### 初始化

```go
// pkg/logger/logger.go
type ctxKey struct{}

// Init 由 main.go 在啟動時呼叫，依環境設定 log level 與格式
func Init(env string) {
    var level slog.Level
    if env == "production" {
        level = slog.LevelInfo
    } else {
        level = slog.LevelDebug
    }
    h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: level})
    slog.SetDefault(slog.New(h))
}
```

### Context 注入與取用

```go
// pkg/logger/logger.go
func WithRequestID(ctx context.Context, rid string) context.Context {
    l := From(ctx).With("request_id", rid)
    return context.WithValue(ctx, ctxKey{}, l)
}

func WithUserID(ctx context.Context, uid string) context.Context {
    l := From(ctx).With("user_id", uid)
    return context.WithValue(ctx, ctxKey{}, l)
}

// From 取出 context 內的 logger，若無則 fallback 至 slog.Default()
func From(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(ctxKey{}).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}
```

### 各層使用方式

```go
// service 層
func (s *UserService) GetUser(ctx context.Context, id string) (*models.User, error) {
    log := logger.From(ctx)
    log.Debug("GetUser called", "id", id)

    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        log.Error("repo.FindByID failed", "err", err, "id", id)
        return nil, err
    }
    return user, nil
}
```

### Log Level 規範

| Level | 用途 |
|-------|------|
| `Debug` | 開發期診斷資訊；production 預設關閉 |
| `Info` | 正常請求、業務事件（建立資源、登入成功等） |
| `Warn` | 可恢復的異常（重試、降級、預期內的失敗） |
| `Error` | 需要 oncall 關注的錯誤（DB 連線失敗、panic recovered） |

### 共通欄位

| 欄位 | 來源 | 說明 |
|------|------|------|
| `time` | slog 自動 | RFC3339 timestamp |
| `level` | slog 自動 | 日誌等級 |
| `msg` | 呼叫參數 | 日誌訊息 |
| `request_id` | RequestID middleware | 串接同一請求所有日誌 |
| `user_id` | JWT middleware | 已驗證使用者 ID |

### 環境變數

| 變數 | 說明 |
|------|------|
| `LOG_LEVEL` | 覆寫預設 log level（`debug` / `info` / `warn` / `error`）；未設定時依 `ENV` 自動判斷 |

### 測試策略

- `From()` / `WithRequestID()` / `WithUserID()` 為純函式，以 table-driven test 覆蓋
- 驗證 logger 寫入欄位時，將 handler 改用 `slog.NewJSONHandler(buf, ...)`，斷言 JSON 輸出包含預期 key

---

## Validator 請求參數驗證模組

### 目的

統一封裝 `github.com/go-playground/validator/v10`，為 handler 提供一致的請求參數驗證入口，並將驗證錯誤轉換成 `apperror.AppError`，使回應格式與其它錯誤一致。

### 設計原則

- 驗證規則透過 struct tag 宣告於 handler 的 Request DTO
- Handler 統一呼叫 `validator.Struct(&req)`，不自行組裝欄位錯誤
- 驗證錯誤一律對應 `400 Bad Request`，並回傳具體欄位訊息
- 自訂規則集中註冊於 `pkg/validator`，避免散落各 handler
- `validator.Struct` 不需要 `ctx`（純語法檢查，不打 I/O）

### 封裝

```go
// pkg/validator/validator.go
var v *validator.Validate

func Init() {
    v = validator.New(validator.WithRequiredStructEnabled())
    // 以 json tag 作為錯誤訊息中的欄位名
    v.RegisterTagNameFunc(func(f reflect.StructField) string {
        name := strings.SplitN(f.Tag.Get("json"), ",", 2)[0]
        if name == "-" {
            return ""
        }
        return name
    })
    // 註冊自訂規則
    _ = v.RegisterValidation("password", passwordRule)
}

// Struct 驗證任意 struct，將 validator 錯誤轉為 apperror.AppError
func Struct(s any) error {
    if err := v.Struct(s); err != nil {
        var verrs validator.ValidationErrors
        if errors.As(err, &verrs) {
            return &apperror.AppError{
                Code:    http.StatusBadRequest,
                Message: formatErrors(verrs),
            }
        }
        return apperror.ErrBadRequest
    }
    return nil
}

func formatErrors(verrs validator.ValidationErrors) string {
    msgs := make([]string, 0, len(verrs))
    for _, fe := range verrs {
        msgs = append(msgs, fmt.Sprintf("%s: %s", fe.Field(), ruleMessage(fe)))
    }
    return strings.Join(msgs, "; ")
}

func ruleMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required": return "為必填"
    case "email":    return "格式錯誤"
    case "min":      return fmt.Sprintf("最少 %s", fe.Param())
    case "max":      return fmt.Sprintf("最多 %s", fe.Param())
    case "password": return "需包含英數字且長度 8-64"
    default:         return fe.Tag()
    }
}
```

### 自訂規則範例

```go
// pkg/validator/validator.go
func passwordRule(fl validator.FieldLevel) bool {
    s := fl.Field().String()
    if len(s) < 8 || len(s) > 64 {
        return false
    }
    var hasLetter, hasDigit bool
    for _, r := range s {
        switch {
        case unicode.IsLetter(r): hasLetter = true
        case unicode.IsDigit(r):  hasDigit = true
        }
    }
    return hasLetter && hasDigit
}
```

### Handler 使用方式

```go
// internal/handlers/user.go
type UpdateUserRequest struct {
    Email    string `json:"email"    validate:"required,email"`
    Nickname string `json:"nickname" validate:"required,min=2,max=32"`
    Password string `json:"password" validate:"omitempty,password"`
}

func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) error {
    var req UpdateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        return apperror.BadRequest("invalid_json", "請求格式錯誤")
    }
    if err := validator.Struct(req); err != nil {
        return err // 已是 *AppError，直接上拋給 httpx.Func
    }

    user, err := h.svc.UpdateUser(r.Context(), chi.URLParam(r, "id"), req.toInput())
    if err != nil {
        return err
    }
    return response.JSON(w, http.StatusOK, user)
}
```

### 常用 tag 參考

| tag | 用途 |
|-----|------|
| `required` | 必填，零值視為缺失 |
| `omitempty` | 零值時略過後續規則 |
| `email` | Email 格式 |
| `min` / `max` | 字串長度、數值大小、陣列元素數量上下限 |
| `oneof=a b c` | 列舉值限制 |
| `uuid` | UUID 格式 |
| `password` | 自訂規則（見上） |

### 初始化時機

`main.go` 啟動時呼叫一次 `validator.Init()`，所有 handler 共用同一個 `*validator.Validate` 實例（thread-safe）。

### 測試策略

- 對自訂規則（如 `passwordRule`）撰寫 table-driven test
- Handler 測試中以非法請求觸發 `Struct()`，斷言回應為 `400` 且 body 含對應欄位訊息

---

## Context 規格

### Context Keys

```go
type contextKey string

const (
    ContextKeyUserID contextKey = "userId"
    ContextKeyRole   contextKey = "role"
)
```

### Handler 取用方式

> Handler 簽章為 `func(w, r) error`，詳見 [HTTP 錯誤處理層](#http-錯誤處理層httpx)。

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) error {
    userID := r.Context().Value(ContextKeyUserID).(string)
    id     := chi.URLParam(r, "id")
    // ...
    return response.JSON(w, http.StatusOK, user)
}
```

### Context 傳遞規則

- Handler → Service → Repository 全程傳遞 `ctx context.Context`
- Repository 所有 DB 查詢都傳入 `ctx`，確保 timeout / cancel 能向下傳播
- 禁止在任何層儲存或複製 context

```go
// repository 層範例
func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    row := r.db.QueryRow(ctx, "SELECT id, email FROM users WHERE id = $1", id)
    // ctx 傳入 DB，client 斷線或 timeout 自動取消查詢
}
```

---

## 分層規格

### Interface 定義

每層透過 interface 溝通，實作在各自的檔案：

```go
// internal/services/user.go
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*models.User, error)
    Update(ctx context.Context, user *models.User) error
}

type UserService struct {
    repo UserRepository
}

// internal/handlers/user.go
type UserServiceInterface interface {
    GetUser(ctx context.Context, id string) (*models.User, error)
    UpdateUser(ctx context.Context, id string, input UpdateUserInput) (*models.User, error)
}

type UserHandler struct {
    svc UserServiceInterface
}
```

### Dependency Injection（在 main.go 組裝）

依賴一律由 `main.go` 由外而內手動組裝（不使用 wire / fx 等框架），順序：

```
config → logger → validator → pool / redis → txMgr → repo → service → handler → router
```

完整 `main.go` 範例見 [啟動與優雅關閉](#啟動與優雅關閉) 章節。

---

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
  "error": "描述訊息"
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

## 統一錯誤處理

### 設計目標

- 單一錯誤型別 `AppError` 同時承載 **HTTP 狀態碼**、**機器可讀代碼**、**使用者訊息**、**原始錯誤**
- 各層只負責「拋」錯誤（`return err`），不負責「寫」HTTP 回應
- 寫入 HTTP 回應的職責集中於 `pkg/httpx`（見下節）
- 支援 `errors.Is` / `errors.As` 標準操作，便於分支判斷與測試

### AppError 型別

```go
// pkg/apperror/error.go
type AppError struct {
    Code    int    // HTTP status（400 / 401 / 404 / 409 / 500 ...）
    Kind    string // 機器可讀代碼，e.g. "user_not_found"、"invalid_credentials"
    Message string // 給 client 看的訊息（可 i18n）
    Err     error  // 原始錯誤（僅記錄在 log，不外洩給 client）
}

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %s: %v", e.Kind, e.Message, e.Err)
    }
    return fmt.Sprintf("%s: %s", e.Kind, e.Message)
}

func (e *AppError) Unwrap() error { return e.Err }

// 用於 errors.Is(err, ErrXxx) — 比對 Kind
func (e *AppError) Is(target error) bool {
    var t *AppError
    if !errors.As(target, &t) {
        return false
    }
    return e.Kind == t.Kind
}
```

### Sentinel 錯誤（提供 errors.Is 比對）

```go
var (
    ErrNotFound     = &AppError{Code: 404, Kind: "not_found",     Message: "資源不存在"}
    ErrUnauthorized = &AppError{Code: 401, Kind: "unauthorized",  Message: "未授權"}
    ErrForbidden    = &AppError{Code: 403, Kind: "forbidden",     Message: "權限不足"}
    ErrBadRequest   = &AppError{Code: 400, Kind: "bad_request",   Message: "請求格式錯誤"}
    ErrConflict     = &AppError{Code: 409, Kind: "conflict",      Message: "資源衝突"}
    ErrInternal     = &AppError{Code: 500, Kind: "internal",      Message: "內部錯誤"}
)
```

### Constructors（建議優先使用）

Constructors 讓 service 層能附上具體的 `Kind` 與 `Message`，並 wrap 原始錯誤：

```go
func NotFound(kind, msg string) *AppError {
    return &AppError{Code: 404, Kind: kind, Message: msg}
}

func BadRequest(kind, msg string) *AppError {
    return &AppError{Code: 400, Kind: kind, Message: msg}
}

func Unauthorized(kind, msg string) *AppError {
    return &AppError{Code: 401, Kind: kind, Message: msg}
}

func Forbidden(kind, msg string) *AppError {
    return &AppError{Code: 403, Kind: kind, Message: msg}
}

func Conflict(kind, msg string) *AppError {
    return &AppError{Code: 409, Kind: kind, Message: msg}
}

// Internal wrap 任意底層錯誤；Message 對外不外洩細節
func Internal(err error) *AppError {
    return &AppError{Code: 500, Kind: "internal", Message: "內部錯誤", Err: err}
}

// Wrap 附加原始錯誤至既有 AppError（保留 Code / Kind / Message）
func (e *AppError) Wrap(err error) *AppError {
    cp := *e
    cp.Err = err
    return &cp
}
```

### 各層拋錯範例

```go
// repository 層：底層錯誤一律 wrap
func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    row, err := r.q.GetUser(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, apperror.NotFound("user_not_found", "找不到此使用者")
    }
    if err != nil {
        return nil, apperror.Internal(err) // DB 連線錯誤等
    }
    return toModel(row), nil
}

// service 層：業務規則錯誤
func (s *UserService) ChangePassword(ctx context.Context, id, old, new string) error {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return err // 直接上拋，不重新包裝
    }
    if !checkPassword(user, old) {
        return apperror.Unauthorized("invalid_credentials", "舊密碼錯誤")
    }
    // ...
}

// handler 層：不再有 errors.As + response.Error 樣板
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) error {
    user, err := h.svc.GetUser(r.Context(), chi.URLParam(r, "id"))
    if err != nil {
        return err
    }
    return response.JSON(w, http.StatusOK, user)
}
```

### 錯誤回應格式

```json
{
  "error": {
    "kind": "user_not_found",
    "message": "找不到此使用者"
  }
}
```

`kind` 給前端做分支判斷（i18n、UI 提示），`message` 為預設可顯示文字。原始 `Err` 不外洩。

---

## HTTP 錯誤處理層（httpx）

### 為什麼不能用一般 middleware

Go 的 `http.Handler.ServeHTTP` 沒有 error 回傳值，真正的 middleware 無法攔截下游 handler 的 error。業界（Mat Ryer、go-kit、Stripe 內部）的標準做法是定義一個**自訂 HandlerFunc 型別**，handler 改回傳 `error`，由型別的 `ServeHTTP` 統一寫入。效果與 middleware 相同，但完全符合 `http.Handler` 契約，與 chi 路由相容。

### 設計

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

### 路由註冊

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

### 錯誤流向總覽

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

### 與 Recoverer 的職責分工

| 失敗類型 | 來源 | 處理者 |
|---------|------|--------|
| 業務錯誤 / 預期失敗 | handler `return err` | `httpx.Func` / `WriteError` |
| 未預期 panic | nil pointer、第三方 panic | `Recoverer` middleware |
| 請求參數驗證失敗 | `validator.Struct` | 回傳 `*AppError` → `httpx.Func` |

### 測試策略

`pkg/httpx` 的測試聚焦於「錯誤 → HTTP 回應」映射：

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

---

## sqlc

> 詳見 [資料庫規格書](./db-spec.md)。

---

## 設定管理（Config）

### 目的

集中管理所有 env，提供型別安全的 `Config` struct，並在啟動時統一驗證。任何模組需要設定一律由 `main.go` 從 `Config` 傳入，**禁止散落呼叫 `os.Getenv`**。

### 設計原則

- **三種環境**：`local` / `production` / `test`，由 `LoadConfig(env)` 外部傳入
- 對應載入 `.env.local` / `.env` / `.env.test`（用 `joho/godotenv`）
- **手寫 helpers**（`getEnv` 系列），不使用第三方 env-tag 框架——保留簡單、顯式、易讀
- **嚴格 fail-fast**：必填缺漏、格式錯誤（int / duration parse 失敗）、跨欄位驗證失敗 → 一律 `return error` 讓 `main.go` 退出，不靜默落回預設值
- **只放「會隨環境改變的設定」**：應用層常數（如服務名、Driver、固定列舉清單）放 `const` 或 `internal/constants`，**禁止混入 `Config`**——避免讓人誤以為可調整
- **敏感欄位自動遮蔽**：`Password` / `Secret` / 含密碼的 URL 透過 `String()` 方法在 log 中印 `***`
- 不可變：載入後不再修改；測試以建構函式注入假值
- `normalizeEnv()` 容錯不合法輸入（落回 `local`）

### Config Struct

```go
// internal/config/config.go
type Config struct {
    App   AppConfig
    DB    DBConfig
    Redis RedisConfig
    JWT   JWTConfig
}

type AppConfig struct {
    Env             string        // local / production / test
    Port            int
    LogLevel        string
    AllowedOrigin   string
    ShutdownTimeout time.Duration
}

type DBConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    Name     string
    SSLMode  string
}

type RedisConfig struct {
    URL string
}

type JWTConfig struct {
    Secret string
}
```

### Env Helpers

```go
// internal/config/config.go

// 必填：缺漏即 return error
func getEnv(key string) (string, error) {
    v := os.Getenv(key)
    if v == "" {
        return "", fmt.Errorf("missing required env: %s", key)
    }
    return v, nil
}

// 可選字串
func getEnvWithDefault(key, defaultValue string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return defaultValue
}

// 可選整數；解析失敗即 error（不靜默落回 default）
func getEnvInt(key string, defaultValue int) (int, error) {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue, nil
    }
    v, err := strconv.Atoi(raw)
    if err != nil {
        return 0, fmt.Errorf("env %s: invalid int %q: %w", key, raw, err)
    }
    return v, nil
}

// 可選 Duration；解析失敗即 error
func getEnvDuration(key string, defaultValue time.Duration) (time.Duration, error) {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue, nil
    }
    v, err := time.ParseDuration(raw)
    if err != nil {
        return 0, fmt.Errorf("env %s: invalid duration %q: %w", key, raw, err)
    }
    return v, nil
}

func normalizeEnv(env string) string {
    switch strings.ToLower(env) {
    case "production", "local", "test":
        return strings.ToLower(env)
    default:
        return "local"
    }
}
```

### LoadConfig

```go
// internal/config/config.go

func LoadConfig(env string) (*Config, error) {
    env = normalizeEnv(env)

    // 1) 依環境載入對應 .env
    switch env {
    case "production":
        _ = godotenv.Load(".env") // 容器內通常由 orchestrator 注入，找不到也 OK
    case "local":
        if err := godotenv.Load(".env.local"); err != nil {
            slog.Warn(".env.local not loaded", "err", err)
        }
    case "test":
        if err := godotenv.Load(".env.test"); err != nil {
            return nil, fmt.Errorf("load .env.test: %w", err)
        }
    }

    // 2) 必填字串
    dbHost,        err := getEnv("DB_HOST");        if err != nil { return nil, err }
    dbUser,        err := getEnv("DB_USER");        if err != nil { return nil, err }
    dbPassword,    err := getEnv("DB_PASSWORD");    if err != nil { return nil, err }
    redisURL,      err := getEnv("REDIS_URL");      if err != nil { return nil, err }
    jwtSecret,     err := getEnv("JWT_SECRET");     if err != nil { return nil, err }
    allowedOrigin, err := getEnv("ALLOWED_ORIGIN"); if err != nil { return nil, err }

    // 3) 型別 envs（解析失敗即 fail-fast）
    dbPort, err := getEnvInt("DB_PORT", 5432)
    if err != nil { return nil, err }

    shutdownTimeout, err := getEnvDuration("SHUTDOWN_TIMEOUT", 15*time.Second)
    if err != nil { return nil, err }

    defaultPort := 8080
    if env == "production" {
        defaultPort = 80
    }
    appPort, err := getEnvInt("PORT", defaultPort)
    if err != nil { return nil, err }

    // 4) 組裝
    cfg := &Config{
        App: AppConfig{
            Env:             env,
            Port:            appPort,
            LogLevel:        getEnvWithDefault("LOG_LEVEL", ""),
            AllowedOrigin:   allowedOrigin,
            ShutdownTimeout: shutdownTimeout,
        },
        DB: DBConfig{
            Host:     dbHost,
            Port:     dbPort,
            User:     dbUser,
            Password: dbPassword,
            Name:     getEnvWithDefault("DB_NAME", "helpzy"),
            SSLMode:  getEnvWithDefault("DB_SSLMODE", "disable"),
        },
        Redis: RedisConfig{URL: redisURL},
        JWT:   JWTConfig{Secret: jwtSecret},
    }

    // 5) 跨欄位驗證
    if err := cfg.Validate(); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

### Validate（跨欄位 / 環境相依規則）

單一欄位的格式 / 必填檢查放在 `getEnv*`；**跨欄位、與環境相關的規則**集中於 `Validate()`，於 `LoadConfig` 最後一步呼叫：

```go
// internal/config/config.go
func (c *Config) Validate() error {
    if len(c.JWT.Secret) < 32 {
        return errors.New("JWT_SECRET must be at least 32 chars")
    }

    if c.App.Env == "production" {
        if c.DB.SSLMode == "disable" {
            return errors.New("DB_SSLMODE must not be 'disable' in production")
        }
        if c.DB.Password == "" {
            return errors.New("DB_PASSWORD must not be empty in production")
        }
        if strings.HasPrefix(c.Redis.URL, "redis://") &&
           !strings.Contains(c.Redis.URL, "@") {
            // 提醒：production Redis URL 通常會有認證
            return errors.New("REDIS_URL appears to have no credentials in production")
        }
    }

    return nil
}
```

**規則放這裡 vs. 放 `getEnv*` 的判準**：
- 「單獨檢查就能判定」（必填、格式）→ `getEnv*` 內
- 「需要看其他欄位 / 環境」→ `Validate()`

### 敏感欄位遮蔽（String 方法）

`Password` / `Secret` / 含密碼的 URL 一旦被 `slog` 之類的記錄器序列化會直接外洩。為各 sub-config 實作 `String()` 統一回傳遮蔽版本：

```go
// internal/config/config.go
func (c DBConfig) String() string {
    return fmt.Sprintf("DBConfig{Host:%s Port:%d User:%s Password:*** Name:%s SSLMode:%s}",
        c.Host, c.Port, c.User, c.Name, c.SSLMode)
}

func (c JWTConfig) String() string {
    return "JWTConfig{Secret:***}"
}

// Redis URL 可能為 redis://user:pass@host:6379；只遮蔽 userinfo 部分
func (c RedisConfig) String() string {
    u, err := url.Parse(c.URL)
    if err != nil {
        return "RedisConfig{URL:***}"
    }
    if u.User != nil {
        u.User = url.UserPassword(u.User.Username(), "***")
    }
    return fmt.Sprintf("RedisConfig{URL:%s}", u.String())
}
```

實作 `fmt.Stringer` 後，`slog.Info("config", "cfg", cfg)` 與 `fmt.Printf("%v", cfg)` 都會自動走遮蔽版本。

### .env 檔規範

| 檔案 | 環境 | 是否進版控 |
|------|------|-----------|
| `.env.local` | local 開發 | ✗（`.gitignore`） |
| `.env` | production（容器內注入） | ✗ |
| `.env.test` | 整合測試 | ✓ 提供範本，但不含真實密碼 |
| `configs/.env.example` | 範本，列出所有 env key | ✓ |

### main.go 載入

```go
env := os.Getenv("ENV") // 唯一直接讀 env 的地方
cfg, err := config.LoadConfig(env)
if err != nil {
    slog.Error("config load failed", "err", err)
    os.Exit(1)
}
```

### 測試策略

- `LoadConfig` 以 `t.Setenv` 設定環境變數，斷言：
  - 必填缺漏 → 回傳對應 error message
  - `DB_PORT=abc` / `SHUTDOWN_TIMEOUT=xxx` → 解析錯誤直接 fail-fast，不靜默走 default
  - 可選欄位走 default
  - `production` / `local` Port 預設值差異
- `Validate()` 為純方法，table-driven 覆蓋：
  - JWT secret 長度不足
  - production + `DB_SSLMODE=disable` → 拒絕
  - production + 空 `DB_PASSWORD` → 拒絕
  - local 環境放寬上述規則
- `String()` 斷言 `Password` / `Secret` / Redis 密碼欄位被替換為 `***`，避免 log 外洩回歸
- `normalizeEnv` / `getEnvInt` / `getEnvDuration` 以 table-driven test 覆蓋邊界（空字串、非法格式、合法解析）

---

## 啟動與優雅關閉

### 為什麼必要

部署滾動更新時 k8s / load balancer 會送 `SIGTERM`，若 server 直接 `os.Exit` 會中斷 in-flight 請求、連線池與快取連線未正確收尾。優雅關閉確保：

1. 不再接收新請求（從 LB 摘掉）
2. 等待現有請求完成（在 `ShutdownTimeout` 內）
3. 關閉 DB、Redis 連線

### main.go 完整骨架

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

### 關閉順序

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

### 環境變數

| 變數 | 預設 | 說明 |
|------|------|------|
| `SHUTDOWN_TIMEOUT` | `15s` | 等待 in-flight 請求完成的最長時間；超過則強制中斷 |

---

## 健康檢查（Health Check）

### 端點

| 端點 | 用途 | 檢查項目 |
|------|------|---------|
| `GET /healthz` | Liveness probe | 程序是否存活；**不**檢查外部依賴 |
| `GET /readyz` | Readiness probe | 是否能服務請求；檢查 DB、Redis 連線 |

### 為什麼分兩個

- **Liveness 失敗** → k8s 重啟 pod；不該因為下游短暫故障而連鎖重啟
- **Readiness 失敗** → k8s 從 service 摘除流量，但不重啟；下游恢復後自動加回
- 兩者混用會導致「Redis 抖動 → 全部 pod 同時重啟 → 雪崩」

### 實作

依賴透過 `Pinger` interface 注入，避免 `HealthHandler` 直接耦合 `*pgxpool.Pool` / `*redis.Client`，便於測試以 mock 替換。

```go
// internal/handlers/health.go

// Pinger 抽象任何「能 ping 確認可用」的依賴。
// pgxpool.Pool 原生符合；redis.Client 用 adapter 包一層。
type Pinger interface {
    Ping(ctx context.Context) error
}

type HealthHandler struct {
    db    Pinger
    cache Pinger
}

func NewHealthHandler(db, cache Pinger) *HealthHandler {
    return &HealthHandler{db: db, cache: cache}
}

func (h *HealthHandler) Live(w http.ResponseWriter, r *http.Request) error {
    return response.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func (h *HealthHandler) Ready(w http.ResponseWriter, r *http.Request) error {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    if err := h.db.Ping(ctx); err != nil {
        return apperror.Internal(fmt.Errorf("db ping: %w", err))
    }
    if err := h.cache.Ping(ctx); err != nil {
        return apperror.Internal(fmt.Errorf("redis ping: %w", err))
    }
    return response.JSON(w, http.StatusOK, map[string]string{"status": "ready"})
}
```

### Pinger Adapter

`pgxpool.Pool.Ping(ctx) error` 已符合 `Pinger`。Redis 客戶端 `Ping(ctx) *StatusCmd` 簽章不符，包一層 adapter：

```go
// pkg/cache/redis.go
type RedisPinger struct{ *redis.Client }

func (p RedisPinger) Ping(ctx context.Context) error {
    return p.Client.Ping(ctx).Err()
}
```

main.go 註冊：

```go
healthHandler := handlers.NewHealthHandler(pool, cache.RedisPinger{Client: rdb})
```

### 路由註冊

兩個端點**註冊在 JWT middleware 之外**（k8s 探針不會帶 token）：

```go
httpx.Get(r, "/healthz", healthHandler.Live)
httpx.Get(r, "/readyz",  healthHandler.Ready)
```

### 測試策略

- `Live`：純函式，斷言 200 + `{"status":"ok"}`
- `Ready`：以 mock `DB` / `Cache` interface 分別模擬 ping 成功 / 失敗，斷言狀態碼

---

## PostgreSQL 規格

### 連線池設定

直接接收 `config.DBConfig`（不另定義一份 struct），啟動失敗即 `return error` 讓 `main.go` 退出。

```go
// pkg/database/postgres.go
func New(c config.DBConfig) (*pgxpool.Pool, error) {
    url := fmt.Sprintf(
        "postgres://%s:%s@%s:%d/%s?sslmode=%s",
        c.User, c.Password, c.Host, c.Port, c.Name, c.SSLMode,
    )
    cfg, err := pgxpool.ParseConfig(url)
    if err != nil {
        return nil, fmt.Errorf("parse db url: %w", err)
    }
    cfg.MaxConns        = 20
    cfg.MinConns        = 5
    cfg.MaxConnLifetime = 1 * time.Hour
    cfg.MaxConnIdleTime = 30 * time.Minute

    pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
    if err != nil {
        return nil, fmt.Errorf("create pool: %w", err)
    }
    return pool, nil
}
```

**池參數**（目前 hardcoded，未來如需依環境調整再搬到 Config）：

| 參數 | 值 | 理由 |
|------|-----|------|
| `MaxConns` | 20 | 單實例上限；視 production 流量再調 |
| `MinConns` | 5 | 維持暖連線避免冷啟動 |
| `MaxConnLifetime` | 1h | 避免 DB 端 idle timeout |
| `MaxConnIdleTime` | 30min | 釋放閒置連線 |

> 環境變數見 [環境變數總表](#環境變數)。Migration 規範見 [資料庫規格書](./db-spec.md)。

---

### Transaction Manager（跨 Repository 共用交易）

#### 目的

當一個 service 方法需要跨多個 repository 操作並維持 ACID（例：建立訂單同時扣庫存、發點數），不能讓每個 repo 各自從 pool 借連線——必須共用同一個 `pgx.Tx`。`TxManager` 提供這個能力，同時不讓 `pgx.Tx` 滲漏進 service 層。

#### 設計：以 context 傳遞 tx

採用「ctx 攜帶 tx」模式（業界主流：Uber、Stripe Go 服務常見）：

- Service 呼叫 `txMgr.WithTx(ctx, fn)`，`fn` 收到的 `ctx` 已綁定 tx
- Repository 不感知 tx 存在，內部呼叫 `Executor(ctx)` 取得「tx 或 pool」
- 沒在交易內時自動 fallback 到 pool，正常查詢不受影響

#### 實作

```go
// pkg/database/tx.go
type ctxKey struct{}

type TxManager struct {
    pool *pgxpool.Pool
}

func NewTxManager(pool *pgxpool.Pool) *TxManager {
    return &TxManager{pool: pool}
}

// WithTx 開啟交易，將綁定 tx 的 ctx 傳給 fn。
// fn 回傳非 nil error → rollback；回傳 nil → commit。
// 支援巢狀呼叫：若 ctx 內已有 tx 則直接複用，不重複 BEGIN。
func (m *TxManager) WithTx(ctx context.Context, fn func(ctx context.Context) error) error {
    if _, ok := ctx.Value(ctxKey{}).(pgx.Tx); ok {
        return fn(ctx) // 已在交易內，直接執行
    }

    tx, err := m.pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback(ctx) // commit 成功後 rollback 是 no-op

    ctxWithTx := context.WithValue(ctx, ctxKey{}, tx)
    if err := fn(ctxWithTx); err != nil {
        return err
    }
    return tx.Commit(ctx)
}

// DBTX 對應 sqlc 生成的 executor interface（Query / QueryRow / Exec）
type DBTX interface {
    Exec(context.Context, string, ...any) (pgconn.CommandTag, error)
    Query(context.Context, string, ...any) (pgx.Rows, error)
    QueryRow(context.Context, string, ...any) pgx.Row
}

// Executor 由 repository 呼叫，取得當前 ctx 的執行者
// — 若在交易內回傳 tx，否則回傳 pool。
func Executor(ctx context.Context, pool *pgxpool.Pool) DBTX {
    if tx, ok := ctx.Value(ctxKey{}).(pgx.Tx); ok {
        return tx
    }
    return pool
}
```

#### Repository 使用方式

Repository **不直接持有 pool**，而是每次查詢透過 `Executor(ctx, pool)` 取得執行者：

```go
// internal/repository/user.go
type UserRepo struct {
    pool *pgxpool.Pool
}

func NewUserRepo(pool *pgxpool.Pool) *UserRepo {
    return &UserRepo{pool: pool}
}

func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    q := sqlc.New(database.Executor(ctx, r.pool))
    row, err := q.GetUser(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, apperror.NotFound("user_not_found", "找不到此使用者")
    }
    if err != nil {
        return nil, apperror.Internal(err)
    }
    return toModel(row), nil
}
```

#### Service 使用方式

```go
// internal/services/order.go
type OrderService struct {
    orderRepo     OrderRepository
    inventoryRepo InventoryRepository
    pointRepo     PointRepository
    tx            *database.TxManager
}

func (s *OrderService) PlaceOrder(ctx context.Context, in PlaceOrderInput) error {
    return s.tx.WithTx(ctx, func(ctx context.Context) error {
        if err := s.inventoryRepo.Deduct(ctx, in.ProductID, in.Qty); err != nil {
            return err
        }
        if _, err := s.orderRepo.Create(ctx, in); err != nil {
            return err
        }
        return s.pointRepo.Grant(ctx, in.UserID, calcPoints(in))
    })
}
```

三個 repo 操作共用同一 `pgx.Tx`，任一步 `return err` 整個 rollback。

#### 守則

- Repository **禁止** 直接呼叫 `pool.Begin()` / `pool.Query()`；一律透過 `Executor(ctx, pool)`
- Service 跨多個 repo 的寫入 → 必須包在 `WithTx` 內
- 單純讀取 / 單一 repo 寫入 → 不需 `WithTx`，直接呼叫即可（自動走 pool）
- `WithTx` 內禁止啟動 goroutine 並把同一 ctx 傳出去 — tx 不是 thread-safe

#### 測試策略

- 以 `testcontainers` 啟動真實 Postgres
- 測案：
  - `WithTx` 中 fn 回傳 error → 資料未變更（rollback 生效）
  - `WithTx` 中 fn 回傳 nil → 資料寫入（commit 生效）
  - 巢狀 `WithTx` → 不重複 BEGIN，外層 commit 才真正寫入
  - 未呼叫 `WithTx` 的查詢 → 走 pool，正常運作

---

## Redis 規格

### 連線設定

```go
// pkg/cache/redis.go
func New(url string) (*redis.Client, error) {
    opt, err := redis.ParseURL(url)
    if err != nil {
        return nil, fmt.Errorf("parse redis url: %w", err)
    }
    return redis.NewClient(opt), nil
}
```

> 環境變數見 [環境變數總表](#環境變數)。

---

## 環境變數

> 統一由 [`internal/config`](#設定管理config) 解析、驗證並注入 `Config` struct，禁止散落使用 `os.Getenv`（`main.go` 讀取 `ENV` 為唯一例外）。

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `ENV` | ✗ | `local` | 執行環境（`local` / `production` / `test`） |
| `PORT` | ✗ | `8080`（prod `80`） | 伺服器埠號 |
| `LOG_LEVEL` | ✗ | 依 `ENV` | 覆寫 log 等級（`debug` / `info` / `warn` / `error`） |
| `SHUTDOWN_TIMEOUT` | ✗ | `15s` | 優雅關閉等待 in-flight 請求的最長時間（`time.Duration` 格式） |
| `ALLOWED_ORIGIN` | ✓ | — | 允許的 CORS Origin（BFF 位址） |
| `DB_HOST` | ✓ | — | PostgreSQL host |
| `DB_PORT` | ✗ | `5432` | PostgreSQL port |
| `DB_USER` | ✓ | — | PostgreSQL 使用者 |
| `DB_PASSWORD` | ✓ | — | PostgreSQL 密碼 |
| `DB_NAME` | ✗ | `helpzy` | PostgreSQL 資料庫名稱 |
| `DB_SSLMODE` | ✗ | `disable` | PostgreSQL SSL 模式 |
| `REDIS_URL` | ✓ | — | Redis 連線字串 |
| `JWT_SECRET` | ✓ | — | JWT 簽署金鑰（與 BFF 共享，min 32 chars） |

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

## TDD 測試策略

### 各層測試方式

| 層 | 測試方式 | 依賴 |
|----|---------|------|
| `pkg/auth` | 單元測試，純函式 | 無 |
| `internal/config` | `t.Setenv` 模擬環境，覆蓋 required / default / 邊界 helper | 無 |
| `pkg/logger` | 單元測試（buffer handler 斷言 JSON 欄位） | 無 |
| `pkg/validator` | 單元測試（table-driven 涵蓋自訂規則） | 無 |
| `pkg/apperror` | 單元測試（`errors.Is` / `Unwrap` / `Wrap` 行為） | 無 |
| `pkg/httpx` | `httptest` 驗證錯誤 → HTTP 回應映射 | 無 |
| `pkg/database/tx` | 整合測試（rollback / commit / 巢狀 / fallback） | PostgreSQL（testcontainers） |
| `pkg/middleware` | `httptest` 模擬 HTTP | 無 |
| `internal/handlers/health` | mock `Pinger` interface 斷言 200/500 | mock |
| `internal/handlers` | `httptest` + mock service | mock |
| `internal/services` | 單元測試 + mock repository / mock TxManager | mock |
| `internal/repository` | 整合測試，連接真實測試 DB | PostgreSQL（testcontainers） |
| `db/queries` | `sqlc vet` 驗證 SQL 語法 | 無 |

### Mock 方式

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

### 測試指令

```bash
go test ./...                    # 所有測試
go test ./internal/handlers/...  # 只跑 handlers
go test ./... -cover             # 覆蓋率
go test ./internal/repository/... -tags integration  # 整合測試
```

---

## 套件清單

```go
// go.mod 預計引入
require (
    github.com/go-chi/chi/v5                  v5.x
    github.com/go-chi/cors                    v1.x   // CORS middleware
    github.com/jackc/pgx/v5                   v5.x
    github.com/redis/go-redis/v9              v9.x
    github.com/golang-jwt/jwt/v5              v5.x
    github.com/go-playground/validator/v10    v10.x  // 請求參數驗證
    github.com/google/uuid                    v1.x   // RequestID 產生
    github.com/joho/godotenv                  v1.x   // 載入 .env 檔
    github.com/stretchr/testify               v1.x   // assert
    github.com/testcontainers/testcontainers-go v0.x // 整合測試
)
```

> 開發工具（sqlc、golang-migrate）安裝詳見 [資料庫規格書](./db-spec.md)。
