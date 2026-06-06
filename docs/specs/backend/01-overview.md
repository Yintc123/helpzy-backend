# 01 — 專案概覽

## 相關規格文件索引

見 [backend-spec.md](../backend-spec.md)。

---

## 技術棧

| 項目 | 技術 | 說明 |
|------|------|------|
| 語言 | Go 1.23 | |
| HTTP Router | `chi` v5 | 輕量、stdlib 相容、Clean Architecture 友善 |
| 資料庫 | PostgreSQL 16 | 主要資料儲存 |
| DB 驅動 | `pgx` v5 | PostgreSQL 官方推薦驅動，支援連線池 |
| ORM | `sqlc` | SQL → 型別安全 Go 程式碼生成 |
| 快取 | Redis 7 | Session JWT 儲存、快取層 |
| Redis 客戶端 | `go-redis` v9 | |
| JWT | `golang-jwt/jwt` v5 | 驗證 BFF 傳入的 JWT |
| 容器化 | Docker + Docker Compose | |

---

## 架構：Clean Architecture

```
┌─────────────────────────────────────────┐
│           Frameworks & Drivers          │  ← chi、pgx、sqlc、go-redis、Docker
├─────────────────────────────────────────┤
│          Interface Adapters             │  ← handlers、repository 實作
├─────────────────────────────────────────┤
│              Use Cases                  │  ← services（商業邏輯）
├─────────────────────────────────────────┤
│               Entities                  │  ← models（純 Go struct）
└─────────────────────────────────────────┘
```

**原則：**
- 內層不依賴外層
- 每一層透過 interface 溝通
- 框架型別（chi、pgx、sqlc 生成碼）只出現在最外層
- `context.Context` 從 handler 貫穿至 repository

---

## 專案結構

```
backend/
├── cmd/
│   └── server/
│       └── main.go               # 進入點：組裝依賴、啟動伺服器、優雅關閉
├── internal/
│   ├── config/                   # 設定管理：env 載入、helper、驗證
│   │   ├── config.go
│   │   └── config_test.go
│   ├── handlers/                 # Interface Adapters：處理 HTTP 請求與回應
│   │   ├── user.go
│   │   ├── user_test.go
│   │   ├── health.go             # /healthz、/readyz 端點
│   │   └── health_test.go
│   ├── services/                 # Use Cases：商業邏輯
│   │   ├── user.go
│   │   └── user_test.go
│   ├── repository/               # Interface Adapters：包裝 sqlc，實作 service 所需的 interface
│   │   ├── user.go
│   │   └── user_test.go
│   └── models/                   # Entities：純業務用資料結構（與 sqlc 生成的 struct 分離）
│       └── user.go
├── db/
│   ├── migrations/               # SQL Migration 檔案
│   │   ├── 000001_create_users.up.sql
│   │   └── 000001_create_users.down.sql
│   ├── queries/                  # sqlc 來源：SQL query 定義檔
│   │   └── user.sql
│   └── sqlc/                     # sqlc 自動生成（禁止手動修改）
│       ├── db.go
│       ├── models.go
│       └── user.sql.go
├── pkg/
│   ├── middleware/               # HTTP Middleware
│   │   ├── jwt.go
│   │   ├── cors.go
│   │   ├── logger.go
│   │   ├── requestid.go
│   │   ├── recoverer.go
│   │   └── timeout.go
│   ├── auth/
│   │   └── jwt.go               # JWT 解析與驗證（純函式）
│   ├── database/
│   │   ├── postgres.go          # PostgreSQL 連線池初始化
│   │   ├── tx.go                # TxManager：跨 repository 共用交易
│   │   └── tx_test.go
│   ├── cache/
│   │   └── redis.go             # Redis 連線初始化
│   ├── logger/
│   │   ├── logger.go            # 結構化 logger 初始化與 context 注入
│   │   └── logger_test.go
│   ├── validator/
│   │   ├── validator.go         # validator/v10 封裝與錯誤轉換
│   │   └── validator_test.go
│   ├── apperror/
│   │   ├── error.go             # 統一錯誤型別與 constructors
│   │   └── error_test.go
│   ├── httpx/
│   │   ├── handler.go           # 自訂 HandlerFunc：回傳 error → 中央寫入器
│   │   └── handler_test.go
│   └── response/
│       └── response.go          # 統一回應格式
├── sqlc.yaml                     # sqlc 設定檔
├── configs/
│   └── .env.example
├── docs/
├── Dockerfile
└── go.mod
```

---

## 套件清單

```go
// go.mod 預計引入
require (
    github.com/go-chi/chi/v5                  v5.x
    github.com/go-chi/cors                    v1.x   // CORS middleware
    github.com/jackc/pgx/v5                   v5.x
    github.com/redis/go-redis/v9              v9.x
    github.com/golang-jwt/jwt/v5              v5.x
    github.com/go-playground/validator/v10    v10.x  // 請求參數驗證
    github.com/google/uuid                    v1.x   // RequestID 產生
    github.com/joho/godotenv                  v1.x   // 載入 .env 檔
    github.com/stretchr/testify               v1.x   // assert
    github.com/testcontainers/testcontainers-go v0.x // 整合測試
)
```
