# ResponseWriter Wrapper 必須實作 `Unwrap()`

**決定**：專案內所有包裝 `http.ResponseWriter` 的型別（middleware writers、test recorders 等）都**必須**實作：

```go
func (w *MyWrapper) Unwrap() http.ResponseWriter {
    return w.ResponseWriter
}
```

違反此規則的 wrapper 一旦掛在路由鏈上，下游任何使用 `http.ResponseController`（WebSocket 升級、SSE flush、動態 timeout 調整）的 handler 都會壞掉。

---

## 背景：HTTP 進階能力的查找機制

`http.ResponseWriter` 介面只定義三個方法：

```go
type ResponseWriter interface {
    Header() http.Header
    Write([]byte) (int, error)
    WriteHeader(statusCode int)
}
```

但實際的 `ResponseWriter` 還可能支援四項「進階能力」：

| 能力 | 用途 | 觸發場景 |
|------|------|---------|
| `Hijack()` | 接管底層 TCP socket | WebSocket 升級、自訂協定隧道 |
| `Flush()` | 立即把 buffer 推給 client | SSE、streaming JSON、chunked 輸出 |
| `SetReadDeadline()` | 動態調整讀取 deadline | 長輪詢、大檔上傳 |
| `SetWriteDeadline()` | 動態調整寫入 deadline | 長連線、慢消費者 |

這些**不在 `ResponseWriter` interface 裡**——它們是 optional capabilities。Go 1.20+ 提供 `http.ResponseController` 作為統一入口：

```go
rc := http.NewResponseController(w)
conn, _, err := rc.Hijack()   // 找得到就用，找不到回 errors.New("feature not supported")
err = rc.Flush()
```

### `ResponseController` 怎麼找這些能力

```go
// Go 標準庫實作（簡化）
func (rc *ResponseController) Hijack() (net.Conn, *bufio.ReadWriter, error) {
    w := rc.rw
    for {
        switch v := w.(type) {
        case interface{ Hijack() (net.Conn, *bufio.ReadWriter, error) }:
            return v.Hijack()
        case interface{ Unwrap() http.ResponseWriter }:
            w = v.Unwrap()  // ← 沿著 Unwrap 鏈往內找
        default:
            return nil, nil, errors.New("feature not supported")
        }
    }
}
```

**關鍵**：它**不靠 Go embed 的自動 method promotion**，而是要求 wrapper 明確宣告 `Unwrap()`。Embed 一個 `http.ResponseWriter` 並不會讓 `ResponseController` 自動穿透——它看到 wrapper 上沒有目標方法、也沒有 `Unwrap()`，就直接放棄。

---

## 我們踩過的具體案例

`pkg/middleware/logger.go` 為了記錄下游 handler 的 HTTP status code，包了一層 `statusWriter`：

```go
type statusWriter struct {
    http.ResponseWriter
    status int
}

func (sw *statusWriter) WriteHeader(code int) {
    sw.status = code
    sw.ResponseWriter.WriteHeader(code)
}
```

這段 code 對 99% 的 REST handler 完全沒事——`Write` / `Header` / `WriteHeader` 都透過 embed promotion 自動轉發。

但 root router `r.Use(middleware.Logger, ...)` 對 **所有路由群組** 生效，包含 WebSocket 路由。當 WS handler 呼叫 `coder/websocket.Accept(w, r, ...)`：

```
1. websocket.Accept 內部: rc := http.NewResponseController(w)
2. rc.Hijack() 被呼叫
3. ResponseController 看到 *statusWriter:
     - 沒有 Hijack 方法 ❌
     - 沒有 Unwrap 方法 ❌
4. 回 errors.New("feature not supported")
5. WS 升級失敗
```

底層的 chi `ResponseWriter`（**有** `Hijack`）就埋在 `statusWriter.ResponseWriter` 那個 embed 欄位裡，但 `ResponseController` 看不到。

### 修法（一行）

```go
func (sw *statusWriter) Unwrap() http.ResponseWriter {
    return sw.ResponseWriter
}
```

加上後：`ResponseController` 看到 `Unwrap()`，往內走到 chi writer，找到 `Hijack` → 升級成功。

---

## 為什麼這 bug 隱蔽

1. **REST API 完全感受不到**：所有 JSON handler 只用 `Write` / `Header` / `WriteHeader`，這三個 embed 自動 promote 完全沒事
2. **單元測試蓋不到**：用 `httptest.NewRecorder()` 跑的測試只測 status code 與 body，不會觸發 `Hijack`
3. **整合測試也很可能蓋不到**：除非有專門測 WS 升級的 e2e
4. **第一次撞到時錯誤訊息誤導**：`ws accept: feature not supported`——讀起來像是 `coder/websocket` 套件問題，不會立刻聯想到中間某層 middleware 的 wrapper

「能不能跑」與「會不會升級」是不同光譜上的問題；統一寫入 `Unwrap()` 是把整個光譜一次蓋住的最小成本做法。

---

## 規範

### 1. 任何 ResponseWriter wrapper 必須實作 `Unwrap()`

包含但不限於：
- `pkg/middleware/logger.go` 的 `statusWriter`
- `pkg/middleware/timeout.go` 的 `timeoutWriter`
- 未來新增的任何 metric / trace / cache / compression middleware writer
- 測試用的 fake / spy writer

### 2. Wrapper 自己若實作了某項進階能力，無須 `Unwrap()` 到該能力

例：若 `gzipWriter` 自己實作 `Flush()`（觸發 gzip flush），就不需要靠 `Unwrap()` 把 `Flush` 委派下去。`ResponseController` 找到自身就停。

### 3. Test：所有新 wrapper 至少要驗 `http.ResponseController` 對其能否拿到底層 writer

```go
func TestWrapper_Unwrap(t *testing.T) {
    base := httptest.NewRecorder()
    wrapped := &MyWrapper{ResponseWriter: base}
    
    unwrapped, ok := any(wrapped).(interface{ Unwrap() http.ResponseWriter })
    require.True(t, ok, "wrapper must implement Unwrap()")
    require.Same(t, base, unwrapped.Unwrap())
}
```

### 4. Code review checklist

PR 引入新的 `http.ResponseWriter` wrapper 時，reviewer 確認：
- [ ] 實作了 `Unwrap() http.ResponseWriter`
- [ ] 該方法回傳的是 embed 欄位，不是 `nil`
- [ ] 對應測試已加上

---

## 受影響的現有規格

| 規格 | 章節 |
|------|------|
| [foundation/09-middleware.md](../specs/foundation/09-middleware.md) | `statusWriter`、`timeoutWriter` |
| [foundation/19-websocket.md](../specs/foundation/19-websocket.md) | `wsx.Accept` 依賴 `ResponseController.Hijack` 能穿透到底層 |
| [specs/chat-spec.md](../specs/chat-spec.md) | WS handler 用 `http.ResponseController.SetWriteDeadline` 清除 `http.Server.WriteTimeout` |

---

## 參考

- Go 1.20 release notes — `http.ResponseController`：<https://go.dev/doc/go1.20#nethttppkgnethttp>
- `net/http` 原始碼 `ResponseController` 實作：<https://pkg.go.dev/net/http#ResponseController>
- `coder/websocket` 升級流程：<https://github.com/coder/websocket/blob/master/accept.go>
