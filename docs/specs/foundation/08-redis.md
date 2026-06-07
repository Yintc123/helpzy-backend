# 08 — Redis 規格

## 連線設定

```go
// pkg/cache/redis.go
import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func New(url string) (*redis.Client, error) {
    opt, err := redis.ParseURL(url)
    if err != nil {
        return nil, fmt.Errorf("parse redis url: %w", err)
    }
    rdb := redis.NewClient(opt)

    // go-redis 是 lazy connect — 與 pgxpool 同理，啟動階段同步 Ping 一次：
    //   - 設定錯（URL 對但目標不通 / auth 失敗）能在啟動就 fail-fast
    //   - 避免「server 已 listen，但第一個 auth 請求才爆 Redis」這種錯位事件
    pingCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := rdb.Ping(pingCtx).Err(); err != nil {
        _ = rdb.Close()
        return nil, fmt.Errorf("ping redis: %w", err)
    }
    return rdb, nil
}
```

> 5 秒 timeout 為啟動專用常數（不抽到 config）；不通就快速失敗，不留模糊空間。Runtime 的 `/readyz` 走 `cfg.App.HealthPingTimeout`，兩個 timeout 互不相干。

---

## Pinger Adapter

`pgxpool.Pool.Ping(ctx) error` 已符合 `Pinger` interface（見 [10-health.md](./10-health.md)）。Redis 客戶端 `Ping(ctx) *StatusCmd` 簽章不符，包一層 adapter：

```go
// pkg/cache/redis.go
type RedisPinger struct{ *redis.Client }

func (p RedisPinger) Ping(ctx context.Context) error {
    return p.Client.Ping(ctx).Err()
}
```

main.go 使用：

```go
healthHandler := handlers.NewHealthHandler(pool, cache.RedisPinger{Client: rdb})
```

---

## 環境變數

| 變數 | 必填 | 說明 |
|------|------|------|
| `REDIS_URL` | ✓ | Redis 連線字串（如 `redis://localhost:6379`） |

---

## 測試策略

連線層極薄；以**單元測試**覆蓋 URL parse 失敗路徑、**整合測試**覆蓋 `Ping` 成功 / 失敗與 `RedisPinger`。

### `pkg/cache/redis_test.go`（單元，純解析錯誤路徑）

URL parse 階段不需要連線，無 Redis 即可測：

| 案例 | 預期 |
|------|------|
| `New("not-a-url")` | err 非 nil，訊息以 `parse redis url:` 開頭 |
| `New("")` | err 非 nil（空字串被視為無效） |
| `New("http://localhost")` | err 非 nil（scheme 錯誤） |

### `pkg/cache/redis_integration_test.go`（整合，build tag `integration`）

`New` 內已含 startup ping，連線成功 / 失敗都要實打：

| 案例 | 預期 |
|------|------|
| `New(<testcontainers URL>)` | 回有效 `*redis.Client`，err = nil |
| `New("redis://127.0.0.1:1")` 對沒人 listen 的 port | err 非 nil，訊息含 `ping redis:` |
| `New("redis://wrong-pass@<container>")` | err 非 nil（auth 失敗） |
| `RedisPinger.Ping(ctx)` 對已關閉的 client | err 非 nil（`redis: client is closed`） |
| `RedisPinger.Ping(ctx)` 對短 timeout ctx | err = `context.DeadlineExceeded` |

整合測試走 `testcontainers`，與 [13-testing.md](./13-testing.md) 的 Redis singleton 共用。

### 不在本層測

- Repository 層的 Redis 操作（refresh / revoke / ws-ticket）由各自 `*_test.go` 用 `testcontainers` 覆蓋，不在 `pkg/cache` 重複
- OTel instrument 行為由 [16-observability.md](./16-observability.md) 的整合測試驗
