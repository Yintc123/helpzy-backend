# 架構決策：Clean Architecture

**決定**：採用 Clean Architecture 四層結構。

```
Frameworks & Drivers     ← chi、pgx、sqlc、go-redis
Interface Adapters       ← handlers、repository 實作
Use Cases                ← services（商業邏輯）
Entities                 ← models（純 Go struct）
```

## 採用的理由

1. **可測試性**：每層透過 interface 溝通，可獨立 mock 測試
2. **可維護性**：商業邏輯不依賴框架，框架升級或替換影響範圍小
3. **TDD 友善**：interface-based 設計讓寫測試先於實作更自然
4. **高流量擴充**：各層職責清晰，效能瓶頸易定位

## 核心原則

- 框架型別只出現在最外層
- 每層透過 interface 溝通，不依賴具體實作
- Dependency Injection 在 `main.go` 手動組裝
