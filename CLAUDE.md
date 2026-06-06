# Helpzy Backend

Go 1.23 RESTful API 專案。

## 開發流程：TDD

嚴格遵守 **Red → Green → Refactor** 循環：

1. **Red**：先寫一個會失敗的測試
2. **Green**：寫最少量的程式碼讓測試通過
3. **Refactor**：重構，確保測試仍然通過

**禁止先寫實作再補測試。**

## 測試工具

- 測試框架：Go 標準庫 `testing`
- HTTP 測試：`net/http/httptest`
- Mock：`testify/mock`（視需求引入）

## 測試指令

```bash
go test ./...               # 執行所有測試
go test ./... -v            # 詳細輸出
go test ./... -cover        # 顯示覆蓋率
go test ./internal/...      # 只測試 internal
```

## 測試檔案規範

- 測試檔與原始檔放在同一目錄，命名為 `*_test.go`
- 整合測試放在 `internal/` 各層的 `*_test.go`，使用 `httptest` 模擬 HTTP

```
internal/
├── handlers/
│   ├── health.go
│   └── health_test.go
├── services/
│   ├── user.go
│   └── user_test.go
└── repository/
    ├── user.go
    └── user_test.go
```

## 測試命名規範

```go
func TestFunctionName_條件_預期結果(t *testing.T) { ... }

// 範例
func TestHealthCheck_WhenCalled_ReturnsStatusOK(t *testing.T) { ... }
```

## 分層測試策略

| 層 | 測試方式 |
|----|---------|
| `handlers` | 使用 `httptest.NewRecorder()` 測試 HTTP 請求／回應 |
| `services` | 純函式單元測試，mock repository |
| `repository` | 整合測試，連接真實測試資料庫 |

## 每次新增功能的步驟

1. 建立 `*_test.go`，寫出描述預期行為的測試（此時測試應失敗）
2. 執行 `go test ./...` 確認紅燈
3. 實作最少程式碼讓測試通過
4. 執行 `go test ./...` 確認綠燈
5. 重構，確認測試仍為綠燈
