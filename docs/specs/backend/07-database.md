# 07 — PostgreSQL 與 Transaction Manager

## 連線池設定

直接接收 `config.DBConfig`（不另定義一份 struct），啟動失敗即 `return error` 讓 `main.go` 退出。

```go
// pkg/database/postgres.go
func New(c config.DBConfig) (*pgxpool.Pool, error) {
    url := fmt.Sprintf(
        "postgres://%s:%s@%s:%d/%s?sslmode=%s",
        c.User, c.Password, c.Host, c.Port, c.Name, c.SSLMode,
    )
    cfg, err := pgxpool.ParseConfig(url)
    if err != nil {
        return nil, fmt.Errorf("parse db url: %w", err)
    }
    cfg.MaxConns        = 20
    cfg.MinConns        = 5
    cfg.MaxConnLifetime = 1 * time.Hour
    cfg.MaxConnIdleTime = 30 * time.Minute

    pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
    if err != nil {
        return nil, fmt.Errorf("create pool: %w", err)
    }
    return pool, nil
}
```

**池參數**（目前 hardcoded，未來如需依環境調整再搬到 Config）：

| 參數 | 值 | 理由 |
|------|-----|------|
| `MaxConns` | 20 | 單實例上限；視 production 流量再調 |
| `MinConns` | 5 | 維持暖連線避免冷啟動 |
| `MaxConnLifetime` | 1h | 避免 DB 端 idle timeout |
| `MaxConnIdleTime` | 30min | 釋放閒置連線 |

> Migration 與 sqlc 規範見 [db-spec.md](./db-spec.md)。

---

## Transaction Manager（跨 Repository 共用交易）

### 目的

當一個 service 方法需要跨多個 repository 操作並維持 ACID（例：建立訂單同時扣庫存、發點數），不能讓每個 repo 各自從 pool 借連線——必須共用同一個 `pgx.Tx`。`TxManager` 提供這個能力，同時不讓 `pgx.Tx` 滲漏進 service 層。

### 設計：以 context 傳遞 tx

採用「ctx 攜帶 tx」模式（業界主流：Uber、Stripe Go 服務常見）：

- Service 呼叫 `txMgr.WithTx(ctx, fn)`，`fn` 收到的 `ctx` 已綁定 tx
- Repository 不感知 tx 存在，內部呼叫 `Executor(ctx)` 取得「tx 或 pool」
- 沒在交易內時自動 fallback 到 pool，正常查詢不受影響

### 實作

```go
// pkg/database/tx.go
type ctxKey struct{}

type TxManager struct {
    pool *pgxpool.Pool
}

func NewTxManager(pool *pgxpool.Pool) *TxManager {
    return &TxManager{pool: pool}
}

// WithTx 開啟交易，將綁定 tx 的 ctx 傳給 fn。
// fn 回傳非 nil error → rollback；回傳 nil → commit。
// 支援巢狀呼叫：若 ctx 內已有 tx 則直接複用，不重複 BEGIN。
func (m *TxManager) WithTx(ctx context.Context, fn func(ctx context.Context) error) error {
    if _, ok := ctx.Value(ctxKey{}).(pgx.Tx); ok {
        return fn(ctx) // 已在交易內，直接執行
    }

    tx, err := m.pool.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback(ctx) // commit 成功後 rollback 是 no-op

    ctxWithTx := context.WithValue(ctx, ctxKey{}, tx)
    if err := fn(ctxWithTx); err != nil {
        return err
    }
    return tx.Commit(ctx)
}

// DBTX 對應 sqlc 生成的 executor interface（Query / QueryRow / Exec）
type DBTX interface {
    Exec(context.Context, string, ...any) (pgconn.CommandTag, error)
    Query(context.Context, string, ...any) (pgx.Rows, error)
    QueryRow(context.Context, string, ...any) pgx.Row
}

// Executor 由 repository 呼叫，取得當前 ctx 的執行者
// — 若在交易內回傳 tx，否則回傳 pool。
func Executor(ctx context.Context, pool *pgxpool.Pool) DBTX {
    if tx, ok := ctx.Value(ctxKey{}).(pgx.Tx); ok {
        return tx
    }
    return pool
}
```

### Repository 使用方式

Repository **不直接持有 pool**，而是每次查詢透過 `Executor(ctx, pool)` 取得執行者：

```go
// internal/repository/user.go
type UserRepo struct {
    pool *pgxpool.Pool
}

func NewUserRepo(pool *pgxpool.Pool) *UserRepo {
    return &UserRepo{pool: pool}
}

func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    q := sqlc.New(database.Executor(ctx, r.pool))
    row, err := q.GetUser(ctx, id)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, apperror.NotFound("user_not_found", "找不到此使用者")
    }
    if err != nil {
        return nil, apperror.Internal(err)
    }
    return toModel(row), nil
}
```

### Service 使用方式

```go
// internal/services/order.go
type OrderService struct {
    orderRepo     OrderRepository
    inventoryRepo InventoryRepository
    pointRepo     PointRepository
    tx            *database.TxManager
}

func (s *OrderService) PlaceOrder(ctx context.Context, in PlaceOrderInput) error {
    return s.tx.WithTx(ctx, func(ctx context.Context) error {
        if err := s.inventoryRepo.Deduct(ctx, in.ProductID, in.Qty); err != nil {
            return err
        }
        if _, err := s.orderRepo.Create(ctx, in); err != nil {
            return err
        }
        return s.pointRepo.Grant(ctx, in.UserID, calcPoints(in))
    })
}
```

三個 repo 操作共用同一 `pgx.Tx`，任一步 `return err` 整個 rollback。

### 守則

- Repository **禁止** 直接呼叫 `pool.Begin()` / `pool.Query()`；一律透過 `Executor(ctx, pool)`
- Service 跨多個 repo 的寫入 → 必須包在 `WithTx` 內
- 單純讀取 / 單一 repo 寫入 → 不需 `WithTx`，直接呼叫即可（自動走 pool）
- `WithTx` 內禁止啟動 goroutine 並把同一 ctx 傳出去 — tx 不是 thread-safe

---

## 測試策略

- 以 `testcontainers` 啟動真實 Postgres
- 測案：
  - `WithTx` 中 fn 回傳 error → 資料未變更（rollback 生效）
  - `WithTx` 中 fn 回傳 nil → 資料寫入（commit 生效）
  - 巢狀 `WithTx` → 不重複 BEGIN，外層 commit 才真正寫入
  - 未呼叫 `WithTx` 的查詢 → 走 pool，正常運作
