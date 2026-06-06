# Helpzy Backend

Helpzy 後端，使用 Go 1.23 開發的 RESTful API 伺服器。

前端：[helpzy-frontend](../frontend)

## 技術棧

| 項目 | 技術 |
|------|------|
| 語言 | Go 1.23 |
| HTTP | 標準庫 `net/http` |
| 資料庫 | PostgreSQL 16 |

## 專案結構

```
backend/
├── cmd/
│   └── server/
│       └── main.go         # 程式進入點
├── internal/               # 私有應用程式邏輯（不對外暴露）
│   ├── handlers/           # HTTP 請求處理器
│   ├── models/             # 資料模型（struct）
│   ├── services/           # 商業邏輯層
│   └── repository/         # 資料存取層（DB 操作）
├── pkg/
│   └── middleware/         # 可共用的 HTTP Middleware
│       ├── cors.go
│       └── logger.go
├── configs/
│   └── .env.example
├── Dockerfile
└── go.mod
```

### 分層說明

```
HTTP Request
    │
    ▼
handlers/       ← 解析請求、回傳回應
    │
    ▼
services/       ← 商業邏輯、資料驗證
    │
    ▼
repository/     ← 資料庫查詢
    │
    ▼
models/         ← 資料結構定義
```

新增功能時，依此順序在各層建立對應檔案。

## 快速開始

### 前置需求

- Go 1.23+
- PostgreSQL 16+（或使用 Docker）

### 啟動伺服器

```bash
# 複製環境變數
cp configs/.env.example configs/.env
# 編輯 configs/.env，填入資料庫設定

# 啟動
go run ./cmd/server
```

伺服器運行於 [http://localhost:8080](http://localhost:8080)

### 編譯

```bash
go build -o bin/server ./cmd/server
./bin/server
```

## API

### Health Check

```
GET /api/health
```

```json
{ "status": "ok" }
```

## 環境變數

複製 `configs/.env.example` 為 `configs/.env` 後修改：

| 變數 | 說明 |
|------|------|
| `PORT` | 伺服器監聽埠號（預設 `8080`） |
| `ENV` | 執行環境（`development` / `production`） |
| `DB_HOST` | 資料庫主機 |
| `DB_PORT` | 資料庫埠號 |
| `DB_NAME` | 資料庫名稱 |
| `DB_USER` | 資料庫使用者 |
| `DB_PASSWORD` | 資料庫密碼 |
| `JWT_SECRET` | JWT 簽署金鑰 |

## Docker

```bash
docker build -t helpzy-backend .
docker run -p 8080:8080 --env-file configs/.env helpzy-backend
```
