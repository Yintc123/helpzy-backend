# 15 — Auth 機制層（Mechanism）

> 本文件規範 auth **機制層**：JWT、password、opaque token、signed ID 純函式，以及 Redis-backed token stores（refresh rotation、family revocation、opaque ticket）。
>
> 機制層**對「使用者是誰」一無所知**——不認 users 表、不認 role 列舉、不認 login 流程。那些屬於政策層，見 [`../features/auth-feature-spec.md`](../features/auth-feature-spec.md)。
>
> Token Family 機制概念見 [`tech-stack/token-family.md`](../../tech-stack/token-family.md)。

## 範圍

### 包含（機制層）

- JWT 簽發 / 驗證（純函式）
- 密碼 hash / 比對（純函式）
- Opaque token 產生（refresh / ticket 通用）
- IP fingerprint hashing
- SignedID + HMAC primitive（cookie 用）
- Refresh token rotation + reuse detection
- Family-level revocation
- Opaque ticket store（atomic GETDEL）
- JWT middleware（含 family 撤銷檢查）

### 不包含（政策層）

- users 表 schema 與 sqlc queries
- Role 列舉值（customer / agent）
- Login / Register / Logout / Refresh 業務流程
- WS Ticket / Visitor cookie 的 payload 結構與業務語意
- AuthService、handler、DTOs
- 訪客升級登入

---

## 模組責任

| 子模組 | 路徑 | 責任 |
|--------|------|------|
| Token 簽發 / 驗證 | `pkg/auth/jwt.go` | 純函式：簽 / 驗 access token |
| 密碼 hash | `pkg/auth/password.go` | 純函式：bcrypt hash / compare |
| Opaque token 產生 | `pkg/auth/token.go` | 純函式：`crypto/rand` + IP hash |
| Signed ID + HMAC | `pkg/auth/signedid.go` | 純函式：產生 / 簽 / 驗章 |
| Refresh store | `pkg/authstore/refresh.go` | Redis 原子 rotation + reuse detection |
| Family revoker | `pkg/authstore/revoke.go` | Redis family-level 撤銷 |
| Ticket store | `pkg/authstore/ticket.go` | Redis 原子 ticket Put / Take（GETDEL）|
| JWT Middleware | `pkg/middleware/jwt.go` | HTTP middleware：驗 access + 查 family 撤銷 + 注 context |

---

## Token Family 機制

完整概念與設計理由見 [`tech-stack/token-family.md`](../../tech-stack/token-family.md)；本節只列**機制層需要在意的不變式**：

1. 每次「登入」產生一個新 `family_id`（UUID v4，由政策層決定產生時機）
2. Refresh rotation 內所有 token 共享同一個 `family_id`（payload 內承載）
3. Rotation 時舊 token 搬到 `refresh_used:` 區作為 reuse detection 證據
4. 同一個 token 第二次出現 = reuse detected
5. 撤銷一個 family = 撤銷該 family 內所有 access + refresh token

### Redis Schema

**機制層只認 opaque payload 字串**——`payload` 結構由政策層決定編解碼：

| Key 格式 | Value | TTL | 用途 |
|---------|-------|-----|------|
| `refresh:{token}` | `<opaque payload>` | constructor 注入 | 有效 refresh token |
| `refresh_used:{token}` | `<opaque payload>` | constructor 注入 | 已 rotate 的 reuse detection 證據 |
| `family_revoked:{family_id}` | `"1"` | constructor 注入 | family 撤銷標記 |
| `ticket:{token}` | `<opaque payload>` | constructor 注入 | 一次性 ticket |

> **為什麼把 payload 結構推給政策層**：機制層不知道「user」「visitor」是什麼。Refresh 的 payload 在政策層可能是 `userID|familyID`，ticket 的 payload 可能是 `user|userID|familyID|origin|ipHash` 或 `anon|visitorID|origin|ipHash`——這些都是業務決策。機制層只負責「存進去、原子取出來、preserve 跨 rotation」。

---

## `pkg/auth/jwt.go`（純函式）

