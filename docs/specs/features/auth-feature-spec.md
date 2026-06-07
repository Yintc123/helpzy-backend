# Auth Feature Spec（政策層）

> 本文件規範 auth **政策層**：業務 schema、業務型別、業務流程、HTTP 端點。
>
> 依賴機制層 [`foundation/15-auth-mechanism.md`](../foundation/15-auth-mechanism.md)；對機制層 store 完全不可知，只透過 interface 串接。
>
> Backend ↔ Frontend 跨服務契約見 [`../jwt-auth-spec.md`](../jwt-auth-spec.md)。
> 訪客身分整體生命週期見 [`../chat-spec.md`](../chat-spec.md)。

## 範圍

### 包含（政策層）

- `users` 表 schema 與 sqlc queries
- `models.User` 業務 entity
- `UserRepository` interface（service 層消費者側定義）
- `Role` 列舉（`customer` / `agent`）
- `VisitorID` 業務型別 + cookie helper
- `WSTicketOrigin` 列舉、role / origin 匹配規則
- Refresh / Ticket payload 編解碼
- `AuthService` 完整流程（Login / Register / Refresh / Logout / WSTicket / LinkVisitor）
- `CustomerIdentity` 型別（WS upgrade 後傳遞給 dispatcher）
- HTTP handlers + request/response DTOs
- 政策層特有錯誤回應

### 不包含（機制層，見 foundation/15-auth-mechanism.md）

- JWT 簽 / 驗
- bcrypt 細節
- Token rotation / family revocation Redis 操作
- Ticket store atomic GETDEL
- JWT middleware 實作

---

## 守則

### 1. `users` 是 auth 必要的最小 schema

此 migration 只放 auth 機制能跑起來的最少欄位。**業務欄位**（nickname、avatar、phone、email_verified_at、preferences、disabled_at 等）一律由各 feature 自己加 migration，**禁止**塞進這個 schema。

### 2. Interface segregation

`UserRepository` 只暴露 auth 自己用的方法（`FindByID`、`FindByEmail`、`GetRole`、`Create`）。其他業務的 user 操作另定義 `UserProfileRepository` 等，不擴充本 interface。

### 3. `models.User` 只放 auth 必要欄位

`ID`、`Email`、`Role`、`PasswordHash`。其他欄位由各 feature 自己擴 `models.User` 或新增 `models.UserProfile`（聚合根模式）。

### 4. 機制 / 政策邊界

- 政策層編解碼 payload（`userID|familyID`、`user|...`、`anon|...`），機制層只看 opaque string
- 機制層的 `Rotate` 報告 reuse、**不自動撤銷**；政策層決定撤銷時機
- Family 是機制概念但**政策層擁有觸發**（登出、reuse → revoke）

---

## Schema

### Migration

```sql
-- db/migrations/000001_create_users.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE users (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT        NOT NULL UNIQUE,
    password_hash TEXT        NOT NULL,
    role          TEXT        NOT NULL CHECK (role IN ('customer','agent')),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX users_email_idx ON users (email);
```

```sql
-- db/migrations/000001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

**欄位設計理由**：

| 欄位 | 為什麼 |
|------|--------|
| `id UUID DEFAULT gen_random_uuid()` | 不依賴 client 產生；不可枚舉 |
| `email UNIQUE` | 唯一登入識別 |
| `password_hash` | bcrypt 輸出；長度 60，用 TEXT 不固定限長以容納未來 algo 切換 |
| `role TEXT CHECK` | 列舉值可由後續 migration 擴充（修改 CHECK constraint） |
| `created_at` / `updated_at` | 審計基本欄；`updated_at` 由 application 寫入 |

### sqlc queries

```sql
-- db/queries/user.sql

-- name: GetUserByID :one
SELECT id, email, password_hash, role, created_at, updated_at
FROM users
WHERE id = $1;

-- name: GetUserRoleByID :one
SELECT role
FROM users
WHERE id = $1;

-- name: GetUserByEmail :one
SELECT id, email, password_hash, role, created_at, updated_at
FROM users
WHERE email = $1;

-- name: CreateUser :one
INSERT INTO users (email, password_hash, role)
VALUES ($1, $2, $3)
RETURNING id, email, role, created_at, updated_at;
```

**為什麼 `GetUserRoleByID` 單獨一條**：refresh 與 ticket consume 流程只需 role，整列回傳浪費頻寬與 deserialize 成本。每分鐘可能跑數萬次，值得 dedicated query。

---

## `internal/models/user.go`

```go
package models

import "github.com/google/uuid"

type User struct {
    ID           uuid.UUID
    Email        string
    Role         Role
    PasswordHash string // 僅 service 層用；不暴露給 handler/response（DTO 不含此欄位）
}
```

> Service 層回傳 `*User` 給 handler 時，handler 必須轉成 response DTO 並省略 `PasswordHash`。**禁止**直接 JSON serialize `models.User`。

---

## Role 列舉

```go
// internal/services/auth/role.go
package authservice

type Role string

