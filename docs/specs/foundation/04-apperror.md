# 04 — 統一錯誤處理（AppError）

## 設計目標

- 單一錯誤型別 `AppError` 同時承載 **HTTP 狀態碼**、**機器可讀代碼**、**使用者訊息**、**原始錯誤**
- 各層只負責「拋」錯誤（`return err`），不負責「寫」HTTP 回應
- 寫入 HTTP 回應的職責集中於 `pkg/httpx`（見 [05-httpx.md](./05-httpx.md)）
- 支援 `errors.Is` / `errors.As` 標準操作，便於分支判斷與測試

---

## AppError 型別

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

---

## Sentinel 錯誤（提供 errors.Is 比對）

```go
var (
    ErrNotFound        = &AppError{Code: 404, Kind: "not_found",        Message: "資源不存在"}
    ErrUnauthorized    = &AppError{Code: 401, Kind: "unauthorized",     Message: "未授權"}
    ErrForbidden       = &AppError{Code: 403, Kind: "forbidden",        Message: "權限不足"}
    ErrBadRequest      = &AppError{Code: 400, Kind: "bad_request",      Message: "請求格式錯誤"}
    ErrConflict        = &AppError{Code: 409, Kind: "conflict",         Message: "資源衝突"}
    ErrTooManyRequests = &AppError{Code: 429, Kind: "too_many_requests", Message: "請求過於頻繁"}
    ErrInternal        = &AppError{Code: 500, Kind: "internal",         Message: "內部錯誤"}
)
```

---

## Constructors（建議優先使用）

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

func TooManyRequests(kind, msg string) *AppError {
    return &AppError{Code: 429, Kind: kind, Message: msg}
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

---

## 各層拋錯範例

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

---

## 錯誤回應格式

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

## 測試策略

- 單元測試：`errors.Is` / `Unwrap` / `Wrap` 行為
- `Is()` 比對 Kind 而非記憶體位址，確保跨層比對正確
