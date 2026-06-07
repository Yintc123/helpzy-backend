# 19 — WebSocket Foundation Primitives（`pkg/wsx`）

## 目的

`cfg.WS.{AllowedOrigins, PingInterval, ReadTimeout}` 屬於 foundation config，但在 foundation 層**沒有任何模組消化它們**——chat-spec 的 customer / agent handler 直接呼叫 `coder/websocket.Accept()`，每個 handler 都要重組 `AcceptOptions`、各自寫 heartbeat 與 close 邏輯，DRY 破口。

本文件規範 foundation 層的 WS 共用 primitives：

| 模組 | 責任 |
|------|------|
| `pkg/wsx/accept.go` | `Accept` 包裝：`AcceptOptions` 由 `cfg.WS` 注入、Origin 驗證、HTTP 1.1 / TLS 假設檢查 |
| `pkg/wsx/heartbeat.go` | Ping/Pong 心跳 loop helper（含 read deadline 推進） |
| `pkg/wsx/close.go` | 統一關閉 helper：normal close / error close / status code 對應 |
| `pkg/wsx/origin.go` | Origin 比對純函式（與 chat-spec 之外的元件共用） |

> **本文件僅規範通用 primitives**。Customer / Agent / 未來其他 WS endpoint 的 handler 流程、session 狀態機、訊息協定皆屬政策層，見 [chat-spec.md](../chat-spec.md)。

---

## Foundation vs Policy 邊界

| 層 | 模組 | 例子 |
|----|------|------|
| Foundation（本檔） | `pkg/wsx` | 升級握手、心跳、關閉、Origin 驗證 |
| Policy | `internal/handlers/ws.go` | `WSHandler.Customer` / `Agent`、ticket consume、轉交 dispatcher |
| Policy | `internal/services/dispatcher` | Session 狀態機、訊息路由、handoff |

**判準**：「換另一種 WS endpoint（例如未來的 `/api/v1/ws/admin-events`）會不會重用？」會 → foundation；不會 → policy。

---

## 設計原則

- **沒有業務知識**：`pkg/wsx` 不認 ticket、user_id、role、conversation——只認 `*http.Request` 與 `*websocket.Conn`
- **Config 透過 struct 注入**：不直接 import `internal/config`，避免層級反轉
- **錯誤一律走 `apperror`**：HTTP 階段（升級前）的錯誤可以被 `httpx.WriteError` 處理；升級後的錯誤回 sentinel 由 caller 決定如何 WS envelope
- **不啟 goroutine**：心跳 loop 由 caller 在 session goroutine 內驅動，`wsx` 只提供同步原語

---

## `pkg/wsx/accept.go`

```go
package wsx

import (
    "fmt"
    "net/http"
    "time"

    "github.com/coder/websocket"

    "helpzy/pkg/apperror"
)

// Config 由 main.go 從 cfg.WS 組裝後傳入，wsx 不直接 import config。
type Config struct {
    AllowedOrigins []string      // 精確比對；不支援萬用字元
    // PingInterval / ReadTimeout 由 Heartbeat 使用，集中於同一個 Config struct 方便 wiring
    PingInterval   time.Duration
    ReadTimeout    time.Duration
}

// Accept 包裝 websocket.Accept，套用一致的 AcceptOptions。
//
// 失敗情境：
//   - Origin 不在白名單 → 回 *apperror.AppError(403, kind=ws_origin_rejected)
//   - websocket.Accept 失敗（非 GET、缺 Upgrade header、HTTP 1.0 等） → 回 *apperror.AppError(400, kind=ws_upgrade_failed)
//
// 成功時 conn 已升級；caller 必須負責 conn.Close。
func Accept(w http.ResponseWriter, r *http.Request, cfg Config) (*websocket.Conn, error) {
    if !originAllowed(r.Header.Get("Origin"), cfg.AllowedOrigins) {
        return nil, apperror.Forbidden("ws_origin_rejected", "未授權的 Origin")
    }
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns:     cfg.AllowedOrigins,
        InsecureSkipVerify: false,
        // CompressionMode 預設 Disabled；CPU > 頻寬，且避免 CRIME-like 攻擊面
    })
    if err != nil {
        return nil, &apperror.AppError{
            Code:    http.StatusBadRequest,
            Kind:    "ws_upgrade_failed",
            Message: "WebSocket 升級失敗",
            Err:     fmt.Errorf("ws accept: %w", err),
        }
    }
    return conn, nil
}
```