const (
    RoleCustomer Role = "customer"
    RoleAgent    Role = "agent"
)

func ParseRole(s string) (Role, error) {
    switch Role(s) {
    case RoleCustomer, RoleAgent:
        return Role(s), nil
    default:
        return "", fmt.Errorf("invalid role: %q", s)
    }
}
```

**未來新增 role 步驟**：
1. 新 migration `ALTER TABLE users` 改 CHECK constraint
2. `role.go` 加常數 + `ParseRole` 分支
3. 對應的 origin / 權限規則（見 `originMatchesRole`）

---

## `UserRepository`（消費者側定義）

```go
// internal/services/auth/service.go
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*models.User, error)
    FindByEmail(ctx context.Context, email string) (*models.User, error)
    GetRole(ctx context.Context, id string) (Role, error)
    Create(ctx context.Context, in CreateUserInput) (*models.User, error)
}

type CreateUserInput struct {
    Email        string
    PasswordHash string
    Role         Role
}
```

實作位置：`internal/repository/user.go`（包 sqlc 生成碼，將 `pgx.ErrNoRows` 轉 `apperror.NotFound`）。

```go
// internal/repository/user.go
type UserRepo struct {
    pool *pgxpool.Pool
}

func NewUserRepo(pool *pgxpool.Pool) *UserRepo
func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error)
func (r *UserRepo) FindByEmail(ctx context.Context, email string) (*models.User, error)
func (r *UserRepo) GetRole(ctx context.Context, id string) (authservice.Role, error)
func (r *UserRepo) Create(ctx context.Context, in authservice.CreateUserInput) (*models.User, error)
```

> Repository 透過 [`database.Executor(ctx, pool)`](../foundation/07-database.md#repository-使用方式) 取得執行者（tx 或 pool），支援跨 repo 交易。

---

## `ConversationLinker`（最小依賴）

`AuthService` 對 conversation 只有單一方法依賴；於 auth service 內側自定義最小 interface，避免依賴 chat-spec 完整 `ConversationRepository`。

```go
// internal/services/auth/service.go
type ConversationLinker interface {
    // LinkVisitorToCustomer 將指定 visitor_id 的對話綁定到 user_id。
    // 實作（冪等）：
    //   UPDATE conversations
    //   SET customer_id = :user_id, updated_at = NOW()
    //   WHERE visitor_id = :vid AND customer_id IS NULL
    LinkVisitorToCustomer(ctx context.Context, visitorID, userID string) error
}
```

`internal/repository/conversation.go` 的完整 `ConversationRepository` 會包含此 interface（Go embedding），但 auth 套件不直接 import chat repository ——**單向依賴**。

---

## `VisitorID` 業務型別

機制層提供 `auth.SignedID` primitive；政策層包出 `VisitorID` 業務語意。

```go
// internal/services/auth/visitor.go
package authservice

import "helpzy/pkg/auth"

// VisitorID 是匿名訪客識別碼，stateless（HMAC 簽章、無 server side state）。
type VisitorID auth.SignedID

const VisitorCookieName = "helpzy.vid"

// IssueVisitor 產生新 visitor 並回 cookie raw value。
func IssueVisitor(secret string) (cookieValue string, vid VisitorID, err error) {
    id, err := auth.GenerateSignedID()
    if err != nil {
        return "", "", err
    }
    return auth.SignID(id, secret), VisitorID(id), nil
}

// ParseVisitor 驗 cookie 並回 VisitorID。
func ParseVisitor(cookieValue, secret string) (VisitorID, error) {
    id, err := auth.VerifySignedID(cookieValue, secret)
    if err != nil {
        return "", err // 由 handler 轉 apperror.Unauthorized("invalid_visitor", ...)
    }
    return VisitorID(id), nil
}
```

### Cookie 屬性

| 屬性 | 值 |
|------|-----|
| Name | `helpzy.vid` |
| HttpOnly | `true`（防 XSS 讀取） |
| Secure | production = `true`；local = `false` |
| SameSite | `Lax`（允許 top-level 跨站導航帶 cookie） |
| Path | `/` |
| Max-Age | `cfg.Visitor.TTL`（預設 30 天） |

### 撤銷策略

- **正常情境不主動撤銷**——stateless 設計，沒有 server side state 可清
- **批次撤銷**：輪換 `VISITOR_COOKIE_SECRET` → 所有現存 cookie 立即失效

### Sentinel 與 mapping

機制層回 `auth.ErrInvalidSignedID`；politik 層 / handler 轉 `apperror.Unauthorized("invalid_visitor", "未授權")`。

---

## Payload 編解碼

機制層 store 對 payload opaque；政策層在 `internal/services/auth/payload.go` 集中編解碼：

```go
// internal/services/auth/payload.go
package authservice

// ── Refresh payload ────────────────────────────────────────

// 格式：userID|familyID
func encodeRefreshPayload(userID, familyID string) string {
    return userID + "|" + familyID
}

