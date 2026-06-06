# DB Migration 決策：golang-migrate

**決定**：使用 `golang-migrate` 管理 PostgreSQL schema 版本。

## 採用的理由

1. 輕量，不綁定特定 ORM
2. 與 sqlc 的 schema 來源（`db/migrations/`）自然整合
3. 支援 up / down migration，可回滾

## 規範

- Migration 檔案放在 `db/migrations/`
- 命名格式：`{序號}_{說明}.up.sql` / `{序號}_{說明}.down.sql`
- sqlc 的 `schema` 來源指向 `db/migrations/`，兩者共用同一份 SQL

## 常用指令

```bash
# 執行所有 migration
migrate -path db/migrations -database $DATABASE_URL up

# 回滾一個版本
migrate -path db/migrations -database $DATABASE_URL down 1
```
