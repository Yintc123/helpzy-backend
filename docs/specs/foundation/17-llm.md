# 17 — LLM Provider 抽象

## 目的

提供可替換、可擴充的 LLM 模組：上層只依賴 provider-neutral 的 `Provider` interface，新增模型廠商（Claude / OpenAI / Ollama 等）= 寫一個子 package；替換預設模型 = 改環境變數。

> 對話狀態機、handoff 觸發、訊息持久化等業務邏輯**不在本文件範圍**，見 [chat-spec.md](../chat-spec.md)。

---

## 目錄結構

```
internal/services/llm/
├── provider.go              # interface 與 canonical 型別（不依賴任何 provider）
├── provider_test.go
├── registry.go              # provider 註冊與選擇
├── registry_test.go
│
├── claude/                  # 各 provider 一個子 package
│   ├── claude.go            # 實作 llm.Provider
│   ├── convert.go           # 內部訊息 ↔ Claude API 格式
│   ├── sse.go               # SSE 解析 → llm.Event
│   └── claude_test.go       # httptest 模擬 Anthropic endpoint
│
├── openai/                  # 未來新增
│   └── ...
│
├── ollama/                  # 未來新增（本地推論）
│   └── ...
│
├── prompts/                 # System prompt 與 tool 定義（純資料，無 provider 依賴）
│   ├── prompts.go           # 對外 export 的 system prompt 變數 / Render 函式
│   ├── customer_support.go  # customer chat 用 system prompt 模板
│   ├── tools.go             # CustomerSupportTools 等對外 tool 清單
│   ├── tools/
│   │   ├── escalate.go      # escalate_to_human tool
│   │   └── escalate_test.go
│   └── *_test.go
│
└── mock/                    # 上層測試用
    └── mock.go
```

> `prompts/` 是純資料 + 純函式，無 provider 相依。完整規格見 [chat-prompts-spec.md](../chat-prompts-spec.md)。

**原則：**
- `llm/` package 本身**不得 import** 任何 `claude/`、`openai/` 子 package
- 子 package 自給自足：HTTP client、格式轉換、SSE 解析皆在內部
- 上層（dispatcher / handlers）只認 `llm.Provider`，不認具體實作

---

## 核心 Interface

```go
// internal/services/llm/provider.go
package llm

type Provider interface {
    Name() string
    Stream(ctx context.Context, req Request) (<-chan Event, error)
}
```

**為什麼只暴露 `Stream` 一個方法？**
即時客服需要逐 chunk 推給顧客；背景任務（如分類、摘要）也能 collect 整段，沒必要為非串流再開 method。

---

## Canonical 型別

### Request

```go
type Request struct {
    System      string
    Messages    []Message
    Tools       []Tool
    Model       string   // 空字串 → provider 預設模型
    Temperature *float64
    MaxTokens   int      // 由 caller 從 cfg.LLM.MaxOutputTokens 填入；不直接讀 env
    CacheHints  []CacheHint // 提示哪段適合 prompt caching（provider 自行決定）
}
```

### Message

```go
type Role string

const (
    RoleUser      Role = "user"
    RoleAssistant Role = "assistant"
    RoleTool      Role = "tool"
)

type Message struct {
    Role    Role
    Content []ContentBlock // 一個 turn 可含多 block（text + tool_use）
}

type ContentBlock struct {
    Type       BlockType        // text | tool_use | tool_result
    Text       string           // type = text
    ToolUseID  string           // type = tool_use / tool_result
    ToolName   string           // type = tool_use
    ToolInput  json.RawMessage  // type = tool_use
    ToolResult string           // type = tool_result
    IsError    bool             // type = tool_result
}

type BlockType string

const (
    BlockText       BlockType = "text"
    BlockToolUse    BlockType = "tool_use"
    BlockToolResult BlockType = "tool_result"
)
```

### Tool

```go
type Tool struct {
    Name        string
    Description string
    InputSchema json.RawMessage // JSON Schema，provider 自行轉換為各家欄位名
}
```

### Event（串流事件）

`Stream` 回傳的 channel 不只吐文字，也吐 tool use 事件，避免 dispatcher 被綁死在「只能輸出文字」的 LLM。

```go
type Event interface{ isEvent() }

type TextDelta    struct{ Text string }
type ToolUseStart struct{ ID, Name string }
type ToolUseDelta struct{ ID, ArgsJSON string } // input JSON 的增量
type ToolUseEnd   struct{ ID string }
type Done         struct{ StopReason string; Usage Usage }
type ErrorEvent   struct{ Err error }

type Usage struct {
    InputTokens       int
    OutputTokens      int
    CacheReadTokens   int // 選填
    CacheWriteTokens  int // 選填
}
```

各型別實作 `isEvent()` 空方法（sealed interface 模式）。

---

## Registry

統一進出口，負責多 provider 註冊與預設選擇。

```go
// internal/services/llm/registry.go
type Registry struct {
    providers map[string]Provider
    fallback  string
}

func NewRegistry() *Registry
func (r *Registry) Register(p Provider)
func (r *Registry) Get(name string) (Provider, error) // name == "" → 用 fallback
func (r *Registry) SetFallback(name string)
```

**Dispatcher 使用方式：**

```go
provider, err := d.llm.Get(session.LLMProvider) // session 可帶 per-conversation override
events, err := provider.Stream(ctx, llm.Request{
    System:    prompts.CustomerSupportSystem(prompts.SystemPromptParams{
        BrandName:   d.cfg.BrandName,                       // 由 ChatConfig 注入
        CurrentDate: time.Now().UTC().Format("2006-01-02"),
    }),
    Messages:  toLLMHistory(session.Messages),
    Tools:     prompts.CustomerSupportTools,
    MaxTokens: d.cfg.MaxOutputTokens,                       // 由 LLMConfig 注入
})
```