### 為什麼自己驗一次 Origin，不全靠 `OriginPatterns`

`coder/websocket` 的 `OriginPatterns` 接受 glob（`*.example.com`）但本專案規格要求**精確比對**（[02-config.md](./02-config.md) `WS_ALLOWED_ORIGINS`）。在 Accept 前自己驗一次，可以：

1. 在錯誤訊息中區分「Origin 被拒」與「升級握手失敗」兩種情境，oncall 比較好 debug
2. 維持與 CORS / `ALLOWED_ORIGINS` 相同的精確比對語意（一致性 > glob 便利性）
3. 仍把白名單傳給 `OriginPatterns` 作為深度防禦——`coder/websocket` 在升級階段會二次檢查

### `originAllowed`（純函式）

```go
// pkg/wsx/origin.go
package wsx

import "strings"

func originAllowed(origin string, whitelist []string) bool {
    if origin == "" {
        // 非瀏覽器 client（curl、native app）可能無 Origin。
        // foundation 層採嚴格策略：缺 Origin → 拒。
        // 若未來需要放寬，由 caller 在進 wsx.Accept 前自行 short-circuit。
        return false
    }
    for _, allowed := range whitelist {
        if strings.EqualFold(origin, allowed) {
            return true
        }
    }
    return false
}
```

---

## `pkg/wsx/heartbeat.go`

```go
package wsx

import (
    "context"
    "errors"
    "time"

    "github.com/coder/websocket"
)

// Heartbeat 在 caller goroutine 內驅動 ping/pong 心跳。
// 每 cfg.PingInterval 送一次 ping，並依 cfg.ReadTimeout 推進 read deadline。
//
// 回傳：
//   - nil：ctx 被取消（正常關閉路徑）
//   - 其他 error：ping 失敗（連線已斷）；caller 應呼叫 Close 結束會話
//
// 設計：
//   - 不啟 goroutine，由 caller 決定執行位置（通常是 session goroutine 的 select 內）
//   - ReadTimeout 比 PingInterval 大，給 client 反應時間（Validate 已強制 ReadTimeout >= PingInterval）
func Heartbeat(ctx context.Context, conn *websocket.Conn, cfg Config) error {
    ticker := time.NewTicker(cfg.PingInterval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return nil
        case <-ticker.C:
            pingCtx, cancel := context.WithTimeout(ctx, cfg.ReadTimeout)
            err := conn.Ping(pingCtx)
            cancel()
            if err != nil {
                if errors.Is(err, context.Canceled) {
                    return nil
                }
                return err
            }
        }
    }
}
```

### 為什麼不在 wsx 內 spawn goroutine

`coder/websocket.Conn` 本身**支援單一 reader + 單一 writer 並行**，但兩個 writer 同時呼叫 `conn.Ping` / `conn.Write` 會 race。在 wsx 內 spawn 一個 ping goroutine、同時 caller 又有 write goroutine，會直接踩到。

讓 caller（dispatcher session loop）自己決定：

- 單 goroutine：在 select 中與其他事件競爭
- 多 goroutine：把 `Heartbeat` 放在「writer goroutine」內，與訊息送出共用 mutex

wsx 不替政策層做這個選擇。

---

## `pkg/wsx/close.go`

