# JWT 驗證規格書

## 概覽

Go Backend 透過驗證 JWT 識別請求來源是否為合法的 Next.js BFF。  
JWT 由 BFF 產生並儲存於 Redis，每次請求時由 BFF 從 Redis 取出後放入 `Authorization` Header 轉發。

```
Next.js BFF ──(Authorization: Bearer <JWT>)──► Go Backend
```

---

## JWT 規格

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

```json
{
  "sub": "user-id",
  "role": "user",
  "iss": "helpzy-bff",
  "iat": 1700000000,
  "exp": 1700003600
}
```

| 欄位 | 型別 | 說明 |
|------|------|------|
| `sub` | string | 使用者 ID |
| `role` | string | 使用者角色（如 `user`、`admin`） |
| `iss` | string | 簽發者，固定為 `helpzy-bff`，防止跨服務 token 混用 |
| `iat` | int64 | 簽發時間（Unix timestamp） |
| `exp` | int64 | 到期時間（簽發後 1 小時） |

### 簽章

| 屬性 | 值 |
|------|----|
| 演算法 | HS256 |
| Secret | 環境變數 `JWT_SECRET`（與 BFF 共享，min 32 chars） |

---

## 請求格式

JWT 放置於每個需要驗證的 API 請求的 `Authorization` Header：

```
Authorization: Bearer <token>
```

---

## Middleware 規格

### 職責

`JWTMiddleware` 負責：

1. 從 `Authorization` Header 取出 token
2. 驗證 token 簽章（使用 `JWT_SECRET`）
3. 驗證 `exp`（是否過期）
4. 驗證 `alg` 必須為 `HS256`（防止 `alg: none` 攻擊）
5. 驗證 `iss` 必須為 `helpzy-bff`
6. 將 Payload 解析後注入至 `context.Context`，供後續 Handler 使用

### 套用範圍

| 路由 | 套用 JWTMiddleware |
|------|--------------------|
| `GET /api/health` | 否 |
| 其他所有 `/api/v1/*` | **是** |

### 錯誤回應

| 情況 | HTTP Status | Body |
|------|-------------|------|
| Header 缺少 `Authorization` | 401 | `{"error": "missing token"}` |
| Bearer 格式錯誤 | 401 | `{"error": "invalid token format"}` |
| 簽章驗證失敗 | 401 | `{"error": "invalid token"}` |
| Token 已過期 | 401 | `{"error": "token expired"}` |
| 演算法不符 | 401 | `{"error": "invalid token"}` |
| `iss` 不符 | 401 | `{"error": "invalid token"}` |

> 所有驗證失敗一律回傳 401，不對外說明具體失敗原因，避免資訊洩漏。

---

## Context 注入規格

驗證成功後，將使用者資訊注入 `context.Context`，Handler 透過 context key 取用。

### Context Key

```go
type contextKey string

const (
    ContextKeyUserID contextKey = "userId"
    ContextKeyRole   contextKey = "role"
)
```

### 取用方式（在 Handler 內）

```go
userId := r.Context().Value(ContextKeyUserID).(string)
role   := r.Context().Value(ContextKeyRole).(string)
```

---

## 檔案結構

```
internal/
└── middleware/
    └── jwt.go              # JWTMiddleware 實作
pkg/
└── auth/
    └── jwt.go              # JWT 解析、驗證邏輯（純函式，可單元測試）
configs/
└── .env.example            # JWT_SECRET 設定
```

### 分層說明

- `pkg/auth/jwt.go`：純函式，只負責解析與驗證 JWT，不依賴 HTTP，方便單元測試
- `internal/middleware/jwt.go`：HTTP Middleware，呼叫 `pkg/auth` 的函式，處理 HTTP 層的讀取與錯誤回應

---

## 流程圖

```
Request
  │
  ▼
JWTMiddleware
  │
  ├─ 取出 Authorization Header
  │     └─ 缺少 → 401 missing token
  │
  ├─ 解析 Bearer token
  │     └─ 格式錯誤 → 401 invalid token format
  │
  ├─ 驗證簽章（HS256 + JWT_SECRET）
  │     └─ 失敗 → 401 invalid token
  │
  ├─ 驗證 exp
  │     └─ 已過期 → 401 token expired
  │
  ├─ 驗證 alg == "HS256"
  │     └─ 不符 → 401 invalid token
  │
  ├─ 驗證 iss == "helpzy-bff"
  │     └─ 不符 → 401 invalid token
  │
  ├─ 注入 userId、role 至 context
  │
  └─ 呼叫 next handler
```

---

## 套件選擇

| 套件 | 說明 |
|------|------|
| `github.com/golang-jwt/jwt/v5` | 官方維護的 JWT 套件，支援 HS256 |

---

## 環境變數

| 變數 | 說明 |
|------|------|
| `JWT_SECRET` | JWT 簽署金鑰，與 Next.js BFF 共享（min 32 chars） |

---

## TDD 測試規格

### `pkg/auth/jwt.go` 單元測試

| 測試案例 | 預期結果 |
|----------|---------|
| 有效 JWT | 回傳正確 userId、role |
| 已過期的 JWT | 回傳 `token expired` 錯誤 |
| 簽章錯誤的 JWT | 回傳 `invalid token` 錯誤 |
| 空字串 | 回傳錯誤 |
| alg 為 none 的 JWT | 回傳 `invalid token` 錯誤 |
| `iss` 不符的 JWT | 回傳 `invalid token` 錯誤 |

### `internal/middleware/jwt.go` 整合測試

使用 `net/http/httptest` 模擬 HTTP 請求：

| 測試案例 | 預期 HTTP Status |
|----------|-----------------|
| 有效 JWT | 200（呼叫 next handler） |
| 無 Authorization Header | 401 |
| Bearer 格式錯誤 | 401 |
| 過期 JWT | 401 |
| 簽章錯誤 JWT | 401 |
