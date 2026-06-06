# 08 — Redis 規格

## 連線設定

```go
// pkg/cache/redis.go
func New(url string) (*redis.Client, error) {
    opt, err := redis.ParseURL(url)
    if err != nil {
        return nil, fmt.Errorf("parse redis url: %w", err)
    }
    return redis.NewClient(opt), nil
}
```

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
