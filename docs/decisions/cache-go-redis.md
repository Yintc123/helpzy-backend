# Redis 客戶端決策：go-redis v9

**決定**：使用 `go-redis v9`，不使用 `rueidis`、`redigo`。

## 比較

| | go-redis v9 | rueidis | redigo |
|---|---|---|---|
| 維護狀態 | 活躍（Redis 官方） | 活躍 | 趨緩 |
| API 易用性 | 高 | 中 | 低（手動） |
| 效能 | 高 | 最高 | 普通 |
| Client-side Cache | 否 | 是 | 否 |

## 選擇 go-redis 的理由

1. **Redis 官方維護**，長期支援有保障
2. API 最直覺，開發速度快
3. 文件完整，範例豐富
4. 對此專案用途（Session 儲存、快取）完全足夠

## 何時考慮改用 rueidis

需要 client-side caching 或壓測後有明顯效能瓶頸時。