func decodeRefreshPayload(s string) (userID, familyID string, err error) {
    parts := strings.SplitN(s, "|", 2)
    if len(parts) != 2 || parts[0] == "" || parts[1] == "" {
        return "", "", fmt.Errorf("invalid refresh payload: %q", s)
    }
    return parts[0], parts[1], nil
}

// ── Ticket payload ────────────────────────────────────────

type ticketKind string

const (
    ticketKindUser ticketKind = "user"
    ticketKindAnon ticketKind = "anon"
)

type ticketPayload struct {
    Kind      ticketKind
    UserID    string         // Kind = user
    FamilyID  string         // Kind = user
    VisitorID string         // Kind = anon
    Origin    WSTicketOrigin
    IPHash    string
}

// 格式：
//   user|userID|familyID|origin|ipHash
//   anon|visitorID|origin|ipHash
func encodeTicketPayload(p ticketPayload) string
func decodeTicketPayload(s string) (ticketPayload, error)
```

**為什麼分 user / anon 兩種 prefix 而非統一格式**：
- 兩種模式語意不同（已登入 vs 匿名）；強迫一個 schema 會出現空欄位混淆
- Prefix 分支是 Tagged Union 的 Go 慣用寫法

---

## `WSTicketOrigin` 與 role 匹配

```go
// internal/services/auth/role.go（接續 role 定義）

type WSTicketOrigin string

const (
    WSOriginCustomer WSTicketOrigin = "customer"
    WSOriginAgent    WSTicketOrigin = "agent"
)

func ParseWSTicketOrigin(s string) (WSTicketOrigin, error) {
    switch WSTicketOrigin(s) {
    case WSOriginCustomer, WSOriginAgent:
        return WSTicketOrigin(s), nil
    default:
        return "", fmt.Errorf("invalid origin: %q", s)
    }
}

func originMatchesRole(origin WSTicketOrigin, role Role) bool {
    switch origin {
    case WSOriginCustomer:
        return role == RoleCustomer
    case WSOriginAgent:
        return role == RoleAgent
    default:
        return false
    }
}
```

---

## `CustomerIdentity`

WS upgrade 後傳遞給 dispatcher 的統一身分型別；登入與訪客模式共用。

```go
// internal/services/auth/service.go
type CustomerIdentity struct {
    UserID    uuid.UUID   // 已登入；匿名為 uuid.Nil
    FamilyID  uuid.UUID   // 已登入；匿名為 uuid.Nil
    VisitorID string      // 匿名；已登入為空字串
    Role      Role        // 已登入；匿名為 ""
}

func (i CustomerIdentity) IsAnonymous() bool {
    return i.UserID == uuid.Nil
}
```

Dispatcher 拿到 `CustomerIdentity` 後**完全不在乎登入與否**，只是把它存進 `Session.Identity` 帶著走。

---

## `AuthService`

### Struct

```go
// internal/services/auth/service.go
package authservice

import (
    "helpzy/internal/config"
    "helpzy/pkg/authstore"
)

type AuthService struct {
    userRepo           UserRepository
    conversationLinker ConversationLinker
    refreshStore       authstore.RefreshStore
    familyRevoker      authstore.FamilyRevoker
    ticketStore        authstore.TicketStore
    jwt                config.JWTConfig
    wsTicket           config.WSTicketConfig
    visitor            config.VisitorConfig
    bcryptCost         int
}

func New(
    userRepo UserRepository,
    conversationLinker ConversationLinker,
    refreshStore authstore.RefreshStore,
    familyRevoker authstore.FamilyRevoker,
    ticketStore authstore.TicketStore,
    jwt config.JWTConfig,
    wsTicket config.WSTicketConfig,
    visitor config.VisitorConfig,
    bcryptCost int,
) *AuthService
```

### Token Pair

```go
type TokenPair struct {
    Access  string
    Refresh string
}
```

### `Login`

```go
// POST /api/v1/auth/login
type LoginInput struct {
    Email    string
    Password string
}

func (s *AuthService) Login(ctx context.Context, in LoginInput) (*TokenPair, error) {
    observability.LoginAttempts.Add(ctx, 1)

    user, err := s.userRepo.FindByEmail(ctx, in.Email)
    if err != nil {
        // 不分辨「帳號不存在」vs「找不到」——統一錯誤訊息防帳號列舉
        if errors.Is(err, apperror.ErrNotFound) {
            observability.LoginFailures.Add(ctx, 1, ... "no_user")
            return nil, apperror.Unauthorized("invalid_credentials", "帳號或密碼錯誤")
        }
        return nil, err
    }

    if err := auth.ComparePassword(user.PasswordHash, in.Password); err != nil {
        observability.LoginFailures.Add(ctx, 1, ... "bad_password")
        return nil, apperror.Unauthorized("invalid_credentials", "帳號或密碼錯誤")
    }

    familyID := uuid.NewString()

    refresh, err := auth.GenerateOpaqueToken()
    if err != nil { return nil, apperror.Internal(err) }

    access, _, err := auth.SignAccess(user.ID.String(), string(user.Role), familyID, s.jwt.Secret, s.jwt.AccessTTL)
    if err != nil { return nil, apperror.Internal(err) }

    if err := s.refreshStore.Save(ctx, refresh,
        encodeRefreshPayload(user.ID.String(), familyID),
    ); err != nil {
        return nil, apperror.Internal(err)
    }

    return &TokenPair{Access: access, Refresh: refresh}, nil
}
```

### `Register`

```go
// POST /api/v1/auth/register
type RegisterInput struct {
    Email    string
    Password string  // 已通過 validator 的 password rule
    Role     Role    // 預設 RoleCustomer；admin 端建立 agent 時帶入
}