```go
package auth

// Issuer 是所有 access token 的 iss claim。
// 簽發與驗證共用此常數。
const Issuer = "helpzy-backend"

type AccessClaims struct {
    Sub       string    // 由政策層決定意義（一般為 user.id）
    Role      string    // 由政策層決定列舉值
    Jti       string    // 機制層自動產生
    Fid       string    // family_id：用於 family-level revocation
    ExpiresAt time.Time // 由 ParseAccess 從 exp claim 解出
}

// SignAccess 簽發 access token。
// caller 傳入任意字串的 sub / role / familyID；機制層不檢查內容意義。
// jti 內部產生；iss = Issuer；exp = now + ttl。
func SignAccess(sub, role, familyID, secret string, ttl time.Duration) (token, jti string, err error)

// ParseAccess 驗證並解析 access token。
// 驗證項目：簽章、alg == HS256、iss == Issuer、exp 未過期。
// 不查 family revocation——由 middleware 在拿到 claims 後負責。
func ParseAccess(token, secret string) (*AccessClaims, error)
```

### Sentinel errors

```go
var (
    ErrTokenExpired     = errors.New("token expired")
    ErrInvalidSignature = errors.New("invalid signature")
    ErrInvalidAlgorithm = errors.New("invalid algorithm")
    ErrInvalidIssuer    = errors.New("invalid issuer")
    ErrMalformedToken   = errors.New("malformed token")
)
```

政策層（middleware / handler）拿到 sentinel 後轉 `apperror.Unauthorized(...)`。

---

## `pkg/auth/password.go`（純函式）

```go
package auth

// HashPassword 以 bcrypt 雜湊明文密碼。cost 由 caller 傳入。
func HashPassword(plaintext string, cost int) (string, error)

// ComparePassword 比對 hash 與明文。
// 匹配回 nil；不符回 bcrypt.ErrMismatchedHashAndPassword。
// caller 應將回傳 error 一律當作「帳號或密碼錯誤」處理，不細分原因。
func ComparePassword(hash, plaintext string) error
```

> **守則**：登入失敗時 error message 一律統一，避免帳號列舉攻擊。此守則在政策層遵守，機制層只提供原子 compare。

---

## `pkg/auth/token.go`（純函式）

```go
package auth

// GenerateOpaqueToken 產生 32-byte crypto/rand 隨機字串，base64url 編碼。
// 用於 refresh token、WS ticket、任何需要「opaque、不可解析」的場景。
// 不簽章、不含 payload——它只是 store 的查表 key。
func GenerateOpaqueToken() (string, error)

// HashIP 將客戶端 IP 與系統 secret 一同 sha256 雜湊。
// 用於 ticket / session 的 IP fingerprint binding；避免明文 IP 入庫。
func HashIP(ip, secret string) string
```

---

## `pkg/auth/signedid.go`（純函式）

`SignedID` 是「base64url(32-byte random) + HMAC 簽章」的通用 primitive；自身**不帶任何業務語意**。政策層用它包出 `VisitorID`、其他簽章 ID 等業務型別。

```go
package auth

// SignedID 是 base64url(32-byte crypto/rand)。
// 純資料型別，本身不含語意。
type SignedID string

// GenerateSignedID 產生新 ID。
func GenerateSignedID() (SignedID, error)

// SignID 組裝 cookie / token 字串：base64url(id).base64url(hmac)
func SignID(id SignedID, secret string) string

// VerifySignedID 驗章後回傳 SignedID。
// 簽章不符 / 格式錯誤 → 回 ("", ErrInvalidSignedID)。
func VerifySignedID(raw, secret string) (SignedID, error)

var ErrInvalidSignedID = errors.New("invalid signed id")
```

---

## `pkg/authstore/refresh.go`

