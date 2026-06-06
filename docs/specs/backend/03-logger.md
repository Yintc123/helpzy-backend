# 03 — Logger 可觀測性模組

## 目的

提供應用層統一的結構化日誌入口，並支援以 `context.Context` 傳遞 `request_id`、`user_id` 等追蹤欄位，使 handler、service、repository 各層輸出的日誌能串成同一條請求鏈。

## 設計原則

- 底層採用標準庫 `log/slog`，輸出 JSON
- Logger 透過 `context` 傳遞，不使用 package-level global（除了 fallback default）
- `request_id` 由 RequestID middleware 注入，logger 自動帶入每筆 log
- 任意層只需 `logger.From(ctx)` 取得 logger，不需重複組裝欄位

---

## 初始化

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

---

## Context 注入與取用

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

---

## 各層使用方式

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

---

## Log Level 規範

| Level | 用途 |
|-------|------|
| `Debug` | 開發期診斷資訊；production 預設關閉 |
| `Info` | 正常請求、業務事件（建立資源、登入成功等） |
| `Warn` | 可恢復的異常（重試、降級、預期內的失敗） |
| `Error` | 需要 oncall 關注的錯誤（DB 連線失敗、panic recovered） |

---

## 共通欄位

| 欄位 | 來源 | 說明 |
|------|------|------|
| `time` | slog 自動 | RFC3339 timestamp |
| `level` | slog 自動 | 日誌等級 |
| `msg` | 呼叫參數 | 日誌訊息 |
| `request_id` | RequestID middleware | 串接同一請求所有日誌 |
| `user_id` | JWT middleware | 已驗證使用者 ID |

---

## 環境變數

| 變數 | 說明 |
|------|------|
| `LOG_LEVEL` | 覆寫預設 log level（`debug` / `info` / `warn` / `error`）；未設定時依 `ENV` 自動判斷 |

---

## 測試策略

- `From()` / `WithRequestID()` / `WithUserID()` 為純函式，以 table-driven test 覆蓋
- 驗證 logger 寫入欄位時，將 handler 改用 `slog.NewJSONHandler(buf, ...)`，斷言 JSON 輸出包含預期 key