func (s *AuthService) Register(ctx context.Context, in RegisterInput) (*models.User, error) {
    hash, err := auth.HashPassword(in.Password, s.bcryptCost)
    if err != nil { return nil, apperror.Internal(err) }

    role := in.Role
    if role == "" { role = RoleCustomer }

    user, err := s.userRepo.Create(ctx, CreateUserInput{
        Email:        in.Email,
        PasswordHash: hash,
        Role:         role,
    })
    if err != nil {
        // 預期 repository 將 unique violation 轉為 apperror.Conflict("user_exists", ...)
        return nil, err
    }
    return user, nil
}
```

> Repository 必須將 PostgreSQL `23505 unique_violation`（email 重複）轉為 `apperror.Conflict("user_exists", "此 email 已註冊")`。

### `Refresh`

```go
// POST /api/v1/auth/refresh
func (s *AuthService) Refresh(ctx context.Context, oldRefresh string) (*TokenPair, error) {
    newRefresh, err := auth.GenerateOpaqueToken()
    if err != nil { return nil, apperror.Internal(err) }

    payload, reused, err := s.refreshStore.Rotate(ctx, oldRefresh, newRefresh)
    if err != nil {
        if errors.Is(err, authstore.ErrTokenNotFound) {
            return nil, apperror.Unauthorized("invalid_token", "未授權")
        }
        return nil, apperror.Internal(err)
    }

    userID, familyID, err := decodeRefreshPayload(payload)
    if err != nil { return nil, apperror.Internal(err) }

    if reused {
        _ = s.familyRevoker.Revoke(ctx, familyID)
        observability.TokenReusedTotal.Add(ctx, 1)
        logger.From(ctx).Warn("refresh token reuse detected",
            "user_id", userID, "family_id", familyID)
        return nil, apperror.Unauthorized("token_reused", "未授權")
    }

    role, err := s.userRepo.GetRole(ctx, userID)
    if err != nil { return nil, err }

    access, _, err := auth.SignAccess(userID, string(role), familyID, s.jwt.Secret, s.jwt.AccessTTL)
    if err != nil { return nil, apperror.Internal(err) }

    return &TokenPair{Access: access, Refresh: newRefresh}, nil
}
```

### `Logout`

```go
// POST /api/v1/auth/logout
func (s *AuthService) Logout(ctx context.Context, claims *auth.AccessClaims, refreshToken string) error {
    if err := s.familyRevoker.Revoke(ctx, claims.Fid); err != nil {
        return apperror.Internal(err)
    }
    return s.refreshStore.Revoke(ctx, refreshToken)
}
```

### `IssueWSTicket`（已登入）

```go
// POST /api/v1/auth/ws-ticket
func (s *AuthService) IssueWSTicket(
    ctx context.Context,
    claims *auth.AccessClaims,
    origin WSTicketOrigin,
    clientIP string,
) (string, error) {
    // 1. Family 撤銷檢查
    revoked, err := s.familyRevoker.IsRevoked(ctx, claims.Fid)
    if err != nil { return "", apperror.Internal(err) }
    if revoked { return "", apperror.Unauthorized("session_revoked", "未授權") }

    // 2. Origin / role 匹配
    role, err := ParseRole(claims.Role)
    if err != nil { return "", apperror.Internal(err) }
    if !originMatchesRole(origin, role) {
        return "", apperror.Forbidden("origin_role_mismatch", "未授權")
    }

    // 3. 產生 + 寫入
    token, err := auth.GenerateOpaqueToken()
    if err != nil { return "", apperror.Internal(err) }

    ipHash := ""
    if clientIP != "" && s.wsTicket.BindIP {
        ipHash = auth.HashIP(clientIP, s.wsTicket.Secret)
    }

    payload := encodeTicketPayload(ticketPayload{
        Kind:     ticketKindUser,
        UserID:   claims.Sub,
        FamilyID: claims.Fid,
        Origin:   origin,
        IPHash:   ipHash,
    })

    if err := s.ticketStore.Put(ctx, token, payload); err != nil {
        return "", apperror.Internal(err)
    }
    return token, nil
}
```

### `IssueAnonymousWSTicket`（訪客）

```go
// POST /api/v1/anonymous/ws-ticket
func (s *AuthService) IssueAnonymousWSTicket(
    ctx context.Context,
    visitorID VisitorID,
    origin WSTicketOrigin,
    clientIP string,
) (string, error) {
    // 訪客只能申請 customer origin
    if origin != WSOriginCustomer {
        return "", apperror.Forbidden("origin_role_mismatch", "未授權")
    }

    token, err := auth.GenerateOpaqueToken()
    if err != nil { return "", apperror.Internal(err) }

    ipHash := ""
    if clientIP != "" && s.wsTicket.BindIP {
        ipHash = auth.HashIP(clientIP, s.wsTicket.Secret)
    }

    payload := encodeTicketPayload(ticketPayload{
        Kind:      ticketKindAnon,
        VisitorID: string(visitorID),
        Origin:    origin,
        IPHash:    ipHash,
    })

    if err := s.ticketStore.Put(ctx, token, payload); err != nil {
        return "", apperror.Internal(err)
    }
    return token, nil
}
```

### `ConsumeWSTicket`

```go
func (s *AuthService) ConsumeWSTicket(
    ctx context.Context,
    token string,
    expectedOrigin WSTicketOrigin,
    peerIP string,
) (CustomerIdentity, error) {
    raw, err := s.ticketStore.Take(ctx, token)
    if err != nil {
        if errors.Is(err, authstore.ErrTicketNotFound) {
            return CustomerIdentity{}, apperror.Unauthorized("ticket_invalid", "未授權")
        }
        return CustomerIdentity{}, apperror.Internal(err)
    }

    payload, err := decodeTicketPayload(raw)
    if err != nil { return CustomerIdentity{}, apperror.Internal(err) }

    if payload.Origin != expectedOrigin {
        return CustomerIdentity{}, apperror.Unauthorized("ticket_origin_mismatch", "未授權")
    }

    if payload.IPHash != "" {
        if auth.HashIP(peerIP, s.wsTicket.Secret) != payload.IPHash {
            return CustomerIdentity{}, apperror.Unauthorized("ticket_ip_mismatch", "未授權")
        }
    }

    switch payload.Kind {
    case ticketKindUser:
        revoked, err := s.familyRevoker.IsRevoked(ctx, payload.FamilyID)
        if err != nil { return CustomerIdentity{}, apperror.Internal(err) }
        if revoked {
            return CustomerIdentity{}, apperror.Unauthorized("session_revoked", "未授權")
        }
        role, err := s.userRepo.GetRole(ctx, payload.UserID)
        if err != nil { return CustomerIdentity{}, err }
        return CustomerIdentity{
            UserID:   uuid.MustParse(payload.UserID),
            FamilyID: uuid.MustParse(payload.FamilyID),
            Role:     role,
        }, nil

    case ticketKindAnon:
        return CustomerIdentity{
            VisitorID: payload.VisitorID,
            Role:      "",
        }, nil

    default:
        return CustomerIdentity{}, apperror.Internal(fmt.Errorf("unknown ticket kind: %q", payload.Kind))
    }
}
```

### `LinkVisitor`

```go
// POST /api/v1/auth/link-visitor
func (s *AuthService) LinkVisitor(
    ctx context.Context,
    claims *auth.AccessClaims,
    visitorID VisitorID,
) error {
    return s.conversationLinker.LinkVisitorToCustomer(
        ctx, string(visitorID), claims.Sub,
    )
}
```

- 操作冪等（重複呼叫無副作用）
- 不刪除 `visitor_id` 欄位 — 保留作為審計與防止 visitor cookie 被重用為其他帳號的入口
- 不回滾：即使 link 失敗，使用者登入仍成功（端點獨立）

---

## HTTP Handlers

### Handler struct

```go
// internal/handlers/auth.go
type AuthHandler struct {
    svc *authservice.AuthService
    cfg config.Config // 取 Visitor.CookieSecret 等
}