> Dispatcher 持有的 `cfg` 是 `dispatcher.Config{BrandName, MaxOutputTokens, InboundChanBuffer, ...}`，由 `main.go` 從 `cfg.Chat` 與 `cfg.LLM` 各取所需欄位組裝。**Dispatcher 不直接 import `config`**，避免層級反轉。

---

## Provider 實作要求

新增 provider 子 package 必須：

1. 實作 `llm.Provider`（`Name()` 回傳唯一識別字串）
2. 將 `llm.Request` 轉換為該 API 的 request body（`convert.go`）
3. 解析串流回應為 `llm.Event`（`sse.go` 或對應協定）
4. 將 provider 專屬錯誤轉為 `apperror.*`（參考 [04-apperror.md](./04-apperror.md)）
5. 透過 `httptest.NewServer` 寫單元測試，**禁止真實打 API**

### 範例：Claude provider 簽章

```go
// internal/services/llm/claude/claude.go
package claude

type Client struct {
    apiKey       string
    defaultModel string
    http         *http.Client
}

func New(apiKey, defaultModel string) *Client
func (c *Client) Name() string                                                { return "claude" }
func (c *Client) Stream(ctx context.Context, req llm.Request) (<-chan llm.Event, error)
```

Claude API 為純 HTTPS + SSE，使用 `net/http` 即可，**不引入官方 SDK** 以保持依賴乾淨。

---

## Wiring（main.go）

```go
// cmd/server/main.go
reg := llm.NewRegistry()

if key := cfg.LLM.ClaudeAPIKey; key != "" {
    reg.Register(claude.New(key, cfg.LLM.ClaudeDefaultModel, cfg.LLM.ClaudeAPIBaseURL))
}
// 未來新增 provider：在此追加一個 if + Register。範例：
//   if key := cfg.LLM.OpenAIAPIKey; key != "" {
//       reg.Register(openai.New(key, cfg.LLM.OpenAIDefaultModel))
//   }
// 對應的 config 欄位（OpenAIAPIKey / OpenAIDefaultModel）也要在 [02-config.md](./02-config.md)
// 的 LLMConfig 與 LoadConfig / Validate / .env.example / env 變數表同步補上。
reg.SetFallback(cfg.LLM.Provider) // e.g. "claude"
```

新增 provider 三步：
1. 寫 `internal/services/llm/<name>/`
2. 在 [02-config.md](./02-config.md) 的 `LLMConfig` 與 `LoadConfig` 加對應 env
3. `main.go` 多一個 `if key := ...; reg.Register(...)`

Dispatcher / handlers / repository 完全不動。

---

## 進階模式（Decorator）

進階能力以「provider 包 provider」的 decorator 模式擴展，皆實作同一個 `Provider` interface：

| 模式 | 用途 | 實作位置（建議） |
|------|------|---------------|
| **Fallback** | Primary 出包自動切換 Secondary | `internal/services/llm/fallback/` |
| **Routing** | 分類後依複雜度走不同模型（如 Haiku → Sonnet） | `internal/services/llm/router/` |
| **Recording** | 請求/回應寫入 DB，供 evaluation / replay | `internal/services/llm/recording/` |
| **RateLimited** | 統一 quota 控制 | `internal/services/llm/ratelimited/` |

> 這些屬於擴展，**首版不需實作**。先驗證單一 provider 跑通，痛點出現再加。

---

## 環境變數

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `LLM_PROVIDER` | ✗ | `claude` | Registry fallback 名稱 |
| `CLAUDE_API_KEY` | ✗(\*) | — | Anthropic API key |
| `CLAUDE_DEFAULT_MODEL` | ✗ | `claude-haiku-4-5` | Claude 預設模型 |
| `CLAUDE_API_BASE_URL` | ✗ | `https://api.anthropic.com` | 測試 / proxy 用 |
| `LLM_REQUEST_TIMEOUT` | ✗ | `60s` | 單次請求超時 |

(\*) 至少要有一個 provider 的 key，否則啟動時 fail-fast。

---

## 測試策略

### Mock provider

```go
// internal/services/llm/mock/mock.go
package mock

type Mock struct {
    Events []llm.Event // 預錄事件按順序吐
    Err    error       // 模擬 Stream 直接失敗
}

func (m *Mock) Name() string { return "mock" }
func (m *Mock) Stream(ctx context.Context, _ llm.Request) (<-chan llm.Event, error) {
    if m.Err != nil { return nil, m.Err }
    out := make(chan llm.Event, len(m.Events))
    for _, e := range m.Events { out <- e }
    close(out)
    return out, nil
}
```

### 各層測試方式（沿用 [11-layers.md](./11-layers.md) 慣例）

| 對象 | 方式 | 依賴 |
|------|------|------|
| `services/dispatcher` | 注入 `mock.Mock`，驗證狀態機與 event 處理 | mock |
| `services/llm/claude` | `httptest.NewServer` 模擬 Anthropic SSE | 無外部 |
| `services/llm/registry` | 純 unit test | 無 |

---

## 與其他模組的關係

| 依賴 | 用途 |
|------|------|
| [02-config.md](./02-config.md) | 載入 `LLM_PROVIDER`、`CLAUDE_API_KEY` 等 |
| [03-logger.md](./03-logger.md) | 記錄請求 / 錯誤 / token usage |
| [04-apperror.md](./04-apperror.md) | Provider 錯誤轉統一型別 |
| [16-observability.md](./16-observability.md) | 串接 Stream 加 OTel span、metrics（token / latency / cost） |

---

## 相關決策

- [LLM Provider 抽象](../../decisions/llm-provider-abstraction.md)
