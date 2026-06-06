# 資料庫規格書

> PostgreSQL 16 schema 設計、Migration 管理與 sqlc 工作流程。

## Schema 設計

> TODO：各資料表的 DDL 定義待補。

### `users`

> TODO：欄位定義（id 型別、email 約束、password_hash、timestamps 等）。

---

## Migration 規範

- Migration 檔案放在 `db/migrations/`
- 命名格式：`{序號}_{說明}.up.sql` / `{序號}_{說明}.down.sql`
- 建議搭配 [golang-migrate](https://github.com/golang-migrate/migrate) 管理版本

---

## sqlc 規格

### 運作方式

```
db/queries/*.sql   ← 你寫 SQL
      │
      │  sqlc generate
      ▼
db/sqlc/*.go       ← 自動生成型別安全的 Go 程式碼（禁止手動修改）
      │
      ▼
internal/repository/*.go  ← 包裝 sqlc，實作 service interface
```

### sqlc.yaml 設定

```yaml
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

### SQL Query 定義（`db/queries/user.sql`）

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

### 生成的程式碼（`db/sqlc/user.sql.go`，自動產生）

```go
// 自動生成，禁止手動修改
func (q *Queries) GetUser(ctx context.Context, id string) (User, error)
func (q *Queries) CreateUser(ctx context.Context, arg CreateUserParams) (User, error)
func (q *Queries) UpdateUser(ctx context.Context, arg UpdateUserParams) (User, error)
```

### Repository 包裝 sqlc（`internal/repository/user.go`）

```go
// repository 負責：
// 1. 持有 sqlc *db.Queries
// 2. 實作 service 層定義的 UserRepository interface
// 3. 將 sqlc 生成的 struct 轉換為 models.User（隔離 sqlc 型別滲透）

type UserRepo struct {
    q *sqlc.Queries
}

func (r *UserRepo) FindByID(ctx context.Context, id string) (*models.User, error) {
    row, err := r.q.GetUser(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return nil, apperror.ErrNotFound
        }
        return nil, apperror.ErrInternal
    }
    return &models.User{
        ID:    row.ID,
        Email: row.Email,
    }, nil
}
```

### 新增查詢的流程（TDD）

1. 在 `db/queries/*.sql` 寫 SQL（先寫測試，確認查詢符合預期）
2. 執行 `sqlc generate` 產生 Go 程式碼
3. 在 `internal/repository/` 包裝生成的函式
4. 執行測試確認通過

### 常用指令

```bash
sqlc generate          # 根據 SQL 生成 Go 程式碼
sqlc vet               # 檢查 SQL 語法與設定
```

---

## 開發工具

```bash
# sqlc：SQL → Go 程式碼生成
brew install sqlc

# golang-migrate：DB migration 管理
brew install golang-migrate
```

---

## 相關文件

- [基礎架構規格](./backend-spec.md)
- [API 規格](./api-spec.md)
