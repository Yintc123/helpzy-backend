# Backend Foundation 規格索引

> Backend 基礎建設規格——不含業務邏輯、所有業務功能共用的底層支援。

| # | 模組 | 說明 |
|---|------|------|
| 01 | [概覽](./foundation/01-overview.md) | 技術棧、Clean Architecture、專案結構、套件清單 |
| 02 | [設定管理](./foundation/02-config.md) | Config struct、env helpers、LoadConfig、敏感欄位遮蔽、環境變數 |
| 03 | [Logger](./foundation/03-logger.md) | 結構化日誌、context 注入、log level 規範 |
| 04 | [AppError](./foundation/04-apperror.md) | 統一錯誤型別、sentinel 錯誤、constructors |
| 05 | [httpx & response](./foundation/05-httpx.md) | 統一回應格式、HTTP 錯誤處理層、錯誤流向 |
| 06 | [Validator](./foundation/06-validator.md) | 請求參數驗證封裝、自訂規則、handler 使用方式 |
| 07 | [Database](./foundation/07-database.md) | PostgreSQL 連線池、TxManager、跨 repo 交易 |
| 08 | [Redis](./foundation/08-redis.md) | Redis 連線、Pinger adapter |
| 09 | [Middleware](./foundation/09-middleware.md) | RequestID、Logger、CORS、Recoverer、JWT、Timeout |
| 10 | [Health Check](./foundation/10-health.md) | /healthz、/readyz、Pinger interface |
| 11 | [分層規格](./foundation/11-layers.md) | Context 規格、interface 定義、DI 組裝、mock 方式 |
| 12 | [啟動與關閉](./foundation/12-startup.md) | main.go 骨架、優雅關閉順序、Docker |
| 13 | [TDD 測試策略](./foundation/13-testing.md) | 各層測試方式、測試指令、命名規範 |
| 14 | [SQL Migration & sqlc](./foundation/14-sqlc-and-migration.md) | golang-migrate、sqlc 工作流、Repository 包裝 pattern |
| 15 | [Auth](./foundation/15-auth.md) | JWT 簽發/驗證、密碼 hash、Refresh token rotation + reuse detection、Family-level 撤銷、Middleware |
