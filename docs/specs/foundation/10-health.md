# 10 — 健康檢查（Health Check）

## 端點

| 端點 | 用途 | 檢查項目 |
|------|------|---------|
| `GET /healthz` | Liveness probe | 程序是否存活；**不**檢查外部依賴 |
| `GET /readyz` | Readiness probe | 是否能服務請求；檢查 DB、Redis 連線 |

## 為什麼分兩個

- **Liveness 失敗** → k8s 重啟 pod；不該因為下游短暫故障而連鎖重啟
- **Readiness 失敗** → k8s 從 service 摘除流量，但不重啟；下游恢復後自動加回
- 兩者混用會導致「Redis 抖動 → 全部 pod 同時重啟 → 雪崩」

---

## 實作

依賴透過 `Pinger` interface 注入，避免 `HealthHandler` 直接耦合 `*pgxpool.Pool` / `*redis.Client`，便於測試以 mock 替換。

```go
// internal/handlers/health.go

// Pinger 抽象任何「能 ping 確認可用」的依賴。
// pgxpool.Pool 原生符合；redis.Client 用 adapter 包一層。
type Pinger interface {
    Ping(ctx context.Context) error
}

type HealthHandler struct {
    db          Pinger
    cache       Pinger
    pingTimeout time.Duration // 由 cfg.App.HealthPingTimeout 注入
}

func NewHealthHandler(db, cache Pinger, pingTimeout time.Duration) *HealthHandler {
    return &HealthHandler{db: db, cache: cache, pingTimeout: pingTimeout}
}

func (h *HealthHandler) Live(w http.ResponseWriter, r *http.Request) error {
    return response.JSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func (h *HealthHandler) Ready(w http.ResponseWriter, r *http.Request) error {
    ctx, cancel := context.WithTimeout(r.Context(), h.pingTimeout)
    defer cancel()

    if err := h.db.Ping(ctx); err != nil {
        return apperror.Internal(fmt.Errorf("db ping: %w", err))
    }
    if err := h.cache.Ping(ctx); err != nil {
        return apperror.Internal(fmt.Errorf("redis ping: %w", err))
    }
    return response.JSON(w, http.StatusOK, map[string]string{"status": "ready"})
}
```

> **為什麼 `pingTimeout` 由 constructor 注入而非每次呼叫帶入**：系統常數（不會隨呼叫變動），constructor 注入符合既有慣例（同 `RefreshStore.ttl`、`FamilyRevoker.ttl`、`TicketStore.ttl`）。實作環境的調校只發生在啟動階段。

---

## 路由註冊

兩個端點**註冊在 JWT middleware 之外**（k8s 探針不會帶 token）：

```go
httpx.Get(r, "/healthz", healthHandler.Live)
httpx.Get(r, "/readyz",  healthHandler.Ready)
```

---

## 測試策略

- `Live`：純函式，斷言 200 + `{"data":{"status":"ok"}}`
- `Ready`：以 mock `Pinger` interface 分別模擬 ping 成功 / 失敗，斷言狀態碼
- `Ready` timeout：mock `Pinger.Ping` 內 `time.Sleep(pingTimeout * 2)`，斷言回傳 `context.DeadlineExceeded` 包成 500
- `Ready` 採用注入的 `pingTimeout`：以不同 ctor 值（`50ms` / `500ms`）跑同一案例驗證 timeout 確實生效（非寫死 2s）