```go
package wsx

import (
    "github.com/coder/websocket"

    "helpzy/pkg/apperror"
)

// CloseNormal 以 1000 Normal Closure 關閉；用於 client 主動下線、session idle timeout 等正常路徑。
// reason 應為機器可讀短句（≤ 123 bytes，WS 規範限制）。
func CloseNormal(conn *websocket.Conn, reason string) error {
    return conn.Close(websocket.StatusNormalClosure, truncateReason(reason))
}

// CloseError 將 AppError 對應為 WS close status code。
//
//	401 / 403 → 4001 Unauthorized（自訂 4xxx）
//	429       → 1013 Try Again Later
//	5xx       → 1011 Internal Error
//	其他      → 1011 Internal Error
//
// 對應理由：標準 WS close code 1000–1015 留給協定本身；4000–4999 為應用層自訂，
// 供 client SDK 解析（例：4001 → 重新走 ticket 申請流程）。
func CloseError(conn *websocket.Conn, err error) error {
    code := websocket.StatusInternalError
    var ae *apperror.AppError
    if asAppErr(err, &ae) {
        switch {
        case ae.Code == 401 || ae.Code == 403:
            code = websocket.StatusCode(4001)
        case ae.Code == 429:
            code = websocket.StatusTryAgainLater
        case ae.Code >= 500:
            code = websocket.StatusInternalError
        }
        return conn.Close(code, truncateReason(ae.Kind))
    }
    return conn.Close(code, "internal_error")
}

func truncateReason(s string) string {
    // WS close reason 上限 123 bytes（規範）
    if len(s) > 123 {
        return s[:123]
    }
    return s
}

// asAppErr 是 errors.As 的 thin wrapper，純粹為了型別簽章可讀
func asAppErr(err error, target **apperror.AppError) bool {
    // 實作 = errors.As(err, target)；獨立提取以便測試替身
    // 此處省略；測試覆蓋見下方
    return apperror.AsAppError(err, target)
}
```

> `apperror.AsAppError` 由 [04-apperror.md](./04-apperror.md) 補一個小 helper 即可（純 `errors.As` 包裝），避免每處都寫 `var ae *AppError; errors.As(err, &ae)` 樣板。本檔不視為已存在；若 04 尚未補，wsx 內可直接 inline `errors.As`。

---

## Caller 使用範例（Customer WS handler）

下面是政策層 `internal/handlers/ws.go` 該寫的樣子——所有 WS 共用機制都從 `pkg/wsx` 取，handler 只剩業務流程：

```go
// internal/handlers/ws.go（政策層；本檔僅作示意，正式規格見 chat-spec.md）
package handlers

import (
    "helpzy/pkg/httpx"
    "helpzy/pkg/logger"
    "helpzy/pkg/wsx"
)

func (h *WSHandler) Customer(w http.ResponseWriter, r *http.Request) error {
    // ── 升級前：錯誤可以走 httpx.WriteError ─────────
    ticket := r.URL.Query().Get("ticket")
    convID := r.URL.Query().Get("conversation_id")

    identity, err := h.auth.ConsumeWSTicket(r.Context(), ticket, services.WSOriginCustomer, clientIP(r))
    if err != nil {
        return err
    }

    conn, err := wsx.Accept(w, r, h.wsCfg) // h.wsCfg 為 wsx.Config，由 main.go 注入
    if err != nil {
        return err
    }

    // ── 升級後：所有錯誤改走 wsx.CloseError 或 WS envelope ─────────
    defer wsx.CloseNormal(conn, "session_ended")

    if err := h.dispatcher.AttachCustomer(r.Context(), conn, identity, convID); err != nil {
        logger.From(r.Context()).Warn("attach customer failed", "err", err)
        _ = wsx.CloseError(conn, err)
        return nil // 升級後不再 return err，避免 httpx 嘗試寫 HTTP body
    }
    return nil
}
```