```go
package authstore

// RefreshStore 對 payload 完全不可知——只負責原子 rotation + reuse detection。
type RefreshStore interface {
    // Save 新 token + opaque payload（登入時呼叫）。
    Save(ctx context.Context, token, payload string) error

    // Rotate 原子旋轉 refresh token：
    //   1. 找到 refresh:{old}：
    //      - 取出 payload
    //      - DEL refresh:{old}
    //      - SET refresh_used:{old} = payload，TTL = constructor 注入值
    //      - SET refresh:{new}      = payload，TTL = constructor 注入值
    //      - 回傳 (payload, reused=false, nil)
    //   2. 找不到 refresh:{old}，但找到 refresh_used:{old}：
    //      - 取出 payload
    //      - 不寫入 new token
    //      - 回傳 (payload, reused=true, nil)  ← caller 必須依政策層決定如何反應
    //   3. 都找不到：
    //      - 回傳 ("", false, ErrTokenNotFound)
    //
    // 整段以 Lua script 在 Redis 端原子完成。
    Rotate(ctx context.Context, oldToken, newToken string) (payload string, reused bool, err error)

    // Revoke 主動移到 refresh_used 區（登出時呼叫）。
    // DEL refresh:{token} + SET refresh_used:{token} = ""（若原本不存在則為 no-op）。
    // 確保「登出後若舊 token 再被重放」仍會觸發 reuse detection。
    Revoke(ctx context.Context, token string) error
}

// Sentinel
var ErrTokenNotFound = errors.New("authstore: refresh token not found")

// Constructor：TTL 為系統常數，constructor 注入；方法簽章保持乾淨。
func NewRefreshStore(rdb *redis.Client, ttl time.Duration) *RefreshStoreImpl
```

### 為什麼 Rotate 必須原子

若拆成 `Lookup → Del → Save` 三步，兩個並發 refresh 請求可能：
- 都通過 `Lookup`，雙雙 SET 新 token → family 內出現兩條合法鏈
- 或 reuse detection 在 race 下漏報

Lua script 確保整段在 Redis 端原子完成。

### 為什麼 reuse detection 不在 store 內直接撤銷

`Rotate` 只**報告**「我看到 reuse」；**是否要 revoke family、何時 revoke** 是政策決定。例如未來可能想先觸發 step-up auth、寄通知再撤銷。機制層不該綁死政策。

---

## `pkg/authstore/revoke.go`

```go
package authstore

// FamilyRevoker：family 是 token rotation 機制的內建概念，與 user / session 解耦。
type FamilyRevoker interface {
    // Revoke 標記整個 token family 失效。
    // 實作：SET family_revoked:{familyID} = "1"，TTL = constructor 注入值（涵蓋 family 內最長壽命 token）。
    Revoke(ctx context.Context, familyID string) error

    // IsRevoked 由 JWT middleware 在驗證 claims 後查詢。
    IsRevoked(ctx context.Context, familyID string) (bool, error)
}

func NewFamilyRevoker(rdb *redis.Client, ttl time.Duration) *FamilyRevokerImpl
```

> Family 是**機制概念**而非政策。它描述「同一條 rotation 鏈內所有 token 的歸屬」，與「使用者 session 的產品語意」無關。一個 user 可同時有多個 family（多裝置）；一個 family 不對應任何 user-facing 物件。

---

## `pkg/authstore/ticket.go`

```go
package authstore

// TicketStore：一次性 opaque token 通用 store。
// 對 payload 完全不可知；政策層自己編解碼。
type TicketStore interface {
    // Put 寫入 token 與 payload，TTL = constructor 注入值。
    Put(ctx context.Context, token, payload string) error

    // Take 原子取出並刪除（GETDEL）。
    //   - 找到：回 (payload, nil)
    //   - 找不到（過期 / 已使用 / 不存在）：回 ("", ErrTicketNotFound)
    Take(ctx context.Context, token string) (payload string, err error)
}

// Sentinel
var ErrTicketNotFound = errors.New("authstore: ticket not found")

func NewTicketStore(rdb *redis.Client, ttl time.Duration) *TicketStoreImpl
```

### 為什麼用 `Put`/`Take` 而非 `Issue`/`Consume`

`Issue` / `Consume` 帶業務語意（「發行」「消費」一張 ticket）；機制層只是個 key-value store with TTL + atomic GETDEL，命名應反映機制——`Put` / `Take` 更精準且不暗示用途。

### 為什麼不暴露 `IssueAnonymous` 等變體

機制層只認 opaque payload。政策層自己決定 payload 內容（`user|...` 或 `anon|...`），機制層不需要為每種變體開一個方法。

---

## `pkg/middleware/jwt.go`

