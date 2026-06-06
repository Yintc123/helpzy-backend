# Context Key

## 介紹

Context Key 是 Go 語言中搭配 `context.Context` 使用的鍵值傳遞機制。透過 `context.WithValue(ctx, key, value)` 將資料附加到 context 上，再用 `ctx.Value(key)` 在下游取出。

### 為什麼不能直接用 `string` 當 key？

Go 官方明確建議：**不要使用內建型別（如 `string`、`int`）當作 context key**，原因是不同套件可能用相同字面量字串，造成 key 衝突且編譯器無法偵測。

標準做法是定義一個**未匯出的自訂型別**作為 key：

```go
type ctxKey int

const (
    userIDKey ctxKey = iota + 1
    roleKey
    jtiKey
)
```

由於 `ctxKey` 是未匯出型別，即使其他套件定義了同名的 `ctxKey`，在型別系統中也屬於不同型別，**物理上不可能撞 key**。

### 標準存取模式

- **寫入**：`WithXxx(ctx, value) context.Context`（與 stdlib `context.WithValue` 命名一致）
- **讀取**：`XxxFrom(ctx) (value, ok)`（回傳 `(value, ok)` 而非強制斷言，避免 panic）

---

## 這個專案用在哪裡

本專案有兩處集中使用 Context Key，皆採用上述標準模式。

### 1. JWT 認證資訊（`pkg/middleware/context.go`）

JWT middleware 解析 token 後，將使用者資訊注入 context 供下游 handler / service 取用：

| Key | 對應資料 | 來源 |
|------|---------|------|
| `userIDKey` | 使用者 ID（JWT `sub`） | JWT middleware |
| `roleKey` | 使用者角色（JWT `role`） | JWT middleware |
| `jtiKey` | Token 唯一識別碼（JWT `jti`） | JWT middleware |

存取 helper：

```go
// 寫入（middleware 用）
func WithUserID(ctx context.Context, id string) context.Context
func WithRole(ctx context.Context, role string) context.Context
func WithJTI(ctx context.Context, jti string) context.Context

// 讀取（handler / service 用）
func UserIDFrom(ctx context.Context) (string, bool)
func RoleFrom(ctx context.Context) (string, bool)
func JTIFrom(ctx context.Context) (string, bool)
```

Handler 端使用：

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) error {
    userID, ok := middleware.UserIDFrom(r.Context())
    if !ok {
        return apperror.Unauthorized("missing_user_id", "未授權")
    }
    // ...
}
```

詳細規範見 [09-middleware.md](../specs/foundation/09-middleware.md#context-keys-與存取-helper單一來源)。

### 2. 結構化日誌（`pkg/logger/logger.go`）

Logger 透過 context 傳遞，讓每層 log 自動帶上 `request_id` 與 `user_id` 等請求屬性，不需在每個函式簽章顯式傳 logger：

```go
func WithRequestID(ctx context.Context, rid string) context.Context {
    l := From(ctx).With("request_id", rid)
    return context.WithValue(ctx, ctxKey{}, l)
}

func WithUserID(ctx context.Context, uid string) context.Context {
    l := From(ctx).With("user_id", uid)
    return context.WithValue(ctx, ctxKey{}, l)
}

func From(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(ctxKey{}).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}
```

**注意：這裡的 context key 模式跟 middleware 不同。**

- **Context key 只有一個** —— `ctxKey{}`（空結構體）
- **存進 context 的 value 是 `*slog.Logger`**，**不是** `request_id` 或 `user_id` 本身
- `request_id` / `user_id` 是用 `slog.Logger.With(...)` 黏在 logger 上的結構化欄位
- 每次呼叫 `WithRequestID` / `WithUserID` 都會：**從 context 取出舊 logger → 加上新欄位 → 把新 logger 蓋回同一個 `ctxKey{}`**

兩種模式對比：

| | Middleware (`pkg/middleware/context.go`) | Logger (`pkg/logger/logger.go`) |
|------|----------|----------|
| Key 數量 | 多個（`userIDKey` / `roleKey` / `jtiKey`） | 單一（`ctxKey{}`） |
| Value 型別 | `string`（資料本身） | `*slog.Logger`（攜帶欄位的 logger 實例） |
| 取用方式 | `UserIDFrom(ctx) (string, bool)` 直接拿值 | `From(ctx) *slog.Logger` 拿 logger 再 `.Info(...)` |
| 用途 | 在 handler/service 取得請求屬性做邏輯判斷 | 讓每層 log 自動帶上請求上下文 |

Service / Repository 層直接 `logger.From(ctx).Info(...)`，無需手動傳遞 logger。詳細規範見 [03-logger.md](../specs/foundation/03-logger.md)。

---

## 為什麼使用這個技術

### 1. 跨層傳遞請求屬性而不污染函式簽章

`userID`、`role`、`requestID` 等屬性在整個請求生命週期幾乎每層都會用到。若用參數逐層傳遞，會讓所有函式簽章被污染：

```go
// 沒用 context key
func GetUser(userID, role, requestID string, id string) (*User, error)

// 用 context key
func GetUser(ctx context.Context, id string) (*User, error)
```

### 2. 中介層注入後自動跟著請求流動

Middleware 在請求進入時注入一次，整條呼叫鏈（handler → service → repository → logger）都能取用。例如 JWT middleware 注入 `userID` 後，最底層的 SQL 查詢日誌也能自動印出是哪個使用者觸發。

### 3. 型別安全，編譯期就抓錯

- **Key 型別未匯出** → 跨套件型別不同，撞 key 在編譯期被擋下。
- **Key 是常數而非字面量字串** → typo 會直接編譯失敗，不會變成「靜默讀到 nil」的 runtime bug。

### 4. Getter 回傳 `(value, ok)` 避免 panic

若直接用 `ctx.Value(key).(string)` 強制斷言，缺值時會 panic。本專案統一回傳 `(value, ok)`，handler 可在缺值時優雅回 401，不會把整個 server 帶崩。

### 5. 符合 Go 官方與社群最佳實踐

Go 標準函式庫（`net/http`、`database/sql`）以及主流框架（chi、gRPC）都採用相同模式。沿用慣例降低團隊學習成本，也讓第三方中介層能無痛整合。

---

## 注意事項

- **Context 只該用於請求範圍的資料**（request-scoped values），如使用者身份、追蹤 ID。**不該用於應用層的依賴注入**（資料庫連線、設定等應透過建構子注入）。
- **Key 型別與常數一律未匯出**，外部只能透過 `With*` / `*From` 存取，封閉性最高。
- **同一個 Key 不要分散在多個檔案宣告**——本專案統一在 `pkg/middleware/context.go`，避免「同套件兩處宣告 `ctxKey` 編譯衝突」與「不同套件各自宣告造成讀寫不對應」。
