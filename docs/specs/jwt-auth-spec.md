# Backend ↔ Frontend 認證契約

> 本文件規範 **Go Backend 與 Next.js client 之間** 的認證 token 格式與流程，雙方都必須遵守。
>
> Backend 端的機制層（JWT 簽 / 驗、token store、family revocation）見 [foundation/15-auth-mechanism.md](./foundation/15-auth-mechanism.md)；政策層（login / register / refresh / users schema）見 [features/auth-feature-spec.md](./features/auth-feature-spec.md)。

## 角色定位

```
User (browser)
   │
   ▼
Next.js (client / SSR)            ← 持有 token、轉發請求；不持有 JWT_SECRET
   │   Authorization: Bearer <access_token>
   ▼
Go Backend (authority)            ← 簽發 + 驗證 token；唯一持有 JWT_SECRET
   │
   ├── PostgreSQL  (users + 密碼 hash)
   └── Redis       (refresh token + blacklist)
```

| 角色 | 職責 |
|------|------|
| **Backend** | 簽發 / 驗證 access token；產生 / 儲存 / 撤銷 refresh token；密碼 hash 驗證；持有 `JWT_SECRET` |
| **Next.js client** | 收 token、保存（建議 httpOnly cookie）、每次請求帶入 `Authorization` Header；**不持有 secret** |

---

## Token 設計

### Access Token（JWT）

短期、stateless、由 Backend 用 `JWT_SECRET` 簽發；client 每次請求帶入。

#### Header
```json
{ "alg": "HS256", "typ": "JWT" }
```

#### Payload
```json
{
  "sub": "user-id",
  "role": "user",
  "iss": "helpzy-backend",
  "jti": "550e8400-e29b-41d4-a716-446655440000",
  "iat": 1700000000,
  "exp": 1700000900
}
```

| 欄位 | 型別 | 說明 |
|------|------|------|
| `sub` | string | 使用者 ID |
| `role` | string | 使用者角色 |
| `iss` | string | 簽發者，固定為 `helpzy-backend` |
| `jti` | string (UUID) | Token 唯一 ID；撤銷時加入 Redis blacklist |
| `iat` | int64 | 簽發時間 |
| `exp` | int64 | 到期時間（`iat + JWT_ACCESS_TTL`） |

#### 簽章

| 屬性 | 值 |
|------|----|
| 演算法 | HS256 |
| Secret | `JWT_SECRET`（min 32 chars，僅 Backend 持有） |
| 預設 TTL | 1 天（`JWT_ACCESS_TTL`） |

---

### Refresh Token（Opaque）

長期、stateful、Backend 產生後**只回傳給 client 一次**；Redis 是單一真相。

| 屬性 | 值 |
|------|----|
| 格式 | 32-byte 隨機（`crypto/rand`），base64url 編碼 |
| **不是** JWT | 無 payload、不可解析——只是查表的 key |
| 儲存位置 | Redis：`refresh:{token} → {user_id, exp}` |
| 預設 TTL | 30 天（`JWT_REFRESH_TTL`） |
| Rotation | 每次 `/auth/refresh` **撤銷舊 token、發新 token**（一次性） |

> **為什麼 refresh token 不用 JWT？** 既然 Backend 有 Redis 可隨時撤銷，refresh token 不需要 stateless 的特性；opaque token 更短、payload 不洩漏資訊、撤銷只要刪 key。Stateless JWT 用於高頻請求的 access token 才划算。

---

## API 端點契約

> 完整路由規格見 [api-spec.md](./api-spec.md)。本表只列認證相關的契約面。

| 端點 | 用途 | 入參 | 回傳 |
|------|------|-----|------|
| `POST /api/v1/auth/register` | 註冊 | `{email, password}` | `{user, access_token, refresh_token}` |
| `POST /api/v1/auth/login` | 登入 | `{email, password}` | `{user, access_token, refresh_token}` |
| `POST /api/v1/auth/refresh` | 刷新 access token | `{refresh_token}` | `{access_token, refresh_token}`（新的一組） |
| `POST /api/v1/auth/logout` | 登出 | `{refresh_token}` | 204 |

### 登出行為

1. 從 Redis 刪除 `refresh:{token}`（refresh token 立即失效）
2. 將當前 access token 的 `jti` 加入 Redis blacklist，TTL 為剩餘 exp（讓 access token 也立即失效）

---

## 驗證規則（Backend 必須執行）

每個帶 `Authorization: Bearer` 的請求都會經過以下驗證鏈：

1. Bearer 格式正確
2. 簽章驗證（HS256 + `JWT_SECRET`）
3. `alg == HS256`（防 `alg: none`）
4. `iss == helpzy-backend`
5. `exp > now`（未過期）
6. `jti` **不在** Redis blacklist 中

任一步失敗 → 401，對外訊息統一為 `未授權`，避免資訊洩漏；具體原因記錄在 server log。

---

## 環境變數

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `JWT_SECRET` | ✓ | — | HS256 簽署金鑰，僅 Backend 持有，min 32 chars |
| `JWT_ACCESS_TTL` | ✗ | `86400` | Access token 有效期（秒，預設 1 天） |
| `JWT_REFRESH_TTL` | ✗ | `2592000` | Refresh token 有效期（秒，預設 30 天） |
| `BCRYPT_COST` | ✗ | `12` | 密碼 hash cost factor |

---

## 套件選擇

| 端 | 套件 |
|----|------|
| Backend (Go) JWT | `github.com/golang-jwt/jwt/v5` |
| Backend (Go) 密碼 | `golang.org/x/crypto/bcrypt` |
| Next.js client | 由 frontend 規格決定（建議用 httpOnly cookie 儲存 token） |