> 升級前 / 升級後的職責分工與 [chat-spec.md `## WS Handler 與 httpx 模式的差異`](../chat-spec.md#ws-handler-與-httpx-模式的差異) 一致；wsx 只是讓這個分工有現成 primitives 可用。

---

## Wiring（在 main.go）

`Phase 2` main.go 多一行組裝 `wsx.Config`，再注入給 WS handler：

```go
// cmd/server/main.go（Phase 2 + 補 wsx wiring）
wsCfg := wsx.Config{
    AllowedOrigins: cfg.WS.AllowedOrigins,
    PingInterval:   cfg.WS.PingInterval,
    ReadTimeout:    cfg.WS.ReadTimeout,
}
wsHandler := handlers.NewWSHandler(authService, dispatcher, wsCfg)
```

> 政策層 `WSHandler` 持有 `wsx.Config`、`AuthService`、`Dispatcher`——三者皆由 main.go 注入，handler 內不感知 foundation 細節。

---

## 環境變數

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `WS_ALLOWED_ORIGINS` | production 必填 | — | WS Origin 白名單（精確比對，逗號分隔） |
| `WS_PING_INTERVAL` | ✗ | `30` | Ping 間隔（秒） |
| `WS_READ_TIMEOUT` | ✗ | `60` | Read deadline（秒，>= `WS_PING_INTERVAL`） |

> 完整清單與 Validate 規則見 [02-config.md](./02-config.md)。

---

## 測試策略

### `pkg/wsx/origin_test.go`（純函式，table-driven）

| 案例 | 預期 |
|------|------|
| `originAllowed("", [...])` | false |
| `originAllowed("https://app.helpzy.com", ["https://app.helpzy.com"])` | true |
| `originAllowed("HTTPS://APP.HELPZY.COM", ["https://app.helpzy.com"])` | true（大小寫不敏感） |
| `originAllowed("https://attacker.com", ["https://app.helpzy.com"])` | false |
| `originAllowed("https://app.helpzy.com.attacker.com", ["https://app.helpzy.com"])` | false（精確比對，無前綴匹配） |

### `pkg/wsx/accept_test.go`（`httptest` + `coder/websocket` client）

| 案例 | 預期 |
|------|------|
| 合法 Origin + 標準 GET + Upgrade headers | 升級成功，回 non-nil `*Conn` |
| Origin 不在白名單 | 回 `*AppError(403, kind=ws_origin_rejected)`，未升級 |
| 缺 Origin header | 回 `*AppError(403, kind=ws_origin_rejected)` |
| 缺 Upgrade header（普通 GET） | 回 `*AppError(400, kind=ws_upgrade_failed)` |
| POST 而非 GET | 同上 |

### `pkg/wsx/heartbeat_test.go`

| 案例 | 預期 |
|------|------|
| ctx 在第一個 tick 前取消 | 立即回 nil |
| 正常 ping 數次後 ctx 取消 | 回 nil；測試端 client 收到 N 次 ping |
| Conn 在中途被對端關閉 | 下一次 ping 回 non-nil error |
| `PingInterval = 0` | （Config 已在 Validate 阻擋；本層 panic 由 caller 責任，不額外防禦） |

### `pkg/wsx/close_test.go`

| 案例 | 預期 close code / reason |
|------|------------------------|
| `CloseNormal(conn, "bye")` | 1000 / `"bye"` |
| `CloseError(conn, apperror.Unauthorized(...))` | 4001 / `unauthorized` |
| `CloseError(conn, apperror.TooManyRequests(...))` | 1013 / `too_many_requests` |
| `CloseError(conn, apperror.Internal(err))` | 1011 / `internal` |
| `CloseError(conn, errors.New("boom"))`（非 AppError） | 1011 / `internal_error` |
| reason > 123 bytes | 截斷至 123 bytes |

---

## 與其它規格的關係

| 規格 | 關聯 |
|------|------|
| [02-config.md](./02-config.md) | `WSConfig` 欄位、Validate 規則 |
| [04-apperror.md](./04-apperror.md) | `CloseError` 對應 AppError → WS close code |
| [09-middleware.md](./09-middleware.md) | WS 路由群組排除 `Timeout` middleware |
| [12-startup.md](./12-startup.md) | Phase 2 main.go 注入 `wsx.Config` |
| [chat-spec.md](../chat-spec.md) | 政策層 handler / dispatcher 使用 wsx primitives |
