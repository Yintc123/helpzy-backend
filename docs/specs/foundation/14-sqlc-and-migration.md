# 14 — SQL Migration 與 sqlc 工作流

## 目的

管理 PostgreSQL schema 演進與型別安全的 query 生成，是所有 repository 共用的基礎建設。

> Schema 設計（表 DDL、欄位約束）屬於業務規格，**不在本文件範圍**；本文件僅規範工具鏈與工作流。

---

## Migration 規範

- **檔案位置**：`db/migrations/`
- **命名格式**：`{序號}_{說明}.up.sql` / `{序號}_{說明}.down.sql`
- **工具**：[golang-migrate](https://github.com/golang-migrate/migrate)
- **工具選型決策**：見 [migration-golang-migrate](../../decisions/migration-golang-migrate.md)

### 範例

```
db/migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_add_user_nickname.up.sql
└── 000002_add_user_nickname.down.sql
```

### 守則

- 每個 migration 必須**可逆**：`up` 與 `down` 都要寫
- 不可改動已套用的 migration；要修正一律新增下一個序號
- production 部署前必須在 staging 跑過完整 up / down 循環

---

## sqlc 工作流

```
db/queries/*.sql              ← 手寫 SQL
      │
      │  sqlc generate
      ▼
db/sqlc/*.go                  ← 自動生成型別安全的 Go 程式碼（禁止手動修改）
      │
      ▼
internal/repository/*.go      ← 包裝 sqlc，實作 service interface
```

### sqlc.yaml 設定

```yaml
# backend/sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "db/queries"
    schema:  "db/migrations"
    gen:
      go:
        package:         "sqlc"
        out:             "db/sqlc"
        emit_json_tags:  true
        emit_interface:  true
        sql_driver:      "pgx/v5"
```

### SQL Query 定義範例（`db/queries/user.sql`）

```sql
-- name: GetUser :one
SELECT id, email, created_at FROM users WHERE id = $1;

-- name: CreateUser :one
INSERT INTO users (id, email, password_hash)
VALUES ($1, $2, $3)
RETURNING id, email, created_at;

-- name: UpdateUser :one
UPDATE users SET email = $2
WHERE id = $1
RETURNING id, email, created_at;
```

### 生成的程式碼簽章（`db/sqlc/user.sql.go`，自動產生）

```go
// 自動生成，禁止手動修改
func (q *Queries) GetUser(ctx context.Context, id string) (User, error)
func (q *Queries) CreateUser(ctx context.Context, arg CreateUserParams) (User, error)
func (q *Queries) UpdateUser(ctx context.Context, arg UpdateUserParams) (User, error)
```

---

## Repository 包裝 sqlc

Repository 層的責任：

1. 透過 [`database.Executor(ctx, pool)`](./07-database.md#repository-使用方式) 取得執行者（tx 或 pool）
2. 將 sqlc 生成的 struct 轉換為 `models.*`，**隔離 sqlc 型別滲透**至 service 層
3. 將 `pgx.ErrNoRows` 等底層錯誤轉為 `apperror.*`

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

// 隔離 sqlc 型別：sqlc.User → models.User
func toModel(row sqlc.User) *models.User {
    return &models.User{
        ID:    row.ID,
        Email: row.Email,
    }
}
```

---

## 新增查詢的 TDD 流程

1. 在 `internal/repository/*_test.go` 寫整合測試（testcontainers + 真實 Postgres），描述查詢應有的行為
2. 在 `db/queries/*.sql` 寫對應 SQL
3. 執行 `sqlc generate` 產生 Go 程式碼
4. 在 `internal/repository/` 包裝生成的函式（如上）
5. 執行測試，確認綠燈
6. 重構

---

## 常用指令

```bash
# Migration
migrate -path db/migrations -database "$DB_URL" up
migrate -path db/migrations -database "$DB_URL" down 1
migrate create -ext sql -dir db/migrations -seq create_users

# sqlc
sqlc generate          # 根據 SQL 生成 Go 程式碼
sqlc vet               # 檢查 SQL 語法與設定
```

---

## 開發工具安裝

```bash
brew install sqlc
brew install golang-migrate
```

---

## 測試策略

- `db/queries/*.sql`：以 `sqlc vet` 在 CI 驗證 SQL 語法
- `internal/repository/*_test.go`：整合測試，連接 testcontainers 啟動的真實 PostgreSQL
- 詳見 [13-testing.md](./13-testing.md)
