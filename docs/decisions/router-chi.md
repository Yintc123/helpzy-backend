# HTTP Router 決策：chi v5

**決定**：使用 `chi`，不使用 `gin`、`echo`、`fiber`。

## 比較

| | chi | gin | echo | fiber |
|---|---|---|---|---|
| 相容 `net/http` | 完全相容 | 否（gin.Context） | 部分 | 否（fasthttp） |
| Clean Architecture | 自然對齊 | 需額外紀律 | 需額外紀律 | 不適合 |
| Context 控制 | 原生 stdlib | 需轉換 | 需轉換 | 不相容 |
| 社群資源 | 少 | 最多 | 中 | 中 |
| 效能 | 高 | 高 | 高 | 最高 |

## 選擇 chi 的理由

1. **Clean Architecture 自然對齊**：handler 就是 `http.Handler`，框架型別不滲透進業務邏輯層
2. **context.Context 完整控制**：timeout、cancel、值注入全走 stdlib 標準，從 handler 貫穿至 repository
3. **可測試性**：handler 直接用 `httptest` 測試，不需建立框架 engine
4. **介面可替換**：所有 middleware 相容 `net/http`，未來換框架只動 `main.go`
5. **高流量場景**：框架本身不是瓶頸，真正的瓶頸在 DB 查詢；chi 的 context 控制能精確釋放資源

## 放棄 gin 的理由

- `gin.Context` 會滲透進 handler 層，需要團隊紀律才能維持 Clean Architecture
- 測試 handler 需要建立 gin engine，增加測試複雜度
- 額外功能（binding、validation）在此架構中用處不大

## 放棄 fiber 的理由

- 基於 `fasthttp` 而非 `net/http`，整個生態系不相容
- 整合第三方套件時受限
