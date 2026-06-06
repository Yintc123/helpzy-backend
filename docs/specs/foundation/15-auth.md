# 15 — Auth：JWT 簽發 / 驗證、密碼 hash、Refresh Token、Blacklist

> Token 格式、Claims、TTL 等**跨服務契約**見 [../jwt-auth-spec.md](../jwt-auth-spec.md)。
> 本文件規範 backend 端的所有 auth 基礎建設實作。

## 模組責任

| 子模組 | 路徑 | 責任 |
|--------|------|------|
| Token 簽發 / 驗證 | `pkg/auth/jwt.go` | 純函式：簽發 access token、驗證、產生 refresh token |
| 密碼 hash | `pkg/auth/password.go` | 純函式：bcrypt hash / compare |
| Refresh repository | `internal/repository/refresh.go` | Redis 操作：儲存 / 原子 rotate（含 reuse detection）/ 撤銷 |
| Revoke repository | `internal/repository/revoke.go` | Redis family-level 撤銷：標記 family、查詢是否已撤銷 |
| JWT Middleware | `pkg/middleware/jwt.go` | HTTP middleware：驗證 access token + 查 family 撤銷狀態 + 注入 context |

---

## Token Family 模型（核心安全機制）

Refresh token rotation 配合 **token theft detection**（OAuth 2.0 Security Best Practice [RFC 6819 §5.2.2.3](https://datatracker.ietf.org/doc/html/rfc6819#section-5.2.2.3)）：

| 概念 | 說明 |
|------|------|
| **Family** | 一次「登入 session」的 token 集合；以 `family_id`（UUID）標識 |
| 同 family 內 | 一個 refresh token 不斷 rotation 產生新 token；access token 與 refresh token 共享同一 `family_id` |
| **Reuse Detection** | 已被 rotate 的 refresh token 再次出現 → 視為**該 family 被盜**，立即撤銷整個 family（含所有未過期 access token） |
| 跨 family | 多裝置登入各自獨立 family；單一裝置登出不影響其他裝置 |

### 為什麼 family 取代「single jti blacklist」

過去設計用 `revoked:{jti}` 黑名單只能撤銷單一 access token——但 token 被盜情境是**整條 token chain 都不能信**（攻擊者掌握 refresh 後可無限 rotate）。family 撤銷一次性失效整條 chain，且不需要列舉所有 jti。

### Redis schema

| Key 格式 | Value | TTL | 用途 |
|---------|-------|-----|------|
| `refresh:{token}` | `{user_id}\|{family_id}` | `REFRESH_TTL` | 有效 refresh token |
| `refresh_used:{token}` | `{user_id}\|{family_id}` | `REFRESH_TTL` | 已被 rotate 過的 refresh token（觸發 reuse detection 的證據） |
| `family_revoked:{family_id}` | `"1"` | `REFRESH_TTL` | family 已被撤銷（被盜或登出）|

---

## `pkg/auth/jwt.go`（純函式）

```go
package auth

// Issuer 是所有 access token 的 iss claim。
// 簽發與驗證共用同一常數——避免字串拼錯到 runtime 才被發現，
// 也讓未來改名（如多租戶 / 多服務拆分）只需動一處。
const Issuer = "helpzy-backend"

type AccessClaims struct {
    Sub       string
    Role      string
    Jti       string
    Fid       string    // family_id：用於 family-level revocation，於 token rotation 時保持不變
    ExpiresAt time.Time // 由 ParseAccess 從 exp claim 解出
}

// SignAccess 簽發 access token。
// caller 傳入 sub / role / familyID；jti 內部產生、iss = Issuer、exp = now + ttl。
func SignAccess(sub, role, familyID, secret string, ttl time.Duration) (token, jti string, err error)

// ParseAccess 驗證並解析 access token。
// 驗證項目：簽章、alg == HS256、iss == Issuer、exp 未過期。
// 不查 family revocation——由 middleware 在拿到 claims 後負責。
func ParseAccess(token, secret string) (*AccessClaims, error)

// GenerateRefreshToken 產生 32-byte crypto/rand 隨機字串，base64url 編碼。
// 不簽章、不含 payload——它只是 Redis 的查表 key。
func GenerateRefreshToken() (string, error)
```

### 為什麼 `AccessClaims` 帶 `Fid` 與 `ExpiresAt`

- `Fid`：JWT middleware 收到 access token 後查 `family_revoked:{fid}`——若整個 session 已被撤銷（reuse detected / 登出），即使 jti 沒過期也立即失效。
- `ExpiresAt`：保留作為診斷用途（例：log 中標示「token 還剩多久過期」、metric 化未過期撤銷次數）；family 模型不再需要它計算 BlockJTI TTL。

### Sentinel errors

```go
var (
    ErrTokenExpired       = errors.New("token expired")
    ErrInvalidSignature   = errors.New("invalid signature")
    ErrInvalidAlgorithm   = errors.New("invalid algorithm")
    ErrInvalidIssuer      = errors.New("invalid issuer")
    ErrMalformedToken     = errors.New("malformed token")
)
```

Middleware 拿到 sentinel 後統一轉為 `apperror.Unauthorized(...)`，具體 kind 對應見[錯誤回應對應](#錯誤回應對應)。

---

## `pkg/auth/password.go`（純函式）

```go
package auth

// HashPassword 以 bcrypt 雜湊明文密碼。cost 由 cfg.BcryptCost 傳入（預設 12）。
func HashPassword(plaintext string, cost int) (string, error)

// ComparePassword 比對 hash 與明文。匹配回 nil；不符回 bcrypt.ErrMismatchedHashAndPassword。
// caller 應將回傳 error 一律當作「帳號或密碼錯誤」處理，不細分原因。
func ComparePassword(hash, plaintext string) error
```

> **守則**：登入失敗時，error message 一律回 `"帳號或密碼錯誤"`，**不區分「帳號不存在」vs「密碼錯誤」**——避免帳號列舉攻擊。

---

## Refresh Token Repository

Refresh token 是 Redis 為單一真相，repository 介面定義在 service 層：

```go
// internal/services/auth.go
type RefreshRepository interface {
    // Save 為新 family 初次寫入 refresh token（登入時呼叫）。
    Save(ctx context.Context, token, userID, familyID string) error

    // Rotate 以單一 Lua script 原子完成：
    //   1. 讀 refresh:{old}
    //      - 找到 → 取出 userID|familyID，DEL refresh:{old}，
    //                SET refresh_used:{old} = userID|familyID（TTL = refresh ttl），
    //                SET refresh:{new} = userID|familyID（TTL = refresh ttl），
    //                回傳 (userID, familyID, reused=false, nil)
    //      - 找不到 → 檢查 refresh_used:{old}
    //                  - 存在 → reuse detected，回傳 (userID, familyID, reused=true, nil)
    //                            caller 必須呼叫 RevokeRepository.RevokeFamily(familyID)
    //                  - 不存在 → 純粹無效或過期，回傳 apperror.Unauthorized
    //
    // 整個流程在 Redis 端一次跑完——避免「讀 → 寫」之間的 race 讓兩個並發 rotate 都成功。
    Rotate(ctx context.Context, oldToken, newToken string) (userID, familyID string, reused bool, err error)

    // Revoke 主動撤銷單一 refresh token（登出時呼叫）。
    // 實作：DEL refresh:{token} + SET refresh_used:{token}——
    // 確保「登出後若舊 token 再被重放」仍會觸發 reuse detection。
    Revoke(ctx context.Context, token string) error
}
```

### Constructor 簽章

```go
// internal/repository/refresh.go
func NewRefreshRepo(rdb *redis.Client, ttl time.Duration) *RefreshRepo
```

TTL 是「組態」、不是「每次呼叫變動的資料」——構造時注入一次，方法簽章保持乾淨。

### Rotation 流程（service 層）

`POST /auth/refresh`：

```go
// internal/services/auth.go
func (s *AuthService) Refresh(ctx context.Context, oldRefresh string) (*TokenPair, error) {
    newRefresh, err := auth.GenerateRefreshToken()
    if err != nil { return nil, apperror.Internal(err) }

    userID, familyID, reused, err := s.refreshRepo.Rotate(ctx, oldRefresh, newRefresh)
    if err != nil { return nil, err }
    if reused {
        // 已 rotate 過的 token 再次出現 → 整個 family 被盜
        _ = s.revokeRepo.RevokeFamily(ctx, familyID)
        observability.TokenReusedTotal.Add(ctx, 1)
        logger.From(ctx).Warn("refresh token reuse detected",
            "user_id", userID, "family_id", familyID)
        return nil, apperror.Unauthorized("token_reused", "未授權")
    }

    role, err := s.userRepo.GetRole(ctx, userID)
    if err != nil { return nil, err }

    accessToken, _, err := auth.SignAccess(userID, role, familyID, s.jwtSecret, s.accessTTL)
    if err != nil { return nil, apperror.Internal(err) }

    return &TokenPair{Access: accessToken, Refresh: newRefresh}, nil
}
```

### 為什麼 Rotate 必須原子

若分成 `Lookup → Del → Save` 三步，兩個並發 refresh 請求可能同時通過 `Lookup`，雙雙 `Del` 失敗（但只一個成功）——更糟的是兩邊都拿到 access token 且兩個新 refresh 都有效，family 內出現兩條 chain。Lua script 確保整段在 Redis 內原子完成。

### Save 不收 TTL 的理由

| Repo 方法 | TTL 性質 | 設計 |
|----------|---------|------|
| `Save` / `Rotate` | 系統全域常數（`cfg.JWT.RefreshTTL`） | **Constructor 注入**，方法不收 |
| `RevokeFamily` | 系統全域常數（`cfg.JWT.RefreshTTL`，涵蓋 family 中最長壽命 token） | **Constructor 注入** |

判準：**「值會不會隨呼叫變動」**。會變 → 方法參數；不會變 → constructor。

---

## Revoke Repository（Family-level Revocation）

```go
// internal/services/auth.go
type RevokeRepository interface {
    // RevokeFamily 標記整個 token family 失效。
    // TTL 由 constructor 注入（cfg.JWT.RefreshTTL）——覆蓋 family 中最長壽命的 token。
    RevokeFamily(ctx context.Context, familyID string) error

    // IsFamilyRevoked 由 JWT middleware 在驗證 claims 後查詢。
    IsFamilyRevoked(ctx context.Context, familyID string) (bool, error)
}
```

### Constructor 簽章

```go
func NewRevokeRepo(rdb *redis.Client, ttl time.Duration) *RevokeRepo
```

### 登出流程

`POST /auth/logout`：

```go
func (s *AuthService) Logout(ctx context.Context, claims *auth.AccessClaims, refreshToken string) error {
    if err := s.revokeRepo.RevokeFamily(ctx, claims.Fid); err != nil {
        return err
    }
    return s.refreshRepo.Revoke(ctx, refreshToken)
}
```

1. **`RevokeFamily(claims.Fid)`** — 整個 family 的 access token 立即失效（含本次登出的 token、與任何尚未過期的並發 access token）
2. **`Refresh.Revoke(refreshToken)`** — 將 refresh token 移入 `refresh_used`；若日後被重放仍會觸發 reuse detection

> 多裝置：登入時為每個裝置產生獨立 `family_id`，因此登出單一裝置不影響其他裝置。

### 為什麼不再用 jti blacklist

`BlockJTI` 只能撤銷單一 access token——但任何「真實安全事件」（token 洩漏、可疑活動）需要撤銷的是**整條 token chain**，不是某一張票券。family 撤銷一次性處理 access + refresh，語意更清楚且 Redis key 數量遠少。

---

## JWT Middleware（`pkg/middleware/jwt.go`）

```go
type JWTMiddleware struct {
    secret     string
    revokeRepo RevokeRepository // 查 family 撤銷狀態
}

func NewJWTMiddleware(secret string, revokeRepo RevokeRepository) *JWTMiddleware {
    return &JWTMiddleware{secret: secret, revokeRepo: revokeRepo}
}

// Context keys 與存取 helper（WithUserID / WithRole / WithJTI / WithFamilyID / *From）
// 統一定義於 pkg/middleware/context.go，見 [09-middleware.md](./09-middleware.md#context-keys-與存取-helper單一來源)。
// 本檔案禁止重複宣告 ctxKey 型別。

func (m *JWTMiddleware) Verify(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token, err := extractBearerToken(r)
        if err != nil {
            httpx.WriteError(w, r, err)
            return
        }

        claims, err := auth.ParseAccess(token, m.secret)
        if err != nil {
            httpx.WriteError(w, r,
                apperror.Unauthorized("invalid_token", "未授權").Wrap(err))
            return
        }

        // 查 family 撤銷狀態：登出 / token theft / 主動撤銷都會走到這裡
        revoked, err := m.revokeRepo.IsFamilyRevoked(r.Context(), claims.Fid)
        if err != nil {
            httpx.WriteError(w, r, apperror.Internal(err))
            return
        }
        if revoked {
            httpx.WriteError(w, r,
                apperror.Unauthorized("session_revoked", "未授權"))
            return
        }

        // 注入 context（helpers 來自 pkg/middleware/context.go，見 09-middleware.md）
        ctx := WithUserID(r.Context(), claims.Sub)
        ctx  = WithRole(ctx, claims.Role)
        ctx  = WithJTI(ctx, claims.Jti)
        ctx  = WithFamilyID(ctx, claims.Fid)
        ctx  = logger.WithUserID(ctx, claims.Sub)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### `extractBearerToken`（內部 helper）

從 `Authorization` header 取出 bearer token；不同失敗情境直接回對應 `*AppError`，由 `httpx.WriteError` 統一寫入。

```go
// pkg/middleware/jwt.go
func extractBearerToken(r *http.Request) (string, error) {
    h := r.Header.Get("Authorization")
    if h == "" {
        return "", apperror.Unauthorized("missing_token", "未授權")
    }
    scheme, token, ok := strings.Cut(h, " ")
    if !ok || !strings.EqualFold(scheme, "Bearer") || token == "" {
        return "", apperror.Unauthorized("invalid_token_format", "未授權")
    }
    return token, nil
}
```

#### 邊界行為

| Header | 結果 | Kind |
|--------|------|------|
| `""`（無 Authorization） | error | `missing_token` |
| `"Bearer abc"` | `"abc"` | — |
| `"bearer abc"` / `"BEARER abc"` | `"abc"` | — RFC 6750 規定 scheme 大小寫不敏感 |
| `"Basic xyz"` | error | `invalid_token_format` |
| `"Bearer"`（無空格） | error | `invalid_token_format` |
| `"Bearer "`（空 token） | error | `invalid_token_format` |
| `"  Bearer abc"`（前導空白） | error | `invalid_token_format` |

#### 設計重點

| 設計 | 理由 |
|------|------|
| 用 `strings.Cut` 而非 `SplitN` | Cut 一次切分、語意更清楚（scheme / token / ok） |
| 用 `strings.EqualFold` 比對 `Bearer` | RFC 6750 §2.1 規定 scheme 不分大小寫 |
| 不檢查 token 內容（只檢查結構） | 內容驗證（簽章 / exp / alg）由 `auth.ParseAccess` 負責，職責分明 |
| Helper 為未匯出（小寫開頭） | 只給 JWT middleware 用，避免外部依賴 |
| 回傳 `*AppError` 而非 sentinel error | middleware 拿到後不需 mapping，直接 `httpx.WriteError` |

### 套用範圍

| 路由 | 套用 JWTMiddleware |
|------|--------------------|
| `GET /healthz`、`GET /readyz` | ✗（探針不帶 token） |
| `POST /api/v1/auth/register` / `login` / `refresh` | ✗（這幾個本來就在驗證之前） |
| `POST /api/v1/auth/logout` | ✓（需要知道當前 jti） |
| 其他所有 `/api/v1/*` | ✓ |

---

## 錯誤回應對應

驗證失敗一律回 `401`、對外訊息統一為 `未授權`；`kind` 留給 client 區分大類，具體原因記錄在 server log。

| 情況 | `kind` | HTTP |
|------|--------|------|
| Header 缺 `Authorization` | `missing_token` | 401 |
| Bearer 格式錯誤 | `invalid_token_format` | 401 |
| 簽章驗證失敗 / `alg` 不符 / `iss` 不符 | `invalid_token` | 401 |
| Token 過期 | `token_expired` | 401 |
| Family 已撤銷（登出 / reuse detected） | `session_revoked` | 401 |
| Refresh token 已被 rotate 過再次出現 | `token_reused` | 401 |
| Redis 查詢失敗 | `internal` | 500 |

---

## 與其它規格的關係

| 規格 | 關聯 |
|------|------|
| [04-apperror.md](./04-apperror.md) | 錯誤型別 |
| [05-httpx.md](./05-httpx.md) | `httpx.WriteError` 寫入錯誤回應 |
| [03-logger.md](./03-logger.md) | `logger.WithUserID` 注入 |
| [08-redis.md](./08-redis.md) | Redis 連線（refresh / blacklist 共用此連線） |
| [09-middleware.md](./09-middleware.md) | Middleware 套用順序 |
| [../jwt-auth-spec.md](../jwt-auth-spec.md) | Backend ↔ Frontend 認證契約 |

---

## DI 組裝

```go
// cmd/server/main.go（節錄）
refreshRepo := repository.NewRefreshRepo(rdb, cfg.JWT.RefreshTTL)
revokeRepo  := repository.NewRevokeRepo(rdb)
authSvc     := service.NewAuthService(userRepo, refreshRepo, revokeRepo, cfg.JWT, cfg.BcryptCost)
authHandler := handlers.NewAuthHandler(authSvc)
jwtMw       := middleware.NewJWTMiddleware(cfg.JWT.Secret, revokeRepo)
```

完整 main.go 見 [12-startup.md](./12-startup.md)。

---

## TDD 測試策略

### `pkg/auth/jwt.go`

| 案例 | 預期 |
|------|------|
| `SignAccess` + `ParseAccess` round-trip | claims 一致、jti 非空、`Fid` 等於傳入值、`ExpiresAt` 約等於 now + ttl（±1s 容忍） |
| `SignAccess` 自動帶入 `iss = Issuer` | ParseAccess 不報 `ErrInvalidIssuer` |
| 過期 token | `ErrTokenExpired` |
| 簽章錯誤 | `ErrInvalidSignature` |
| `alg: none` | `ErrInvalidAlgorithm` |
| `iss` 被竄改為其他值 | `ErrInvalidIssuer` |
| 缺 `fid` claim | `ErrMalformedToken` |
| `GenerateRefreshToken` | 長度固定、兩次呼叫不重複 |

### `pkg/auth/password.go`

| 案例 | 預期 |
|------|------|
| `HashPassword` + `ComparePassword` round-trip | 不報錯 |
| 錯誤密碼 | `bcrypt.ErrMismatchedHashAndPassword` |
| 相同明文兩次 hash 結果不同（salt） | OK |

### `pkg/middleware/jwt.go`

#### `extractBearerToken`（純函式，table-driven）

| 輸入 | 預期 token | 預期 error kind |
|------|-----------|----------------|
| 無 Authorization header | — | `missing_token` |
| `"Bearer abc.def.ghi"` | `"abc.def.ghi"` | nil |
| `"bearer abc.def.ghi"` | `"abc.def.ghi"` | nil |
| `"BEARER abc.def.ghi"` | `"abc.def.ghi"` | nil |
| `"Basic xyz"` | — | `invalid_token_format` |
| `"Bearer"`（無空格） | — | `invalid_token_format` |
| `"Bearer "`（空 token） | — | `invalid_token_format` |
| `"  Bearer abc"`（前導空白） | — | `invalid_token_format` |

#### Verify middleware（`httptest` + mock `RevokeRepository`）

| 案例 | 預期 |
|------|------|
| 有效 token + family 未撤銷 | 200，context 含 UserID / Role / JTI / FamilyID |
| 無 Authorization | 401，kind=`missing_token` |
| Bearer 格式錯誤 | 401，kind=`invalid_token_format` |
| 過期 token | 401，kind=`token_expired` |
| Family 已撤銷 | 401，kind=`session_revoked` |
| Redis 查詢失敗 | 500，kind=`internal` |

### Repository 整合測試

`testcontainers` 啟動 Redis：
- `RefreshRepo`
  - Save → Rotate(old, new) → 取得 userID/familyID，`reused=false`
  - 連續 Rotate 同一個 old token → 第二次 `reused=true`，回傳 familyID
  - Save → Revoke → Rotate（重放）→ `reused=true`
  - `Rotate` 的原子性：以 `errgroup` 並發兩個 rotate，僅一個成功、另一個觸發 reuse 或拒絕
- `RevokeRepo`：RevokeFamily → IsFamilyRevoked = true；TTL 到期後自動消失
