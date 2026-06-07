# API 規格書

> Backend 提供給 Next.js 前端的 RESTful API 規格。

## 路由設計

### 結構

```go
// cmd/server/main.go
r := chi.NewRouter()

// 全域 Middleware
r.Use(middleware.Logger)
r.Use(middleware.CORS)
r.Use(middleware.Recoverer)

// 公開路由（不需要 JWT）
r.Get("/api/health", handlers.HealthCheck)
r.Post("/api/v1/auth/login", userHandler.Login)

// 受保護路由（需要 JWT）
r.Group(func(r chi.Router) {
    r.Use(jwtMiddleware.Verify)
    r.Use(middleware.Timeout(cfg.App.RequestTimeout))

    r.Get("/api/v1/users/{id}", userHandler.GetUser)
    r.Put("/api/v1/users/{id}", userHandler.UpdateUser)
})
```

### 路由清單

| Method | Path | JWT | 說明 |
|--------|------|-----|------|
| GET | `/api/health` | 否 | 健康檢查 |
| POST | `/api/v1/auth/register` | 否 | 註冊 |
| POST | `/api/v1/auth/login` | 否 | 登入 |
| POST | `/api/v1/auth/refresh` | 否 | 刷新 access token（rotation） |
| POST | `/api/v1/auth/logout` | **是** | 登出（撤銷 access + refresh） |
| POST | `/api/v1/auth/ws-ticket` | **是** | 發行 WebSocket 一次性升級票（登入顧客 / 客服） |
| POST | `/api/v1/auth/link-visitor` | **是** | 將 visitor cookie 對應的對話綁定到當前 user_id |
| POST | `/api/v1/anonymous/ws-ticket` | **Visitor Cookie** | 發行匿名訪客的 WS 升級票 |
| GET | `/api/v1/users/{id}` | **是** | 取得使用者資料 |
| PUT | `/api/v1/users/{id}` | **是** | 更新使用者資料 |
| POST | `/api/v1/conversations` | **是** | 建立新對話（顧客） |
| GET | `/api/v1/conversations/{id}` | **是** | 取得對話歷史 |
| GET | `/api/v1/conversations/{id}/messages` | **是** | 取得訊息分頁 |
| POST | `/api/v1/conversations/{id}/takeover` | **是**（agent） | 客服接手對話 |
| POST | `/api/v1/conversations/{id}/release` | **是**（agent） | 客服結束接手 |
| GET | `/api/v1/ws/customer` | **Ticket** | WebSocket 升級（顧客）— 見下 |
| GET | `/api/v1/ws/agent` | **Ticket** | WebSocket 升級（客服）— 見下 |

---

## 端點規格

> TODO：各端點的 request body、response body、欄位驗證、錯誤碼對照待定義。

### `POST /api/v1/auth/login`

- Request body: TODO
- Response body: TODO
- 錯誤碼: TODO

### `GET /api/v1/users/{id}`

- Path param: `id` — TODO（UUID? 數字?）
- Response body: TODO
- 錯誤碼: TODO

### `PUT /api/v1/users/{id}`

- Path param: `id` — TODO
- Request body: TODO（可更新欄位、驗證規則）
- Response body: TODO
- 錯誤碼: TODO

---

---

## WebSocket Ticket 端點

### `POST /api/v1/auth/ws-ticket`

發行一張短期一次性 ticket，給已登入使用者（顧客或客服）升級 WebSocket 使用。完整規格見 [features/auth-feature-spec.md](./features/auth-feature-spec.md#issuewsticket已登入)。

- Auth: JWT
- Request body:
  ```json
  { "origin": "customer" | "agent" }
  ```
- Response:
  ```json
  { "ticket": "<43字元 base64url>", "expires_in": 30 }
  ```
- 錯誤：
  - 403 `origin_role_mismatch` — JWT role 與請求 origin 不符（例：customer 角色申請 agent ticket）
  - 401 `session_revoked` — 該 session 已被撤銷

### `POST /api/v1/anonymous/ws-ticket`

訪客版本的 ws-ticket。**不掛 JWT middleware**；從 visitor cookie 解出身分。

- Auth: Visitor Cookie（`helpzy.vid`，HMAC 簽章），見 [features/auth-feature-spec.md](./features/auth-feature-spec.md#visitorid-業務型別)
- Request body:
  ```json
  { "origin": "customer" }
  ```
- Response: 同 `/api/v1/auth/ws-ticket`
- 錯誤：
  - 401 `invalid_visitor` — visitor cookie 缺漏 / 簽章不符
  - 403 `origin_role_mismatch` — body 帶 `agent`（訪客不可申請）

### `POST /api/v1/auth/link-visitor`

把 visitor cookie 對應的所有對話綁定到當前已登入使用者。註冊或登入流程結束時呼叫一次。

- Auth: JWT
- Request: 需同時帶 visitor cookie（HMAC 驗證後取 VisitorID）
- Response: `204 No Content`
- 行為：`UPDATE conversations SET customer_id = :user_id WHERE visitor_id = :vid AND customer_id IS NULL`
- 冪等：重複呼叫無副作用
- 錯誤：
  - 401 `invalid_visitor` — visitor cookie 簽章不符
  - 200 / 204 — 即使沒有任何列被更新（visitor 沒對話 / 已 link 過）也算成功

---

## WebSocket 端點

> 完整訊息 envelope、狀態機與 handoff 流程見 [chat-spec.md](./chat-spec.md)。

### `GET /api/v1/ws/customer`

- Query:
  - `ticket` — 一次性 ticket（由 `/api/v1/auth/ws-ticket` 取得）
  - `conversation_id` — 對話 UUID（缺則由 server 開新對話）
- 升級流程：
  1. handler 呼叫 `auth.ConsumeWSTicket(ticket, origin=customer, peerIP)`
  2. 通過後 `websocket.Accept()` 升級
  3. 連線交給 `dispatcher.AttachCustomer(...)`
- 訊息格式：JSON envelope，見 [chat-spec.md#訊息-envelope](./chat-spec.md#訊息-envelope)
- 斷線重連：必須**重新申請 ticket**，不可重用

### `GET /api/v1/ws/agent`

- Query: `ticket` — 一次性 ticket（origin = agent）
- 同上升級流程，交給 `dispatcher.AttachAgent(...)`
- 客服一條連線可同時收多筆對話的事件（透過 envelope 內 `conversation_id` 區分）

---

## 相關文件

- [Backend Foundation 規格](./backend-foundation-spec.md)
- [JWT 跨服務契約](./jwt-auth-spec.md)
- [即時客服對話系統](./chat-spec.md)