func NewAuthHandler(svc *authservice.AuthService, cfg config.Config) *AuthHandler
```

### `POST /api/v1/auth/register`

```go
type registerRequest struct {
    Email    string `json:"email"    validate:"required,email"`
    Password string `json:"password" validate:"required,password"`
}

type registerResponse struct {
    ID    string `json:"id"`
    Email string `json:"email"`
    Role  string `json:"role"`
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) error {
    var req registerRequest
    if err := httpx.DecodeJSON(r, &req); err != nil { return err }
    if err := validator.Struct(req); err != nil { return err }

    user, err := h.svc.Register(r.Context(), authservice.RegisterInput{
        Email:    req.Email,
        Password: req.Password,
        // Role: 省略 → service 預設 customer
    })
    if err != nil { return err }

    return response.JSON(w, http.StatusCreated, registerResponse{
        ID: user.ID.String(), Email: user.Email, Role: string(user.Role),
    })
}
```

### `POST /api/v1/auth/login`

```go
type loginRequest struct {
    Email    string `json:"email"    validate:"required,email"`
    Password string `json:"password" validate:"required"`
}

type tokenPairResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) error {
    var req loginRequest
    if err := httpx.DecodeJSON(r, &req); err != nil { return err }
    if err := validator.Struct(req); err != nil { return err }

    pair, err := h.svc.Login(r.Context(), authservice.LoginInput{
        Email: req.Email, Password: req.Password,
    })
    if err != nil { return err }

    return response.JSON(w, http.StatusOK, tokenPairResponse{
        AccessToken: pair.Access, RefreshToken: pair.Refresh,
    })
}
```

### `POST /api/v1/auth/refresh`

```go
type refreshRequest struct {
    RefreshToken string `json:"refresh_token" validate:"required"`
}

