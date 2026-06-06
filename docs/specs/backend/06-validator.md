# 06 — Validator 請求參數驗證模組

## 目的

統一封裝 `github.com/go-playground/validator/v10`，為 handler 提供一致的請求參數驗證入口，並將驗證錯誤轉換成 `apperror.AppError`，使回應格式與其它錯誤一致。

## 設計原則

- 驗證規則透過 struct tag 宣告於 handler 的 Request DTO
- Handler 統一呼叫 `validator.Struct(&req)`，不自行組裝欄位錯誤
- 驗證錯誤一律對應 `400 Bad Request`，並回傳具體欄位訊息
- 自訂規則集中註冊於 `pkg/validator`，避免散落各 handler
- `validator.Struct` 不需要 `ctx`（純語法檢查，不打 I/O）

---

## 封裝

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

---

## 自訂規則範例

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

---

## Handler 使用方式

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

---

## 常用 tag 參考

| tag | 用途 |
|-----|------|
| `required` | 必填，零值視為缺失 |
| `omitempty` | 零值時略過後續規則 |
| `email` | Email 格式 |
| `min` / `max` | 字串長度、數值大小、陣列元素數量上下限 |
| `oneof=a b c` | 列舉值限制 |
| `uuid` | UUID 格式 |
| `password` | 自訂規則（見上） |

---

## 初始化時機

`main.go` 啟動時呼叫一次 `validator.Init()`，所有 handler 共用同一個 `*validator.Validate` 實例（thread-safe）。

---

## 測試策略

- 對自訂規則（如 `passwordRule`）撰寫 table-driven test
- Handler 測試中以非法請求觸發 `Struct()`，斷言回應為 `400` 且 body 含對應欄位訊息
