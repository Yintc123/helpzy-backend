# 即時客服對話系統規格

> Helpzy 核心 feature：顧客與系統之間的即時對話。**預設由 AI 回應**，特定條件下移交真人客服接手。

## 系統概觀

```
┌──────────────────┐         ┌──────────────────┐
│  Customer Chat   │         │  Agent Dashboard │
│  (Next.js)       │         │  (Next.js)       │
└────────┬─────────┘         └────────┬─────────┘
         │ WebSocket                  │ WebSocket
         ▼                            ▼
   ┌──────────────────────────────────────────┐
   │              Go Backend                  │
   │                                          │
   │  handlers/ws         ← upgrade、訊息收發  │
   │      │                                   │
   │      ▼                                   │
   │  services/dispatcher ← 狀態機、路由       │
   │      │            │                      │
   │      ▼            ▼                      │
   │  services/llm   services/agent_queue     │
   │  (Registry)     (推給線上客服)            │
   │      │                                   │
   │      ▼                                   │
   │  repository/    ← 對話歷史、狀態          │
   └──────────────────────────────────────────┘
                       │
                       ▼
                  PostgreSQL 16
```

| 模組 | 文件 |
|------|------|
| LLM 抽象層（Provider / Registry） | [foundation/17-llm.md](./foundation/17-llm.md) |
| 認證契約（顧客 / 客服皆走 JWT） | [jwt-auth-spec.md](./jwt-auth-spec.md) |
| API 端點清單 | [api-spec.md](./api-spec.md) |

---

## 顧客身分模式

顧客端**支援未登入訪客直接開聊**，避免註冊摩擦。兩種身分共用同一套對話流程，差別只在識別欄位與 ticket 端點。

### 兩種身分

| 模式 | 識別 | Cookie | WS Ticket 端點 |
|------|------|--------|---------------|
| 已登入顧客 | `customer_id` (user_id) | Session cookie（`helpzy.sid`） | `POST /api/v1/auth/ws-ticket` |
| 匿名訪客 | `visitor_id`（32-byte opaque） | Visitor cookie（`helpzy.vid`） | `POST /api/v1/anonymous/ws-ticket` |