func (h *AuthHandler) Refresh(w http.ResponseWriter, r *http.Request) error {
    var req refreshRequest
    if err := httpx.DecodeJSON(r, &req); err != nil { return err }
    if err := validator.Struct(req); err != nil { return err }

    pair, err := h.svc.Refresh(r.Context(), req.RefreshToken)
    if err != nil { return err }

    return response.JSON(w, http.StatusOK, tokenPairResponse{
        AccessToken: pair.Access, RefreshToken: pair.Refresh,
    })
}
```

### `POST /api/v1/auth/logout`

```go
type logoutRequest struct {
    RefreshToken string `json:"refresh_token" validate:"required"`
}

func (h *AuthHandler) Logout(w http.ResponseWriter, r *http.Request) error {
    var req logoutRequest
    if err := httpx.DecodeJSON(r, &req); err != nil { return err }
    if err := validator.Struct(req); err != nil { return err }

    claims, err := claimsFromContext(r.Context())
    if err != nil { return err }

    if err := h.svc.Logout(r.Context(), claims, req.RefreshToken); err != nil {
        return err
    }
    return response.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

// claimsFromContext 從 middleware 注入的四個 context value 重組 claims。
// 缺值代表路由配置漏掛 JWT middleware → 401。
func claimsFromContext(ctx context.Context) (*auth.AccessClaims, error) {
    sub, ok1 := middleware.UserIDFrom(ctx)
    role, ok2 := middleware.RoleFrom(ctx)
    jti, ok3 := middleware.JTIFrom(ctx)
    fid, ok4 := middleware.FamilyIDFrom(ctx)
    if !ok1 || !ok2 || !ok3 || !ok4 {
        return nil, apperror.Unauthorized("missing_claims", "未授權")
    }
    return &auth.AccessClaims{Sub: sub, Role: role, Jti: jti, Fid: fid}, nil
}
```

### `POST /api/v1/auth/ws-ticket`

```go
type wsTicketRequest struct {
    Origin string `json:"origin" validate:"required,oneof=customer agent"`
}

type wsTicketResponse struct {
    Ticket    string `json:"ticket"`
    ExpiresIn int    `json:"expires_in"` // seconds
}

func (h *AuthHandler) WSTicket(w http.ResponseWriter, r *http.Request) error {
    var req wsTicketRequest
    if err := httpx.DecodeJSON(r, &req); err != nil { return err }
    if err := validator.Struct(req); err != nil { return err }

    claims, err := claimsFromContext(r.Context())
    if err != nil { return err }

    origin, err := authservice.ParseWSTicketOrigin(req.Origin)
    if err != nil { return apperror.BadRequest("invalid_origin", "origin 無效") }

    ticket, err := h.svc.IssueWSTicket(r.Context(), claims, origin, clientIP(r))
    if err != nil { return err }

    return response.JSON(w, http.StatusOK, wsTicketResponse{
        Ticket:    ticket,
        ExpiresIn: int(h.cfg.WSTicket.TTL.Seconds()),
    })
}
```

### `POST /api/v1/anonymous/ws-ticket`

```go
func (h *AuthHandler) AnonymousWSTicket(w http.ResponseWriter, r *http.Request) error {
    cookie, err := r.Cookie(authservice.VisitorCookieName)
    if err != nil {
        // 無 cookie：發新 visitor cookie + 申請 ticket
        cookieValue, vid, err := authservice.IssueVisitor(h.cfg.Visitor.CookieSecret)
        if err != nil { return apperror.Internal(err) }
        setVisitorCookie(w, h.cfg, cookieValue)
        return h.issueAnonymousTicket(w, r, vid)
    }
    vid, err := authservice.ParseVisitor(cookie.Value, h.cfg.Visitor.CookieSecret)
    if err != nil {
        return apperror.Unauthorized("invalid_visitor", "未授權").Wrap(err)
    }
    return h.issueAnonymousTicket(w, r, vid)
}

func (h *AuthHandler) issueAnonymousTicket(w http.ResponseWriter, r *http.Request, vid authservice.VisitorID) error {
    var req wsTicketRequest
    if err := httpx.DecodeJSON(r, &req); err != nil { return err }
    if err := validator.Struct(req); err != nil { return err }

    origin, err := authservice.ParseWSTicketOrigin(req.Origin)
    if err != nil { return apperror.BadRequest("invalid_origin", "origin 無效") }

    ticket, err := h.svc.IssueAnonymousWSTicket(r.Context(), vid, origin, clientIP(r))
    if err != nil { return err }

    return response.JSON(w, http.StatusOK, wsTicketResponse{
        Ticket:    ticket,
        ExpiresIn: int(h.cfg.WSTicket.TTL.Seconds()),
    })
}

func setVisitorCookie(w http.ResponseWriter, cfg config.Config, value string) {
    http.SetCookie(w, &http.Cookie{
        Name:     authservice.VisitorCookieName,
        Value:    value,
        Path:     "/",
        HttpOnly: true,
        Secure:   cfg.App.Env == "production",
        SameSite: http.SameSiteLaxMode,
        MaxAge:   int(cfg.Visitor.TTL.Seconds()),
    })
}
```

### `POST /api/v1/auth/link-visitor`

```go
func (h *AuthHandler) LinkVisitor(w http.ResponseWriter, r *http.Request) error {
    cookie, err := r.Cookie(authservice.VisitorCookieName)
    if err != nil {
        return apperror.BadRequest("missing_visitor", "缺少 visitor cookie")
    }
    vid, err := authservice.ParseVisitor(cookie.Value, h.cfg.Visitor.CookieSecret)
    if err != nil {
        return apperror.Unauthorized("invalid_visitor", "未授權").Wrap(err)
    }
    claims, err := claimsFromContext(r.Context())
    if err != nil { return err }

    if err := h.svc.LinkVisitor(r.Context(), claims, vid); err != nil {
        return err
    }
    return response.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

### `clientIP` helper

```go
// internal/handlers/auth.go
func clientIP(r *http.Request) string {
    host, _, err := net.SplitHostPort(r.RemoteAddr)
    if err != nil {
        return strings.TrimSpace(r.RemoteAddr)
    }
    return host
}
```

> 信任邊界與 RateLimit middleware 一致：若 backend 在反向代理後，依賴在最外層掛 `chi/middleware.RealIP` 改寫 `RemoteAddr`。本 helper 不直接讀 `X-Forwarded-For`。

---

## 政策層錯誤回應對應

機制層的 kind 見 [`foundation/15-auth-mechanism.md`](../foundation/15-auth-mechanism.md#機制層錯誤回應對應)。

| 情況 | `kind` | HTTP |
|------|--------|------|
| 帳號 / 密碼錯誤 | `invalid_credentials` | 401 |
| 註冊 email 衝突 | `user_exists` | 409 |
| 缺 context claims（路由漏掛 middleware） | `missing_claims` | 401 |
| Refresh token 已被 rotate 過 | `token_reused` | 401 |
| WS Ticket 不存在 / 已使用 | `ticket_invalid` | 401 |
| WS Ticket origin 與升級端點不符 | `ticket_origin_mismatch` | 401 |
| WS Ticket IP fingerprint 不符 | `ticket_ip_mismatch` | 401 |
| WS Ticket origin 與 caller role 不符 | `origin_role_mismatch` | 403 |
| Visitor cookie 簽章不符 / 格式錯誤 | `invalid_visitor` | 401 |
| 訪客 link 缺 cookie | `missing_visitor` | 400 |
| 訪客嘗試申請 agent origin | `origin_role_mismatch` | 403 |
| Origin 列舉值錯誤 | `invalid_origin` | 400 |

---

## DI 組裝

```go
// cmd/server/main.go（節錄）

// ── 機制層（foundation） ────────────────────────────────
refreshStore  := authstore.NewRefreshStore(rdb, cfg.JWT.RefreshTTL)
familyRevoker := authstore.NewFamilyRevoker(rdb, cfg.JWT.RefreshTTL)
ticketStore   := authstore.NewTicketStore(rdb, cfg.WSTicket.TTL)
jwtMw         := middleware.NewJWTMiddleware(cfg.JWT.Secret, familyRevoker)

// ── 政策層 ──────────────────────────────────────────
userRepo         := repository.NewUserRepo(pool)
conversationRepo := repository.NewConversationRepo(pool) // 滿足 ConversationLinker

authSvc := authservice.New(
    userRepo, conversationRepo,
    refreshStore, familyRevoker, ticketStore,
    cfg.JWT, cfg.WSTicket, cfg.Visitor, cfg.BcryptCost,
)
authHandler := handlers.NewAuthHandler(authSvc, *cfg)
```

完整 main.go 見 [`12-startup.md`](../foundation/12-startup.md)。

---

## 與其它規格的關係

| 規格 | 關聯 |
|------|------|
| [`foundation/15-auth-mechanism.md`](../foundation/15-auth-mechanism.md) | 機制層（依賴方向） |
| [`foundation/14-sqlc-and-migration.md`](../foundation/14-sqlc-and-migration.md) | Migration + sqlc 工作流 |
| [`foundation/06-validator.md`](../foundation/06-validator.md) | DTO `validate:` tag |
| [`foundation/09-middleware.md`](../foundation/09-middleware.md) | 路由套用順序、Context keys |
| [`../jwt-auth-spec.md`](../jwt-auth-spec.md) | Backend ↔ Frontend 跨服務契約 |
| [`../chat-spec.md`](../chat-spec.md) | 訪客身分整體生命週期、handoff 流程 |

---

## TDD 測試策略

### `internal/repository/user.go`（整合，testcontainers Postgres）

| 案例 | 預期 |
|------|------|
| `Create` 新 email | 回 user、`created_at` / `updated_at` 非零 |
| `Create` 重複 email | `apperror.Conflict("user_exists", ...)` |
| `FindByID` 存在 | 回 user |
| `FindByID` 不存在 | `apperror.NotFound("user_not_found", ...)` |
| `FindByEmail` | 同上 |
| `GetRole` 存在 | 回 `Role` |
| `GetRole` 不存在 | `apperror.NotFound("user_not_found", ...)` |

### `internal/services/auth/payload.go`（純函式，table-driven）

| 案例 | 預期 |
|------|------|
| `encodeRefreshPayload` + `decodeRefreshPayload` round-trip | 一致 |
| `decodeRefreshPayload("")` | error |
| `decodeRefreshPayload("foo")`（無 \|） | error |
| `decodeRefreshPayload("foo\|")`（空 family） | error |
| `encodeTicketPayload` user / anon + decode round-trip | 一致 |
| `decodeTicketPayload` prefix 不合法 | error |
| `decodeTicketPayload` user 模式缺欄位 | error |

### `internal/services/auth/visitor.go`（純函式）

| 案例 | 預期 |
|------|------|
| `IssueVisitor` + `ParseVisitor` round-trip | 一致 |
| `ParseVisitor` 簽章被竄改 | error |
| `ParseVisitor` 不同 secret | error（用於 secret 輪換） |

### `internal/services/auth/role.go`（純函式）

| 案例 | 預期 |
|------|------|
| `ParseRole("customer")` / `"agent"` | OK |
| `ParseRole("admin")` | error |
| `ParseWSTicketOrigin` | 同上 |
| `originMatchesRole(customer, RoleCustomer)` | true |
| `originMatchesRole(agent, RoleCustomer)` | false |
| `originMatchesRole(customer, RoleAgent)` | false |

### `internal/services/auth/service.go`（單元測試，mock UserRepo + mock stores）

| 方法 | 案例 | 預期 |
|------|------|------|
| `Login` | email 不存在 | `invalid_credentials` |
| `Login` | password 錯 | `invalid_credentials`、metric `LoginFailures` +1 |
| `Login` | 成功 | TokenPair；`refreshStore.Save` 被呼叫且 payload 正確 |
| `Register` | email 衝突 | `user_exists`（repo 上拋） |
| `Register` | 成功 | bcrypt cost 與 cfg 一致 |
| `Refresh` | `ErrTokenNotFound` | `invalid_token` |
| `Refresh` | `reused=true` | `familyRevoker.Revoke` 被呼叫、metric `TokenReusedTotal` +1、回 `token_reused` |
| `Refresh` | 正常 | 新 TokenPair、role 從 repo 取 |
| `Logout` | 正常 | `familyRevoker.Revoke(fid)` + `refreshStore.Revoke(token)` 都被呼叫 |
| `IssueWSTicket` | family revoked | `session_revoked`，不寫入 ticketStore |
| `IssueWSTicket` | role / origin 不符 | `origin_role_mismatch` |
| `IssueWSTicket` | 成功 | `ticketStore.Put` payload 為 `user\|...` |
| `IssueAnonymousWSTicket` | origin=agent | `origin_role_mismatch` |
| `IssueAnonymousWSTicket` | 成功 | payload 為 `anon\|...` |
| `ConsumeWSTicket` | `ErrTicketNotFound` | `ticket_invalid` |
| `ConsumeWSTicket` | origin 不符 | `ticket_origin_mismatch` |
| `ConsumeWSTicket` | ipHash 不符 | `ticket_ip_mismatch` |
| `ConsumeWSTicket` | user kind + family revoked | `session_revoked` |
| `ConsumeWSTicket` | user kind 成功 | `CustomerIdentity{UserID, FamilyID, Role}` |
| `ConsumeWSTicket` | anon kind 成功 | `CustomerIdentity{VisitorID, Role: ""}`，不查 family |
| `LinkVisitor` | 冪等 | 重複呼叫無錯 |

### `internal/handlers/auth.go`（`httptest` + mock service）

| 案例 | 預期 |
|------|------|
| `Login` 200 | response body 含 access/refresh |
| `Login` 401 | error body kind=`invalid_credentials` |
| `Register` 409 | kind=`user_exists` |
| `Register` 400 invalid email | kind=`validation_failed` |
| `Refresh` 200 | 同 Login |
| `Logout` 缺 context | kind=`missing_claims` |
| `AnonymousWSTicket` 無 cookie | 自動發新 cookie + 發 ticket |
| `AnonymousWSTicket` 無效 cookie | kind=`invalid_visitor` |
| `LinkVisitor` 缺 cookie | kind=`missing_visitor` |
| `claimsFromContext` 缺任一 key | kind=`missing_claims` |
