# Token Family（Refresh Token 鏈失竊偵測）

> **本文件聚焦在「Token Family 是什麼、如何運作、本專案如何使用」。**
> 政策層的觸發時機（登出 / 異常處置）見 [`features/auth-feature-spec.md`](../specs/features/auth-feature-spec.md)。

## 介紹

Token Family 是一種搭配 **refresh token rotation** 使用的安全機制，源自 OAuth 2.0 Security Best Practice（[RFC 6819 §5.2.2.3](https://datatracker.ietf.org/doc/html/rfc6819#section-5.2.2.3)），用來**偵測 refresh token 被盜並切斷整條 token 鏈**。

### 背景：純 rotation 為何不夠

Refresh token rotation 的基本流程：

```
登入：發 R1（refresh）+ A1（access）
A1 過期 → 用 R1 換 R2 + A2，R1 失效
A2 過期 → 用 R2 換 R3 + A3，R2 失效
...
```

問題：若攻擊者偷到 R1，**伺服器無法分辨**「合法使用者在換」還是「攻擊者拿被盜 token 在換」——兩者都會把 R1 拿來打 `/refresh`，兩者都會成功。

### Family 的解法

把同一次登入產出的所有 token 標記成同一個 `family_id`（UUID），舊 refresh token 被旋轉時**不直接刪除**，而是搬到 `refresh_used:` 區保留作為「已換過」的證據。

#### Rotation 流程

```
正常旋轉：
  refresh:R1 = "userID|F1"
  ── 用 R1 換 R2 ─→
  (刪除) refresh:R1
  (寫入) refresh_used:R1 = "userID|F1"   ← 證據區
  (寫入) refresh:R2 = "userID|F1"        ← 新有效 token
```

#### Reuse Detection

若稍後又有人用 R1 來換：

- `refresh:R1` 查不到（已失效）
- `refresh_used:R1` 查得到 → **reuse detected**

這是強訊號：**這條鏈不再可信**（不管是攻擊者拿被盜 token，或使用者拿了一份舊備份重打）。

#### 撤銷整條鏈

偵測到 reuse 時，撤銷整個 family：

```
SET family_revoked:F1 = "1" EX <refresh_ttl>
```

效果：
- 所有 refresh token（R3、R4...）失效（service 層在 rotate 前檢查）
- 所有 access token（A3、A4...）失效（middleware 在驗 token 後查 `family_revoked:{fid}`）

攻擊者跟合法使用者**都被踢出**。代價：使用者要重登一次；報酬：被盜的鏈 100% 切斷。

### Access Token 必須攜帶 `fid`

讓 access token 也能被 family 撤銷觸發，access token 的 claims 必須包含 `fid`：

```go
type AccessClaims struct {
    Sub  string
    Role string
    Jti  string
    Fid  string   // family_id
    ExpiresAt time.Time
}
```

JWT middleware 驗證流程的最後一步查 family 撤銷：

```go
claims := ParseAccess(token)
if revoked, _ := revoker.IsRevoked(ctx, claims.Fid); revoked {
    return 401
}
```

一個 family key 涵蓋整條鏈，**不需要列舉個別 jti**。

---

## 這個專案用在哪裡

> 以下路徑反映 [機制 / 政策分離](../specs/foundation/15-auth-mechanism.md) 的結構。

### 元件對照

| 角色 | 路徑 | 責任 |
|------|------|------|
| Family ID 產生 | `internal/services/auth/service.go` (`Login`) | 登入時產生新 UUID |
| Access token 帶 fid | `pkg/auth/jwt.go` `SignAccess(sub, role, familyID, ...)` | 純函式：寫入 JWT claims |
| Refresh rotation + reuse detection | `pkg/authstore/refresh.go` `RefreshStore.Rotate` | Lua script 原子完成 |
| Family 撤銷狀態 | `pkg/authstore/revoke.go` `FamilyRevoker` | `Revoke(familyID)` / `IsRevoked(familyID)` |
| Middleware 撤銷檢查 | `pkg/middleware/jwt.go` | 驗 token 後查 `IsRevoked(claims.Fid)` |
| 業務層觸發 | `internal/services/auth/service.go` | 登出 / reuse detected 時呼叫 `FamilyRevoker.Revoke` |

### Redis schema

| Key 格式 | Value | TTL | 用途 |
|---------|-------|-----|------|
| `refresh:{token}` | `userID\|familyID` | `JWT_REFRESH_TTL` | 有效的 refresh token |
| `refresh_used:{token}` | `userID\|familyID` | `JWT_REFRESH_TTL` | 已換過的 token，作為 reuse detection 證據 |
| `family_revoked:{familyID}` | `"1"` | `JWT_REFRESH_TTL` | family 已被撤銷 |

> 機制層只認 opaque `payload` 字串；上表中的 `userID|familyID` 結構由政策層編解碼。詳見 [`features/auth-feature-spec.md`](../specs/features/auth-feature-spec.md#payload-編解碼)。

### 觸發 family 撤銷的時機

1. **登出**（`POST /auth/logout`）→ revoke 當前 family
2. **Reuse detected**（rotate 時發現 refresh_used 有舊 token）→ revoke 對應 family
3. **未來**：管理員強制登出、異常活動偵測 → revoke 指定 family

具體流程見 [`features/auth-feature-spec.md`](../specs/features/auth-feature-spec.md)。

---

## 為什麼使用這個技術

### 1. 把「無法分辨」變成「可偵測訊號」

純 rotation 無法分辨合法 vs 被盜；family + reuse detection 將模糊地帶轉成明確訊號。一旦 reuse 出現，伺服器就知道有問題並可立即反應。

### 2. 一次撤銷涵蓋整條鏈

| 方案 | 撤銷單一 access | 撤銷未來會發出的 access | 撤銷 refresh |
|------|----------------|-----------------------|--------------|
| 單一 jti blacklist | ✅ | ❌（攻擊者繼續用 refresh 換新 access） | ❌ |
| Family revocation | ✅（middleware 查 fid） | ✅（新 access 也帶相同 fid） | ✅（rotate 前檢查） |

用「一個 key 涵蓋一條鏈」取代「列舉所有 jti」，**結構乾淨且 Redis key 數量遠少**。

### 3. 多裝置自然獨立

每次登入產生新 family：

```
手機登入 → F1
筆電登入 → F2
平板登入 → F3
```

- 手機按「登出」→ revoke F1，筆電 / 平板不受影響
- 偵測到 F2 被盜 → revoke F2，其他裝置正常
- 「登出所有裝置」→ revoke F1, F2, F3

不需要額外的 device tracking 表。

### 4. Stateless access + 可撤銷 兩全

Access token 仍是純 JWT（stateless 驗章），不需要每次打 Redis 查「這個 token 還有效嗎」。**只在 family 撤銷狀態做一次 Redis 查詢**（命中率高、O(1)）。

兼顧 JWT 的效能優勢與 server-side 可撤銷的安全性。

### 5. 攻擊面分析

| 攻擊情境 | 純 rotation | Family + reuse detection |
|---------|-------------|--------------------------|
| 攻擊者偷到 refresh | 可持續換新 token，難偵測 | 合法使用者下次 rotate 時觸發 reuse → 整條鏈撤銷 |
| 攻擊者偷到 access | 等過期才失效 | 對應 refresh chain reuse 時 access 也立即失效 |
| 攻擊者搶在使用者前 rotate | 帳號被綁架 | 仍會被偵測（使用者下次嘗試 rotate 觸發 reuse） |

Family **不能 100% 阻止攻擊**（無法阻止攻擊者搶先 rotate），但能**保證攻擊事件最終會被偵測**並提供撤銷工具。

---

## 注意事項

### 1. `refresh_used:` 的 TTL 必須涵蓋鏈內最長壽命 token

證據區必須保留到「鏈內所有 token 都過期」之後才能清除——否則攻擊者可以**等待證據過期**再重放舊 token。

實務做法：`refresh_used:` TTL = `JWT_REFRESH_TTL`（與 refresh token 同壽命）。`family_revoked:` 同理。

### 2. Rotate 必須原子

`Rotate` 操作包含：
1. `GET refresh:{old}`
2. 找到 → DEL old + SET refresh_used:old + SET refresh:new
3. 找不到 → 查 `refresh_used:{old}` 判斷是 reuse 還是純無效

若拆成多步 Redis 命令，並發 rotate 可能：
- 兩個都通過 GET → 兩邊都 SET 新 token → family 內出現兩條合法鏈
- 或 reuse detection 在 race 下漏報

實作必須用 **Lua script** 包成 atomic。

### 3. 機制只報告 reuse，不自動撤銷

機制層只負責「告訴你 reuse 發生了」，**不**自動 revoke。是否要 revoke、何時 revoke 是政策決定（例如可能想先寄通知再 revoke，或先觸發 step-up auth）。本專案的政策是：**reuse detected 立即 revoke + log warn + metric +1**。

### 4. Family revocation 不需要清除被撤銷的 token

`family_revoked:{fid}` 設好後，被撤銷的 refresh / access token 仍存在於 Redis / client 中（直到自然過期）。**這是刻意的**——既然 family 已撤銷，token 本身在不在不影響安全結果，且省下「列舉並刪除」的操作。

### 5. Family ID 不可猜測

`family_id` 必須是 UUID v4（128-bit random），確保攻擊者無法**枚舉 family** 探測撤銷狀態（`IsRevoked` 對任意 UUID 都回 false，攻擊者得不到資訊）。**禁止**用遞增 ID 或可猜測序列。

### 6. 不要在錯誤回應暴露 reuse 訊號

Reuse detected 時，回應給 client 的 error kind 用 `token_reused` 或一律統一成 `invalid_token`——**不要在 response message 細分為「你的 refresh 被偷了」**。攻擊者不該從錯誤訊息得知偵測機制細節。

### 7. Reuse 是強訊號，必須計入指標

`reuse detected` 應該：
- 計 metric（`auth.refresh.reused_total`）
- 達閾值告警 oncall（單一帳號短時間多次 reuse → 帳號可能被持續攻擊，應強制改密碼 / 鎖定）

詳見 [16-observability.md](../specs/foundation/16-observability.md)。

### 8. 單裝置 logout = 撤一個 family，不是全部

「登出當前裝置」= revoke 當前 family（單一 family）；不是 revoke 該使用者所有 token。多裝置情境下這是正確的 UX——不要為了「省事」一次撤所有。

### 9. Refresh TTL 越長，未偵測窗口越大

Family 能切斷已偵測的鏈，但**無法縮短「攻擊者偷到 token 到下次 rotate 之間的窗口」**。Refresh TTL 越長，這個窗口越大。本專案預設 30 天，是 web app 一般範圍；高安全需求（金融 / 健康資料）建議 7 天甚至更短。

### 10. Family 是機制概念，不是「使用者 session」

Family 描述的是「token rotation 鏈」，**不是**「使用者的登入 session」這種產品概念。一個使用者可同時有多個 family（多裝置），一個 family 不對應任何使用者可見的物件。產品層若要「session 列表」功能，要另外存資料表，不要用 family 當代理。

---

## 相關文件

- [foundation/15-auth-mechanism.md](../specs/foundation/15-auth-mechanism.md) — 機制層完整規格（含 `pkg/auth`、`pkg/authstore`、JWT middleware）
- [features/auth-feature-spec.md](../specs/features/auth-feature-spec.md) — 政策層（login / logout 觸發撤銷的時機與業務規則）
- [ticket.md](./ticket.md) — WS Ticket 機制；ticket consume 時會檢查 family 撤銷狀態
- [contextkey.md](./contextkey.md) — `claims.Fid` 注入 context 的方式
- [RFC 6819 §5.2.2.3](https://datatracker.ietf.org/doc/html/rfc6819#section-5.2.2.3) — OAuth 2.0 Security Best Practice 對 refresh rotation 的官方建議