兩個 ticket 端點都產生**相同格式**的 WS upgrade ticket（見 [features/auth-feature-spec.md](./features/auth-feature-spec.md#issuewsticket已登入)）；差別只在 ticket payload 內：

```
登入：  {user_id}|{family_id}|customer|{ip_hash}
匿名：  visitor|{visitor_id}|customer|{ip_hash}
```

升級 handler 用同一個 consume 流程；anonymous 模式跳過 `family_revoked` 檢查（沒有 family）。

### 訪客模式生命週期

```
首次訪問 /chat
   │
   ▼
BFF Route Handler 偵測無 visitor cookie
   │
   ├─► 產生 helpzy.vid（HttpOnly，HMAC 簽名，TTL 30d）
   │
   ▼
BFF /api/chat/ws-ticket
   │
   ├─► 有 session → 走 customer ticket（既有流程）
   └─► 只有 visitor → 走 anonymous ticket（POST backend /api/v1/anonymous/ws-ticket）
   │
   ▼
Browser 拿 ticket → WS upgrade
   │
   ▼
Backend 建 conversation：visitor_id = "..."，customer_id = NULL
   │
   ▼
對話進行中...
   │
   ▼（可選）使用者註冊 / 登入
   │
   ▼
BFF /api/auth/register 或 login，request 同時帶 visitor cookie + 註冊資訊
   │
   ▼
Backend POST /api/v1/auth/link-visitor
   │
   ├─► 把 conversations WHERE visitor_id = X 全部改成 customer_id = new_user_id
   └─► 已斷的 WS 連線：client 重新申請 ticket（這次走 customer endpoint）
```

### 訪客 vs 登入的不變式

| 不變式 | 訪客 | 登入 |
|--------|------|------|
| 訊息歷史不會因身分轉換而遺失 | ✅ visitor 對話被綁定到 user_id | — |
| 一個 visitor cookie 可有多個對話 | ✅ | ✅ 同 user_id |
| 客服 dashboard 看得到匿名對話 | ✅ 顯示「訪客 #短碼」 | 顯示 email |
| 客服可接手匿名對話 | ✅ 流程相同 | — |
| Visitor cookie 過期 | 對話仍在 DB；新對話另起 visitor_id | — |
| 多裝置同步 | ❌（visitor cookie 不跨裝置） | ✅（同 user_id） |

詳細 visitor cookie 規格見 [features/auth-feature-spec.md](./features/auth-feature-spec.md#visitorid-業務型別)。

---

## 對話狀態機

每個 `conversation` 持有一個 `mode` 狀態：

```
        ┌─────────────────────────────────────────────┐
        │                                             │
        ▼                                             │
   ┌─────────┐  AI tool call ↑     ┌──────────────────┴──┐
   │   ai    │ ───────────────────▶│ pending_handoff     │
   └─────────┘  關鍵字 / 顧客主動    └──────────┬──────────┘
        ▲                                     │
        │ 客服「結束接手」                       │ 客服「接手」
        │                                     ▼
        │                            ┌─────────┐
        └────────────────────────────│  human  │
                                     └─────────┘
```

| 狀態 | 顧客訊息流向 | 客服可見 |
|------|------------|---------|
| `ai` | dispatcher → `services/llm` → 串回顧客 | 可旁觀 |
| `pending_handoff` | dispatcher → `agent_queue`（推給線上客服） | 出現在「待接手」清單 |
| `human` | dispatcher → assigned agent 的 WS | 接手的客服獨佔 |

### 轉換觸發條件

| 轉換 | 觸發來源 |
|------|---------|
| `ai → pending_handoff` | (a) LLM 呼叫 `escalate_to_human` tool；(b) 顧客關鍵字（白名單）；(c) 連續錯誤次數超過閾值 |
| `pending_handoff → human` | 客服在 dashboard 點「接手」 |
| `human → ai` | 客服點「結束接手」或工作階段逾時 |

### 並發控制

每個 conversation 在 server side 對應一個 **單一 goroutine + channel**（actor pattern）。所有狀態變更與訊息處理都序列化在這個 goroutine 內，避免並發 race condition。

### Session Lifecycle

Session goroutine 的生命週期跟著 conversation 的活躍度，不跟著任何單一 WS 連線：

```
首次顧客連線
     │
     ▼
┌─────────────────┐  雙方都在線         ┌─────────────────┐
│  Session 啟動    │ ──────────────────► │  Session active │ ◄──────┐
│  載入歷史 + mode │                     │  inbound 處理   │        │ 顧客/客服 重連
└─────────────────┘                     └────────┬────────┘        │
                                                 │                 │
                                       任一端離線（顧客或客服）       │
                                                 ▼                 │
                                        ┌─────────────────┐        │
                                        │  Session idle   │ ───────┘
                                        │  保留 in-memory │
                                        │  state          │
                                        └────────┬────────┘
                                                 │ idle 超過 SESSION_IDLE_TTL
                                                 │（預設 10 分鐘）
                                                 ▼
                                        ┌─────────────────┐
                                        │  Session 結束   │ Flush + GC
                                        │  state → DB     │
                                        └─────────────────┘
```

| 階段 | 觸發 | 動作 |
|------|------|------|
| **啟動** | 首次 WS upgrade 成功且 `dispatcher.AttachCustomer` / `AttachAgent` 找不到既有 session | 從 `repository.Load(convID)` 載入歷史、初始化 `Session{Mode, Customer, Agent}`、起 goroutine `run(ctx)` |
| **重連** | 升級時 dispatcher 找到既有 session | 不重起 goroutine；把新的 `*websocket.Conn` 接到 session 對應欄位（`Customer` 或 `Agent`） |
| **單端離線** | WS `Close` / `read error` | 對應欄位設為 `nil`、進 idle 狀態、啟動 idle timer；訊息仍可收（但會 buffer） |
| **雙端離線** | 兩端皆 nil | 維持 idle；若 AI 模式下仍有待回應的 stream，**繼續完成**（不丟掉 LLM 的回應）並寫入 DB |
| **GC** | idle 超過 `SESSION_IDLE_TTL`、或 `mode='human'` 且雙方離線超過 `CHAT_AGENT_QUEUE_TTL` | flush 任何 in-memory 暫存 → `repo.UpdateMode` → 關閉 inbound channel → goroutine 退出 |
| **重啟後續對話** | 顧客之後再連回同一 `conversation_id` | 走「啟動」流程，由 DB 重建 state |

#### 關鍵不變式

1. **訊息不會因為連線中斷而丟失**：所有訊息一律先 `repo.AppendMessage` 再廣播。
2. **Session 是 conversation 的 in-memory 投影**：對話的真實狀態永遠以 DB 為準；in-memory state 隨時可從 DB 重建。
3. **AI 回應不被斷線中斷**：streaming 進行中若顧客斷線，dispatcher 仍消費完 `events` channel 並寫入 DB；重連後顧客會看到完整訊息。

#### Dispatcher 管理表

```go
type Dispatcher struct {
    sessions sync.Map  // map[uuid.UUID]*Session
    // ...
}

func (d *Dispatcher) AttachCustomer(ctx, conn, claims, convID) error {
    // 1. LoadOrStore session
    sess, loaded := d.sessions.LoadOrStore(convID, ...)
    // 2. 接 conn → session.Customer
    // 3. 若 !loaded，啟動 goroutine
}
```

`sync.Map` 是 dispatcher 唯一的共享狀態；session 內部一律靠 channel 序列化。

---

## WebSocket 端點

| 端點 | 角色 | 認證 |
|------|------|------|
| `GET /api/v1/ws/customer?ticket=<...>&conversation_id=<id>` | 顧客連線 | WS Ticket（origin = customer） |
| `GET /api/v1/ws/agent?ticket=<...>` | 客服連線 | WS Ticket（origin = agent） |

### 為何不直接驗 JWT

前端走 **BFF 架構**（[frontend/bff-auth-spec](../../../frontend/docs/specs/bff-auth-spec.md)），JWT **不外洩給瀏覽器**；匿名訪客根本沒有 JWT。WS 升級階段一律用 **短期一次性 ticket**：

| 身分 | BFF 申請 ticket 來源 | Backend 端點 |
|------|---------------------|-------------|
| 已登入顧客 / 客服 | Session cookie + CSRF | `POST /api/v1/auth/ws-ticket` |
| 匿名訪客 | Visitor cookie + CSRF | `POST /api/v1/anonymous/ws-ticket` |

Browser 用拿到的 ticket 直連 `wss://backend/api/v1/ws/customer?ticket=...`。**WS upgrade handler 對兩種來源走同一個 consume 流程**，內部根據 ticket payload 的「身分標頭」（user_id 或 visitor）決定後續處理。

完整 ticket 規格見 [features/auth-feature-spec.md](./features/auth-feature-spec.md#consumewsticket)。

### 升級流程

1. 走標準 chi router；WS 升級路由**不掛 JWT middleware**
2. handler 先呼叫 `authSvc.ConsumeWSTicket(ctx, ticket, origin, peerIP)`：
   - GETDEL 原子消費 ticket
   - 驗證 `origin` 與端點匹配
   - 驗證 family 未撤銷
   - 驗證 IP fingerprint（若啟用）
3. 通過後呼叫 `coder/websocket` 的 `Accept()` 升級
4. 升級後，handler 將連線連同 `*auth.AccessClaims` 交給 `dispatcher.AttachCustomer(...)` 或 `dispatcher.AttachAgent(...)`
5. 後續訊息收發在 dispatcher 持有的 goroutine 內進行

### Handler 骨架

```go
// internal/handlers/ws.go
func (h *WSHandler) Customer(w http.ResponseWriter, r *http.Request) {
    ticket := r.URL.Query().Get("ticket")
    if ticket == "" {
        httpx.WriteError(w, r, apperror.Unauthorized("ticket_missing", "未授權"))
        return
    }

    // identity 可能是登入顧客（claims.Sub = user_id）或匿名訪客（claims.Sub = ""、claims.VisitorID 非空）
    identity, err := h.auth.ConsumeWSTicket(
        r.Context(), ticket, services.WSOriginCustomer, clientIP(r),
    )
    if err != nil {
        httpx.WriteError(w, r, err)
        return
    }

    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns: h.allowedOrigins,
    })
    if err != nil { return }

    convID := uuid.MustParse(r.URL.Query().Get("conversation_id"))
    h.dispatcher.AttachCustomer(r.Context(), conn, identity, convID)
}
```

> Ticket 一旦 Consume 成功即從 Redis 刪除；後續斷線重連必須**重新申請 ticket**。

### WS Handler 與 httpx 模式的差異

`pkg/httpx` 規範普通 handler「`return error` → 中央 `httpx.WriteError` 寫入 JSON 回應」。**WS upgrade handler 不能套用相同模式**，原因：

| 階段 | 是否能用 httpx |
|------|--------------|
| Ticket consume / origin 驗證 | ✅ 可用，**升級前**所有錯誤都走 `httpx.WriteError` |
| `websocket.Accept()` 之後 | ❌ TCP 已被接管，HTTP response writer 不可再寫 |
| 進入 session goroutine 後 | ❌ 改用 WS error envelope 傳給 client |

#### 標準寫法

```go
// internal/handlers/ws.go
func (h *WSHandler) Customer(w http.ResponseWriter, r *http.Request) error {
    // ── 升級前：用 httpx 慣例 ────────────────────────────
    ticket := r.URL.Query().Get("ticket")
    if ticket == "" {
        return apperror.Unauthorized("ticket_missing", "未授權")
    }
    claims, err := h.auth.ConsumeWSTicket(
        r.Context(), ticket, services.WSOriginCustomer, clientIP(r),
    )
    if err != nil { return err }

    convID, err := parseUUID(r.URL.Query().Get("conversation_id"))
    if err != nil { return err }

    // ── 升級：失敗只能 log，不再寫 HTTP response ───────────
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns: h.cfg.WS.AllowedOrigins,
    })
    if err != nil {
        logger.From(r.Context()).Warn("ws accept failed", "err", err)
        return nil // httpx 不再寫東西
    }

    // ── 升級後：交給 dispatcher；錯誤透過 WS envelope 回 client ──
    h.dispatcher.AttachCustomer(r.Context(), conn, claims, convID)
    return nil
}
```

要點：
1. **`websocket.Accept` 之後絕不 `return err`**，因為 httpx 中央寫入器會嘗試寫 HTTP body，污染已升級的 TCP 連線（client 會看到亂碼）
2. **`Accept` 失敗也 `return nil`**：此時 response 已被 `Accept` 內部嘗試寫過，再寫會 double-write
3. **升級後的錯誤一律走 WS envelope**：`dispatcher` 內收到的錯誤包成 `error` envelope 推給 client，再選擇是否關閉連線（`conn.Close(websocket.StatusInternalError, ...)`）

#### 路由註冊

```go
// cmd/server/main.go（節錄）
// WS 路由不掛 JWT middleware
r.Group(func(r chi.Router) {
    r.Use(middleware.Timeout(0)) // WS 不適用 request timeout
    r.Method("GET", "/api/v1/ws/customer", httpx.Handler(wsHandler.Customer))
    r.Method("GET", "/api/v1/ws/agent",    httpx.Handler(wsHandler.Agent))
})
```

注意 `Timeout(0)` 關掉預設的 request timeout —— WS 是長連線。其他 chi middleware（RequestID、Logger、Recoverer）仍可保留。

### 套件選擇

| 用途 | 套件 |
|------|------|
| WebSocket | [`github.com/coder/websocket`](https://github.com/coder/websocket)（前 nhooyr/websocket） |

**為什麼不用 Socket.IO？** Next.js 15 + Go 都原生支援 WS，多一層協定沒有實質好處。

---

## 訊息 Envelope

WebSocket 上的訊息一律為 JSON，結構統一：

```json
{
  "type": "user_message" | "assistant_delta" | "assistant_end"
        | "system_event" | "tool_use" | "handoff" | "error",
  "id": "msg_uuid",
  "conversation_id": "conv_uuid",
  "ts": "2026-06-07T10:00:00Z",
  "payload": { ... }
}
```

### 顧客→Server

```json
{ "type": "user_message", "payload": { "text": "我想退貨" } }
```

### Server→顧客（AI 模式 streaming）

```json
{ "type": "assistant_delta", "id": "msg_123", "payload": { "text_delta": "您好，" } }
{ "type": "assistant_delta", "id": "msg_123", "payload": { "text_delta": "請提供..." } }
{ "type": "assistant_end",   "id": "msg_123", "payload": { "stop_reason": "end_turn" } }
```

### Server→顧客（切換到 human）

```json
{ "type": "system_event", "payload": { "event": "handoff_pending", "queue_position": 2 } }
{ "type": "system_event", "payload": { "event": "agent_joined",   "agent_name": "Mia" } }
```

### Server→客服（dashboard）

```json
{ "type": "handoff", "payload": {
    "conversation_id": "conv_uuid",
    "customer_id": "user_uuid",
    "reason": "顧客要求退款金額超過 AI 權限",
    "history_preview": [...]
}}
```

---

## Canonical 訊息資料模型

對話歷史採用 **provider-neutral 的 canonical schema** 儲存，理由與設計細節見 [decisions/llm-provider-abstraction.md](../decisions/llm-provider-abstraction.md)。

### 資料表

```sql
CREATE TABLE conversations (
    id                UUID PRIMARY KEY,

    -- 顧客識別二擇一：登入顧客填 customer_id；匿名訪客填 visitor_id。
    -- 訪客註冊後執行 link-visitor：customer_id = 新 user_id、visitor_id 維持作為審計。
    customer_id       UUID,                         -- 已登入顧客的 user_id
    visitor_id        TEXT,                         -- 匿名 visitor cookie value（base64url）

    mode              TEXT NOT NULL
                      CHECK (mode IN ('ai','pending_handoff','human')),
    assigned_agent_id UUID,
    llm_provider      TEXT,     -- 預設用哪家（可為 NULL，由 registry fallback 決定）
    llm_model         TEXT,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT customer_or_visitor
        CHECK (customer_id IS NOT NULL OR visitor_id IS NOT NULL)
);

CREATE INDEX idx_conversations_customer_id ON conversations(customer_id) WHERE customer_id IS NOT NULL;
CREATE INDEX idx_conversations_visitor_id  ON conversations(visitor_id)  WHERE visitor_id IS NOT NULL;

CREATE TABLE messages (
    id              UUID PRIMARY KEY,
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    seq             BIGINT NOT NULL,                -- 對話內單調遞增（app 層產生）
    sender_type     TEXT NOT NULL
                    CHECK (sender_type IN
                           ('customer','assistant','agent','tool','system')),
    sender_id       UUID,                           -- agent 才填；其餘為 NULL

    content         JSONB NOT NULL,                 -- canonical block 陣列，見下

    llm_provider    TEXT,
    llm_model       TEXT,
    input_tokens    INT,
    output_tokens   INT,

    provider_meta   JSONB,                          -- stop_reason、cache 命中等不規格化資訊

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (conversation_id, seq)
);

CREATE INDEX idx_messages_conv_seq ON messages(conversation_id, seq);
```

### `content` JSONB 格式

一筆訊息可含多個 block（assistant turn 常同時有 text + tool_use）：

```json
[
  { "type": "text", "text": "好的，我幫您查詢訂單" },
  { "type": "tool_use", "id": "tu_01", "name": "lookup_order",
    "input": { "order_id": "A123" } }
]
```

Tool 結果為下一筆 `sender_type='tool'` 訊息：

```json
[
  { "type": "tool_result", "tool_use_id": "tu_01",
    "content": "{...}", "is_error": false }
]
```

對應 Go 型別等同 [foundation/17-llm.md](./foundation/17-llm.md#message) 的 `llm.ContentBlock`。**Repository 層認 `models.Message`，LLM service 認 `llm.Message`**，兩者由 `service` 層的純函式互轉。

---

## Handoff 觸發機制

### (a) AI Tool Call（主要）

定義 escalation tool 給 LLM：

```go
var EscalateToHuman = llm.Tool{
    Name:        "escalate_to_human",
    Description: "當你無法處理顧客請求時呼叫此工具，移交真人客服。",
    InputSchema: json.RawMessage(`{
        "type": "object",
        "properties": {
            "reason":    { "type": "string", "description": "需要真人接手的原因" },
            "category":  { "type": "string", "enum": ["refund","complaint","technical","other"] }
        },
        "required": ["reason"]
    }`),
}
```

當 dispatcher 從 stream 收到 `ToolUseEnd{Name: "escalate_to_human"}`：
1. `session.TransitionTo(ModePendingHandoff)`
2. `agent_queue.Enqueue(session, reason, category)`
3. 推 `system_event: handoff_pending` 給顧客

### (b) 顧客關鍵字（輔助）

config 可配置關鍵字白名單（如「找真人」、「客服」、「人工」），dispatcher 在送進 LLM 前先檢查，命中即直接觸發 handoff。

### (c) 客服主動接管

Agent dashboard 顯示所有 `mode = 'ai'` 對話，客服可直接點「接手」，跳過 `pending_handoff` 進入 `human` 模式。

---

## 服務骨架

### `internal/services/dispatcher`

```go
type Dispatcher struct {
    llm    *llm.Registry
    repo   ConversationRepository
    agents AgentQueue
    log    logger.Logger
}

type ConversationRepository interface {
    Load(ctx context.Context, id uuid.UUID) (*models.Conversation, []models.Message, error)
    AppendMessage(ctx context.Context, msg *models.Message) error
    UpdateMode(ctx context.Context, id uuid.UUID, mode models.ConversationMode) error
}

type AgentQueue interface {
    Enqueue(session *Session, reason string) error
    Assign(conversationID uuid.UUID, agentID uuid.UUID) error
}

// CustomerIdentity 統一已登入與匿名兩種來源，由 ConsumeWSTicket 回傳。
type CustomerIdentity struct {
    UserID    uuid.UUID  // 已登入時填；匿名為 uuid.Nil
    VisitorID string     // 匿名時填；已登入為空字串
}

func (c CustomerIdentity) IsAnonymous() bool { return c.UserID == uuid.Nil }

func (d *Dispatcher) AttachCustomer(ctx context.Context, conn *websocket.Conn, identity CustomerIdentity, convID uuid.UUID) error
func (d *Dispatcher) AttachAgent(ctx context.Context, conn *websocket.Conn, agentID uuid.UUID) error
```

### Session（per-conversation goroutine）

```go
type Session struct {
    ID         uuid.UUID
    Mode       Mode
    Customer   *websocket.Conn      // nil 表示顧客已離線
    Identity   CustomerIdentity     // 開啟對話時的顧客身分（不會變，但可能事後升級）
    Agent      *websocket.Conn      // 僅在 human 模式
    inbound    chan inboundMsg      // 全部訊息進這裡序列化；buffer 由 dispatcher 注入
}

// newSession 的 inboundBuf 來自 dispatcher.Config.InboundChanBuffer
// （上游為 cfg.Chat.InboundChanBuffer，預設 64）。
func newSession(id uuid.UUID, identity CustomerIdentity, mode Mode, inboundBuf int) *Session {
    return &Session{
        ID:       id,
        Identity: identity,
        Mode:     mode,
        inbound:  make(chan inboundMsg, inboundBuf),
    }
}

func (s *Session) run(ctx context.Context) { /* select { case msg := <-s.inbound: ... } */ }
```

> **inbound buffer 大小（`CHAT_INBOUND_CHAN_BUFFER`，預設 64）的取捨**：太小 → 高頻訊息時 producer 阻塞，影響 WS read 吞吐；太大 → 記憶體 footprint 隨 active session 數線性增長。64 對應「顧客快打字 + assistant streaming 同時進行」的典型 burst，幾乎不會卡。壓測再依實測調整。

### 訪客升級為登入帳號

`POST /api/v1/auth/link-visitor`（見 [api-spec.md](./api-spec.md)）執行後：

1. Backend SQL：`UPDATE conversations SET customer_id = :new_user_id WHERE visitor_id = :old_vid AND customer_id IS NULL`
2. **不通知** in-memory session — Identity 對 dispatcher 純資訊欄位，不影響任何邏輯
3. 客戶端會在登入流程中重新申請 ticket（這次走 `auth` endpoint），自然重連到既有 session

---

## 客服 Assignment 策略：FCFS

採用 **First-Come-First-Served + 客服自選**，不做自動指派：

| 行為 | 規則 |
|------|------|
| 待接手清單排序 | `pending_handoff` 時間最早的排最前面 |
| 指派方式 | 客服在 dashboard 點「接手」按鈕，**伺服器端 atomic claim** |
| 並發接手保護 | `UPDATE conversations SET assigned_agent_id = :me, mode = 'human' WHERE id = :id AND assigned_agent_id IS NULL` — 影響列數 = 0 表示被搶先 |
| 一個客服可同時接幾個 | 不設上限（由客服自行控管）；dashboard 顯示「我的進行中對話」計數 |
| 客服離線（WS 斷） | 對話不自動釋放；其他客服可手動「強制接手」（管理員權限）—**v1 不做，v2 再加** |
| 對話過 `CHAT_AGENT_QUEUE_TTL` 仍無人接手 | 自動回 AI 模式並通知顧客 |

> 不選 round-robin 或 skill-based 的原因見 [decisions/agent-assignment-fcfs.md](../decisions/agent-assignment-fcfs.md)。MVP 規模下 FCFS 簡單足夠。

---

## 跨 Provider 帶歷史的相容性

換 provider 不代表自動相容；以下情境必須有測試覆蓋：

1. 一段含 Claude 風格 `tool_use` 的歷史，餵給 OpenAI provider → 必須能轉成 `tool_calls` 格式
2. `tool_result` 的 `tool_use_id` 配對必須一致
3. System prompt 位置不同（Claude top-level / OpenAI 第一筆訊息）

**強制測試**：`internal/services/llm/cross_provider_compat_test.go`，固定一段 canonical 歷史，跑過每一個註冊 provider 的 `FromCanonical`，驗證皆能轉換且無資料遺失。

---

## 環境變數

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `CHAT_ESCALATION_KEYWORDS` | ✗ | `找真人,客服,人工` | 觸發 handoff 的關鍵字，逗號分隔 |
| `CHAT_AGENT_QUEUE_TTL` | ✗ | `300` | 待接手對話的逾時時間（秒） |
| `CHAT_MAX_HISTORY_TOKENS` | ✗ | `8000` | 餵給 LLM 的歷史最大 token 數（超過截斷舊訊息） |
| `CHAT_SESSION_IDLE_TTL` | ✗ | `600` | Session 雙端離線後 in-memory 保留時間（秒） |
| `WS_PING_INTERVAL` | ✗ | `30` | WebSocket 心跳（秒） |
| `WS_READ_TIMEOUT` | ✗ | `60` | 讀取超時（秒） |
| `WS_ALLOWED_ORIGINS` | production 必填 | — | WS 升級允許的 Origin（逗號分隔） |
| `VISITOR_COOKIE_SECRET` | ✓ | — | Visitor cookie HMAC secret，min 32 chars |
| `VISITOR_TTL` | ✗ | `2592000`（30 天） | Visitor cookie 有效期（秒） |

LLM 相關變數見 [foundation/17-llm.md](./foundation/17-llm.md#環境變數)。

---

## 測試策略

| 對象 | 方式 |
|------|------|
| `dispatcher` 狀態機 | 注入 mock LLM provider + mock repo，驗證每種觸發條件 |
| `dispatcher` 並發 | 單一 goroutine + channel，用 race detector 跑 `go test -race` |
| WebSocket handler | `httptest.NewServer` + `websocket.Dial`，驗證 envelope 收發 |
| Cross-provider 歷史 | 固定 canonical 歷史，遍歷所有 provider 的轉換 |
| 端到端 | E2E test：mock LLM 回 escalation tool → 驗證 conversation 轉到 `pending_handoff`，agent_queue 收到通知 |

---

## MVP 不做的事

- ❌ Redis pub/sub 跨 instance 廣播 — 單 instance 先跑通
- ❌ AI/人類 hybrid 草擬模式 — 純 AI ↔ Human 切換
- ❌ 多客服分群組 / SLA / 轉接特定客服 / 自動指派
- ❌ 對話加密（個資需求出現時再加 `pgcrypto` 或 app 層 AES-GCM）
- ❌ 多模型 routing / fallback decorator（先驗證單一 provider）
- ❌ AI 成本 / 濫用控制 — 暫靠 LLM provider 自身 rate limit + 既有 `RATE_LIMIT_AUTHED_PER_MIN`；用量起飛後再加 per-conversation token cap
- ❌ 客服「強制接手」(剝奪其他客服的對話) — v2
- ❌ 訪客跨裝置續聊 — Visitor cookie 不跨裝置；要跨需註冊

---

## 相關文件

- [foundation/17-llm.md](./foundation/17-llm.md) — LLM Provider 抽象
- [decisions/llm-provider-abstraction.md](../decisions/llm-provider-abstraction.md) — 抽象設計決策
- [jwt-auth-spec.md](./jwt-auth-spec.md) — 認證契約
- [api-spec.md](./api-spec.md) — API 端點清單
