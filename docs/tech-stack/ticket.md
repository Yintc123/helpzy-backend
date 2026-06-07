# Ticket（一次性短期令牌）

> **本文件聚焦在「ticket 是什麼、如何運作、本專案如何使用」。**
> 「為何選擇 ticket 模式而非 BFF 全程代理 WS」的選型理由見 [`decisions/websocket-ticket-pattern.md`](../../../frontend/docs/decisions/websocket-ticket-pattern.md)。

## 介紹

Ticket 是一種**短期、單次使用、不可解析**的令牌（opaque token），用來授權某個特定的「一次性敏感動作」，而**不暴露長期憑證**。常見別稱：one-time token、nonce、handover token、upgrade ticket。

### 為什麼不直接用長期憑證（如 JWT）？

JWT 雖然方便，但有幾個性質與「一次性短期動作」需求相衝：

| 性質 | JWT | 一次性動作需要的 |
|------|-----|----------------|
| 可撤銷性 | 必須等 `exp` 過期 / 額外維護黑名單 | 用完即立刻失效 |
| 是否可解析 | 是（payload 是 base64 JSON） | 不需要任何資訊，越 opaque 越好 |
| TTL | 通常 15 分鐘 ~ 數天 | 數十秒 |
| 使用次數 | 不限 | 嚴格一次 |
| 重放保護 | 弱（簽章對攻擊者已知） | 必須 |

把 JWT 拿來當一次性令牌使用，會付出「能解析」「不能立刻撤銷」這兩項代價，卻拿不到對應的好處。Ticket 模式以**「Redis 為單一真相」**的取捨換取上述所有性質。

### 標準產生 / 消費模式

**產生（Issue）：**
1. `crypto/rand` 產生 32 byte 隨機值 → `base64url` 編碼成 ~43 字元字串
2. 以該字串為 key 寫入 Redis：`SET ticket:{token} = "<payload>" EX <ttl>`
3. payload 通常包含「誰可以用」「對哪個對象有效」「綁定的 fingerprint」等資訊

**消費（Consume）：**
1. 對 Redis 執行 `GETDEL ticket:{token}` —— **原子操作**
2. 找不到 → 視為過期 / 已被使用 / 不存在，拒絕
3. 找到 → 解析 payload，做額外校驗（撤銷狀態、IP fingerprint、origin 等）

`GETDEL` 是關鍵。**不能拆成 `GET` + `DEL` 兩步**：兩個並發消費者可能都通過 `GET`，雙雙以為自己合法，破壞「單次」保證。

---

## 這個專案用在哪裡

目前單一用途：**WebSocket 升級票**——機制層 `pkg/auth/token.go` + `pkg/authstore/ticket.go`，政策層業務編排在 `internal/services/auth/`。

### 背景：為什麼 WS 升級需要 Ticket

前端走 [BFF 架構](../../../frontend/docs/specs/bff-auth-spec.md) —— 瀏覽器不持有 JWT，REST 走 BFF 代理時由 BFF 注入 `Authorization` header。但 **WebSocket 升級階段瀏覽器無法請 BFF 注入 header**（直連 backend 才有正常的 long-lived 連線體驗），勢必需要某個瀏覽器**可以**持有、又**不能濫用**的 credential。

Ticket 正好補上這個缺口：

```
Browser ──POST /api/chat/ws-ticket（cookie）──► BFF
                                                 │
                                       BFF 用 JWT 向 backend 換 ticket
                                                 ▼
                                            Go Backend
                                          發行 30s opaque ticket
              ◄────────────────────── { ticket }
Browser ──WSS /api/v1/ws/customer?ticket=...──► Go Backend（直連）
                                                 │
                                        GETDEL ws_ticket:{token}
                                        驗證 origin + IP + family
                                        通過 → Upgrade WS
```

### 元件對照

| 角色 | 路徑 | 責任 |
|------|------|------|
| 產生函式 | `pkg/auth/token.go` `GenerateOpaqueToken()` | 純函式：`crypto/rand` + `base64url`（泛用，不限 WS） |
| IP fingerprint | `pkg/auth/token.go` `HashIP(ip, secret)` | `sha256(ip + secret)`，避免明文 IP 入庫 |
| Store | `pkg/authstore/ticket.go` `TicketStore` | 機制層：`Put` / `Take`（GETDEL）對 Redis 操作，payload opaque |
| Service | `internal/services/auth/service.go` | 政策層：payload 編解碼、family revocation 檢查、origin 校驗 |
| Handler | `internal/handlers/auth.go` `WSTicket` | `POST /api/v1/auth/ws-ticket` 端點 |
| WS Handler | `internal/handlers/ws.go` | 升級前呼叫 `ConsumeWSTicket` |

