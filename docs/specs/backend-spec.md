# Backend 規格文件索引

## 開發順序

| # | 模組 | 說明 |
|---|------|------|
| 01 | [概覽](./backend/01-overview.md) | 技術棧、Clean Architecture、專案結構、套件清單 |
| 02 | [設定管理](./backend/02-config.md) | Config struct、env helpers、LoadConfig、敏感欄位遮蔽、環境變數 |
| 03 | [Logger](./backend/03-logger.md) | 結構化日誌、context 注入、log level 規範 |
| 04 | [AppError](./backend/04-apperror.md) | 統一錯誤型別、sentinel 錯誤、constructors |
| 05 | [httpx & response](./backend/05-httpx.md) | 統一回應格式、HTTP 錯誤處理層、錯誤流向 |
| 06 | [Validator](./backend/06-validator.md) | 請求參數驗證封裝、自訂規則、handler 使用方式 |
| 07 | [Database](./backend/07-database.md) | PostgreSQL 連線池、TxManager、跨 repo 交易 |
| 08 | [Redis](./backend/08-redis.md) | Redis 連線、Pinger adapter |
| 09 | [Middleware](./backend/09-middleware.md) | RequestID、Logger、CORS、Recoverer、JWT、Timeout |
| 10 | [Health Check](./backend/10-health.md) | /healthz、/readyz、Pinger interface |
| 11 | [分層規格](./backend/11-layers.md) | Context 規格、interface 定義、DI 組裝、mock 方式 |
| 12 | [啟動與關閉](./backend/12-startup.md) | main.go 骨架、優雅關閉順序、Docker |
| 13 | [TDD 測試策略](./backend/13-testing.md) | 各層測試方式、測試指令、命名規範 |

## 其他規格文件

| 文件 | 說明 |
|------|------|
| [api-spec.md](./api-spec.md) | 路由、端點 request/response |
| [db-spec.md](./db-spec.md) | Schema、Migration、sqlc |
| [jwt-auth-spec.md](./jwt-auth-spec.md) | Token 結構、Middleware |