```go
package middleware

import (
    "helpzy/pkg/auth"
    "helpzy/pkg/authstore"
)

type JWTMiddleware struct {
    secret  string
    revoker authstore.FamilyRevoker
}

func NewJWTMiddleware(secret string, revoker authstore.FamilyRevoker) *JWTMiddleware {
    return &JWTMiddleware{secret: secret, revoker: revoker}
}

// Verify 驗證 access token 並注入 context。
// Context keys（WithUserID / WithRole / WithJTI / WithFamilyID）統一定義於
// pkg/middleware/context.go，見 09-middleware.md。
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

        revoked, err := m.revoker.IsRevoked(r.Context(), claims.Fid)
        if err != nil {
            httpx.WriteError(w, r, apperror.Internal(err))
            return
        }
        if revoked {
            httpx.WriteError(w, r,
                apperror.Unauthorized("session_revoked", "未授權"))
            return
        }

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
| `"bearer abc"` / `"BEARER abc"` | `"abc"` | — RFC 6750 規定 scheme 不分大小寫 |
| `"Basic xyz"` | error | `invalid_token_format` |
| `"Bearer"`（無空格） | error | `invalid_token_format` |
| `"Bearer "`（空 token） | error | `invalid_token_format` |
| `"  Bearer abc"`（前導空白） | error | `invalid_token_format` |

#### 設計重點

| 設計 | 理由 |
|------|------|
| `strings.Cut` 而非 `SplitN` | Cut 一次切分、語意更清楚 |
| `strings.EqualFold` 比對 `Bearer` | RFC 6750 §2.1 規定 scheme 不分大小寫 |
| 不檢查 token 內容（只檢查結構） | 內容驗證由 `auth.ParseAccess` 負責 |
| 未匯出（小寫開頭） | 只給 JWT middleware 用 |
| 回 `*AppError` 而非 sentinel | middleware 拿到後直接 `httpx.WriteError` |

---

## 機制層錯誤回應對應

機制層觸發的錯誤 kind（政策層特有的 kind 見 [`auth-feature-spec.md`](../features/auth-feature-spec.md)）：

| 情況 | `kind` | HTTP |
|------|--------|------|
| Header 缺 `Authorization` | `missing_token` | 401 |
| Bearer 格式錯誤 | `invalid_token_format` | 401 |
| 簽章驗證失敗 / `alg` 不符 / `iss` 不符 | `invalid_token` | 401 |
| Token 過期 | `token_expired` | 401 |
| Family 已撤銷 | `session_revoked` | 401 |
| Redis 查詢失敗 | `internal` | 500 |

---

## DI 樣板（foundation 部分）

機制層只組裝 stores 與 middleware；service 由政策層接手。

```go
// cmd/server/main.go（機制層 wiring 部分）
refreshStore  := authstore.NewRefreshStore(rdb, cfg.JWT.RefreshTTL)
familyRevoker := authstore.NewFamilyRevoker(rdb, cfg.JWT.RefreshTTL)
ticketStore   := authstore.NewTicketStore(rdb, cfg.WSTicket.TTL)