### Redis schema

| Key 格式 | Value | TTL |
|---------|-------|-----|
| `ticket:{token}` | `user\|{user_id}\|{family_id}\|{origin}\|{ip_hash}` or `anon\|{visitor_id}\|{origin}\|{ip_hash}` | `WS_TICKET_TTL`（預設 30s） |

> 機制層 store 只看 opaque 字串；上表的結構由政策層編解碼。

詳見 [features/auth-feature-spec.md](../specs/features/auth-feature-spec.md#payload-編解碼) 與 [foundation/15-auth-mechanism.md](../specs/foundation/15-auth-mechanism.md#pkgauthstoretiketgo)。

### Issue / Consume 行為

**Issue（service 層概念碼）：**

```go
func (s *AuthService) IssueWSTicket(ctx, claims, origin, clientIP) (string, error) {
    // 1. family 是否已被撤銷（防止登出後仍能發 ticket）
    if revoked, _ := s.familyRevoker.IsRevoked(ctx, claims.Fid); revoked {
        return "", apperror.Unauthorized("session_revoked", "未授權")
    }
    // 2. origin 與 caller role 必須匹配
    if !originMatchesRole(origin, ParseRoleOrPanic(claims.Role)) {
        return "", apperror.Forbidden("origin_role_mismatch", "未授權")
    }
    // 3. 產生 opaque token + 編碼 payload + 寫機制層 store
    token, _ := auth.GenerateOpaqueToken()
    ipHash := auth.HashIP(clientIP, s.wsTicket.Secret)
    payload := encodeTicketPayload(ticketPayload{
        Kind: ticketKindUser, UserID: claims.Sub, FamilyID: claims.Fid,
        Origin: origin, IPHash: ipHash,
    })
    s.ticketStore.Put(ctx, token, payload)
    return token, nil
}
```

**Consume（機制層 `Take` 原子 GETDEL，政策層解碼 + 校驗）：**

```go
func (s *AuthService) ConsumeWSTicket(ctx, token, expectedOrigin, peerIP) (CustomerIdentity, error) {
    raw, err := s.ticketStore.Take(ctx, token)
    if errors.Is(err, authstore.ErrTicketNotFound) {
        return CustomerIdentity{}, apperror.Unauthorized("ticket_invalid", "未授權")
    }
    p, _ := decodeTicketPayload(raw)

    if p.Origin != expectedOrigin {
        return CustomerIdentity{}, apperror.Unauthorized("ticket_origin_mismatch", "未授權")
    }
    if p.IPHash != "" && auth.HashIP(peerIP, s.wsTicket.Secret) != p.IPHash {
        return CustomerIdentity{}, apperror.Unauthorized("ticket_ip_mismatch", "未授權")
    }
    if p.Kind == ticketKindUser {
        if revoked, _ := s.familyRevoker.IsRevoked(ctx, p.FamilyID); revoked {
            return CustomerIdentity{}, apperror.Unauthorized("session_revoked", "未授權")
        }
    }
    // ...組裝 CustomerIdentity 回傳（user 或 anon 分支）
}
```

---

## 為什麼使用這個技術

### 1. 立即可撤銷，不需要等過期

GETDEL 一執行，ticket 立刻從 Redis 消失。JWT 必須等 `exp` 或維護黑名單，與「單次使用」的語意天生不合。

### 2. 不暴露任何 user-facing 資訊

Ticket 是純隨機字串，沒有 base64 payload 可解。即使被中間人擷取，也無法得知是誰的 ticket、可以做什麼。

### 3. 攻擊面遠小於 JWT

| 攻擊類型 | JWT | Ticket |
|---------|-----|--------|
| `alg: none` 攻擊 | 必須防 | 不存在（沒有 alg） |
| Key confusion（HS / RS 換用） | 必須防 | 不存在（沒有簽章） |
| Payload tampering | 簽章防 | 不存在（沒有 payload） |
| Replay | 必須防（jti 黑名單） | GETDEL 天然防 |

### 4. 短，適合塞 URL query string

Base64url(32 bytes) = 43 字元，遠短於 JWT（通常數百字元）。WebSocket 升級用 query string 傳遞 credential 已是業界慣例，短 token 顯著降低 URL 長度、log 噪音與 referrer 洩漏風險。

### 5. 用 BFF + Ticket 同時保住兩個不變式

| 不變式 | 由誰維持 |
|-------|--------|
| 瀏覽器永遠不持有 JWT | BFF（REST proxy） |
| 瀏覽器不持有「能多次使用 / 可解析 / 長壽命」的 credential | Ticket 設計（30s + 單次 + opaque） |

只用 BFF 撐不住 WS（無法代理長連線）；只用 Ticket 撐不住長期 session（30s 太短）。兩者組合才完整。

### 6. 同樣模式可被未來其他短期動作沿用

Ticket pattern 是泛用的——機制層 `pkg/authstore/ticket.go` 的 `TicketStore` 已經是泛用 `Put` / `Take` API（payload opaque）。之後若有「下載私密檔案連結」「email 驗證」「magic link 登入」「跨服務 OTP」等需求，**直接複用同一個 `TicketStore`**（或為新場景另建一個獨立 namespace 的 store），各自 payload 編解碼於對應 service 層。

```
internal/services/
├── auth/                  # 現有：WS ticket、refresh、family revocation
├── download/              # 未來：私密下載連結（複用 TicketStore）
└── magic_link/            # 未來：免密登入（複用 TicketStore）
```

機制層的 `pkg/auth/token.go` `GenerateOpaqueToken()` 已是泛用 primitive，不需要為每個場景再抽一份。

---

## 注意事項

### 1. TTL 必須短

預設 30 秒。**不要為了「讓使用者多點時間連線」而把 TTL 拉長到分鐘級**——這會放大「被攔截後可重連」的窗口。若使用者真的點完按鈕 30 秒內還沒升 WS，他更該重來一遍而非重用舊 ticket。

### 2. 必須原子消費

實作 Repository 時務必用 `GETDEL`（Redis 6.2+）或等價 Lua script。**禁止**：

```go
// ✗ 錯誤示範
val, _ := rdb.Get(ctx, key).Result()
rdb.Del(ctx, key)
```

兩個並發 consume 可能都通過 `Get`，破壞單次保證。

### 3. 絕對不寫 log

完整 ticket 即使 30s TTL，落在 log 仍是攻擊面（log 通常保留數天）。需要追蹤時，僅 log：

- ticket 長度（驗證格式）
- 前 8 字元 + `***`（除錯標識）
- 對應的 `user_id` / `family_id`（這些已在 log context 內）

### 4. WS 必須走 wss

明文 ws 會讓 ticket 在 query string 被任何中間網路設備看到。**production 部署絕不可用 ws**。

### 5. 跨服務必須共享同一 Redis

Issue 與 Consume 必須對同一 Redis 實例（或同一 cluster），否則 GETDEL 找不到 key。Backend horizontal scaling 沒問題，但 Redis 必須為共享。

### 6. IP fingerprint 是雙面刃

- **加分**：擋掉 ticket 在不同網路位置被使用的情境
- **扣分**：mobile NAT 切換、雙網卡、企業 proxy 場景可能誤殺合法請求

可由 config `WS_TICKET_BIND_IP=false` 開關。**雲端 LB / 多層 CDN 環境建議關閉**，靠 30s TTL + 單次消費已足夠。

### 7. 與 family revocation 串接

Ticket 在發行後、消費前的 30 秒空窗可能遇到 family 被撤銷（reuse detected、登出）。Consume 流程**必須**檢查 `family_revoked:{family_id}`，否則撤銷的 session 仍可升 WS。

### 8. 不要重用同一個 ticket

斷線重連必須**重新申請新 ticket**。前端應拒絕將拿過一次的 ticket 寫入任何 storage / cache。

---

## 相關文件

- [foundation/15-auth-mechanism.md](../specs/foundation/15-auth-mechanism.md) — 機制層（store、純函式）
- [features/auth-feature-spec.md](../specs/features/auth-feature-spec.md) — 政策層（業務流程、payload 編解碼）
- [chat-spec.md](../specs/chat-spec.md) — WS 升級流程
- [frontend/ws-ticket-spec.md](../../../frontend/docs/specs/ws-ticket-spec.md) — Browser ↔ BFF 端契約
- [decision: websocket-ticket-pattern](../../../frontend/docs/decisions/websocket-ticket-pattern.md) — 為何採用此架構