jwtMw := middleware.NewJWTMiddleware(cfg.JWT.Secret, familyRevoker)
```

接下來 `refreshStore` / `familyRevoker` / `ticketStore` 注入給 `authservice.New(...)`，由政策層編排業務流程。

完整 main.go 見 [12-startup.md](./12-startup.md)。

---

## 與其它規格的關係

| 規格 | 關聯 |
|------|------|
| [04-apperror.md](./04-apperror.md) | 錯誤型別 |
| [05-httpx.md](./05-httpx.md) | `httpx.WriteError` |
| [03-logger.md](./03-logger.md) | `logger.WithUserID` 注入 |
| [08-redis.md](./08-redis.md) | Redis client |
| [09-middleware.md](./09-middleware.md) | Middleware 套用順序、Context keys |
| [16-observability.md](./16-observability.md) | `auth.refresh.reused_total` 等 metric 由政策層在偵測到 reuse 時觸發 |
| [`tech-stack/token-family.md`](../../tech-stack/token-family.md) | Token Family 機制概念 |
| [`tech-stack/ticket.md`](../../tech-stack/ticket.md) | Opaque ticket 機制 |
| [`tech-stack/contextkey.md`](../../tech-stack/contextkey.md) | Context key 模式 |
| [`../features/auth-feature-spec.md`](../features/auth-feature-spec.md) | 政策層 |

---

## TDD 測試策略

### `pkg/auth/jwt.go`

| 案例 | 預期 |
|------|------|
| `SignAccess` + `ParseAccess` round-trip | claims 一致、jti 非空、`Fid` = 傳入值、`ExpiresAt` ≈ now + ttl（±1s 容忍） |
| `SignAccess` 自動帶入 `iss = Issuer` | ParseAccess 不報 `ErrInvalidIssuer` |
| 過期 token | `ErrTokenExpired` |
| 簽章錯誤 | `ErrInvalidSignature` |
| `alg: none` | `ErrInvalidAlgorithm` |
| `iss` 被竄改 | `ErrInvalidIssuer` |
| 缺 `fid` claim | `ErrMalformedToken` |

### `pkg/auth/password.go`

| 案例 | 預期 |
|------|------|
| `HashPassword` + `ComparePassword` round-trip | 不報錯 |
| 錯誤密碼 | `bcrypt.ErrMismatchedHashAndPassword` |
| 相同明文兩次 hash 結果不同（salt） | OK |

### `pkg/auth/token.go`

| 案例 | 預期 |
|------|------|
| `GenerateOpaqueToken` | 長度 ≈ 43 字元、兩次呼叫不重複 |
| `HashIP` 同 ip + secret 兩次 | 結果相同 |
| `HashIP` 不同 secret | 結果不同（防 rainbow） |
| `HashIP` 空字串 ip | 視為「未綁定」由 caller 決定不存（不在 helper 內處理） |

### `pkg/auth/signedid.go`

| 案例 | 預期 |
|------|------|
| `GenerateSignedID` | 長度固定、兩次呼叫不重複 |
| `SignID` + `VerifySignedID` round-trip | 取出相同 SignedID |
| 簽章部分被竄改 | `ErrInvalidSignedID` |
| ID 部分被竄改 | `ErrInvalidSignedID` |
| 格式錯誤（缺 `.`） | `ErrInvalidSignedID` |
| 不同 secret 驗證 | `ErrInvalidSignedID`（用於 secret 輪換） |

### `pkg/authstore/refresh.go`（整合測試，testcontainers Redis）

- `Save` → `Rotate(old, new)` → 回 (payload, reused=false, nil)、`refresh:new` 存在、`refresh_used:old` 存在
- 連續 `Rotate` 同一個 old → 第二次回 (payload, reused=true, nil)、**不寫入** new token
- `Save` → `Revoke` → `Rotate`（重放）→ reused=true
- 找不到的 token `Rotate` → `ErrTokenNotFound`
- 原子性：`errgroup` 並發兩個 rotate 同一個 old → 僅一個 reused=false、另一個 reused=true
- TTL：寫入後超過 ttl，key 自動消失

### `pkg/authstore/revoke.go`（整合測試）

- `Revoke` → `IsRevoked` = true
- 未撤銷的 family `IsRevoked` = false
- 撤銷後超過 TTL，key 消失，`IsRevoked` 回 false

### `pkg/authstore/ticket.go`（整合測試）

- `Put` → `Take` → 回正確 payload
- 同一 ticket 第二次 `Take` → `ErrTicketNotFound`
- 並發 `errgroup` `Take` 同一 ticket → 僅一個成功（GETDEL 原子）
- 未 `Take` 達 TTL → key 自動消失，再 `Take` → `ErrTicketNotFound`

### `pkg/middleware/jwt.go`

#### `extractBearerToken`（純函式，table-driven）

| 輸入 | 預期 token | 預期 error kind |
|------|-----------|----------------|
| 無 Authorization | — | `missing_token` |
| `"Bearer abc.def.ghi"` | `"abc.def.ghi"` | nil |
| `"bearer abc.def.ghi"` | `"abc.def.ghi"` | nil |
| `"BEARER abc.def.ghi"` | `"abc.def.ghi"` | nil |
| `"Basic xyz"` | — | `invalid_token_format` |
| `"Bearer"`（無空格） | — | `invalid_token_format` |
| `"Bearer "`（空 token） | — | `invalid_token_format` |
| `"  Bearer abc"`（前導空白） | — | `invalid_token_format` |

#### Verify middleware（`httptest` + mock `FamilyRevoker`）

| 案例 | 預期 |
|------|------|
| 有效 token + family 未撤銷 | 200，context 含 UserID / Role / JTI / FamilyID |
| 無 Authorization | 401，kind=`missing_token` |
| Bearer 格式錯誤 | 401，kind=`invalid_token_format` |
| 過期 token | 401，kind=`invalid_token` |
| Family 已撤銷 | 401，kind=`session_revoked` |
| Redis 查詢失敗 | 500，kind=`internal` |
