# Chat Prompts 規格書

> 規範 system prompt、tool 定義、prompt 修改流程。**本專案 prompt 寫死在 Go 程式碼**（不走 DB / env），原因見 [chat-spec.md 顧客身分模式](./chat-spec.md) 旁的決策說明。

## 目的

- Prompt 與程式碼同步演進、版本控制清楚、可被 unit test 直接驗
- 修改 prompt 等同修改程式碼，走相同的 PR / CI / TDD 流程
- 避免「prompt 改完忘記改 tool schema 或反之」的同步問題

> Prompt 來源的選型理由（為何不放 DB / env）由本檔說明；無獨立 decision 文件 — 屬於小型實作決策、可在 spec 內就地說明。

---

## 目錄結構

```
internal/services/llm/prompts/
├── prompts.go              # 對外 export 的 system prompt 變數
├── prompts_test.go         # 結構性測試：含必要 placeholder、長度、語言
├── customer_support.go     # 主要 system prompt 內容（template 變數）
├── tools.go                # 對外 export 的 tool 定義（llm.Tool 陣列）
├── tools_test.go           # schema 驗證、tool name 唯一性
└── tools/
    ├── escalate.go         # escalate_to_human tool
    └── escalate_test.go    # input schema 範例驗證
```

> 為什麼 `prompts/tools/` 又開一層？因為每個 tool 不只是 schema，**還有對應的 Go-side handler**（解析 input、執行動作、組 tool result）。一個 tool 自成一個檔案夾較易擴充（未來 `tools/lookup_order/`、`tools/refund_request/`）。

---

## System Prompt

### `customer_support.go`

```go
package prompts

import (
    "fmt"
    "strings"
)

// CustomerSupportSystemTemplate 是 customer chat 的 system prompt 模板。
// {{brandName}} / {{currentDate}} 由 Render 函式填入。
const CustomerSupportSystemTemplate = `你是 {{brandName}} 的智慧客服助理。

行為準則：
- 用繁體中文回應，語氣專業、簡潔、友善。
- 只回答與本品牌產品或服務相關的問題。
- 不知道或無法處理的問題，呼叫 escalate_to_human 工具交給真人客服。
- 不要編造產品資訊、價格、政策內容。
- 不要對顧客的人身議題（健康、法律、財務）提供建議。

今日日期：{{currentDate}}
`

type SystemPromptParams struct {
    BrandName   string // 由 caller 從 cfg.Chat.BrandName 注入；prompts 套件不直接讀 env
    CurrentDate string // ISO 8601 yyyy-mm-dd
}

func CustomerSupportSystem(p SystemPromptParams) string {
    s := CustomerSupportSystemTemplate
    s = strings.ReplaceAll(s, "{{brandName}}", p.BrandName)
    s = strings.ReplaceAll(s, "{{currentDate}}", p.CurrentDate)
    return s
}
```

> **BrandName 為何走 env 而非寫死於 prompt template？**  
> 同一份 binary 部署到不同租戶（multi-tenant / 白標）只需改 `CHAT_BRAND_NAME` env 即可上線，不需重 build。`prompts` 套件本身保持 pure（不 import config），由 dispatcher / service 層在組 `Request` 時填入。

### 為什麼用字串模板而不是 `text/template`

- 模板簡單（兩個 placeholder），引入 `text/template` 增加 build cost 與閱讀負擔
- `text/template` 的錯誤難以在 compile 時抓
- `strings.ReplaceAll` 三行內可被 unit test 驗清楚

未來模板複雜（多分支、迴圈）再考慮。**目前不必**。

---

## Tool 定義

### `tools.go`

對外暴露的「目前啟用的工具清單」：

```go
package prompts

import (
    "github.com/helpzy/backend/internal/services/llm"
    "github.com/helpzy/backend/internal/services/llm/prompts/tools"
)

// CustomerSupportTools 是顧客 chat 預設啟用的工具集。
// dispatcher 透過此清單組 llm.Request.Tools。
var CustomerSupportTools = []llm.Tool{
    tools.EscalateToHuman,
}
```

### `tools/escalate.go`

```go
package tools

import (
    "encoding/json"
    "github.com/helpzy/backend/internal/services/llm"
)

const EscalateToHumanName = "escalate_to_human"

var EscalateToHuman = llm.Tool{
    Name: EscalateToHumanName,
    Description: "當你無法回答顧客問題、或顧客明確要求真人協助時呼叫此工具，將對話交給人類客服。",
    InputSchema: json.RawMessage(`{
        "type": "object",
        "properties": {
            "reason": {
                "type": "string",
                "description": "簡述為何需要真人接手（給客服快速判斷用）"
            },
            "category": {
                "type": "string",
                "enum": ["refund", "complaint", "technical", "account", "other"],
                "description": "問題分類"
            }
        },
        "required": ["reason", "category"]
    }`),
}

// EscalateToHumanInput 為 LLM 呼叫此 tool 時的 input 結構。
type EscalateToHumanInput struct {
    Reason   string `json:"reason"`
    Category string `json:"category"`
}

// ParseEscalateInput 解析 tool input；解析失敗回 nil + error，由 dispatcher 處理。
func ParseEscalateInput(raw json.RawMessage) (*EscalateToHumanInput, error) {
    var in EscalateToHumanInput
    if err := json.Unmarshal(raw, &in); err != nil {
        return nil, err
    }
    return &in, nil
}
```

### Dispatcher 處理 tool call 的概念碼

```go
// internal/services/dispatcher/dispatcher.go
switch ev := event.(type) {
case llm.ToolUseEnd:
    raw := /* 從 buffer 拼出 input JSON */
    switch ev.Name {
    case tools.EscalateToHumanName:
        in, err := tools.ParseEscalateInput(raw)
        if err != nil { /* log + tool_result 回 is_error: true */ }
        d.transitionToHandoff(sess, in.Reason, in.Category)
    default:
        // 未註冊 tool：log + tool_result 回 is_error
    }
}
```

---

## 結構性測試

Prompt 與 tool 都會被 unit test 驗，避免後續編輯時破壞契約。

### `prompts_test.go`

```go
func TestCustomerSupportSystem_ContainsRequiredSections(t *testing.T) {
    out := prompts.CustomerSupportSystem(prompts.SystemPromptParams{
        BrandName:   "Helpzy",
        CurrentDate: "2026-06-07",
    })
    require.Contains(t, out, "Helpzy")
    require.Contains(t, out, "2026-06-07")
    require.Contains(t, out, "escalate_to_human") // tool 名稱被提及
    require.NotContains(t, out, "{{")             // placeholder 全部填掉
}
```

### `tools_test.go`

```go
func TestCustomerSupportTools_NamesUnique(t *testing.T) {
    seen := map[string]bool{}
    for _, tool := range prompts.CustomerSupportTools {
        require.False(t, seen[tool.Name], "duplicate tool name: %s", tool.Name)
        seen[tool.Name] = true
    }
}

func TestEscalateToHuman_SchemaIsValidJSON(t *testing.T) {
    var v any
    require.NoError(t, json.Unmarshal(tools.EscalateToHuman.InputSchema, &v))
}

func TestEscalateToHuman_RequiredFields(t *testing.T) {
    // 驗 input schema 的 required 至少含 reason / category
    // ...
}
```

### `tools/escalate_test.go`

```go
func TestParseEscalateInput_Valid(t *testing.T) {
    in, err := tools.ParseEscalateInput(json.RawMessage(
        `{"reason":"refund","category":"refund"}`,
    ))
    require.NoError(t, err)
    require.Equal(t, "refund", in.Category)
}

func TestParseEscalateInput_BadJSON(t *testing.T) {
    _, err := tools.ParseEscalateInput(json.RawMessage(`{not json`))
    require.Error(t, err)
}
```

---

## 修改 Prompt 的 TDD 流程

prompt 改動屬於行為改動，**比一般程式碼改動更需要測試覆蓋**：

1. 在 `prompts_test.go` 加新測試斷言「prompt 包含 X」「不包含 Y」 → 紅
2. 改 `CustomerSupportSystemTemplate` 內容 → 綠
3. 如果改了會影響工具 → 同步改 `tools.go` + `tools_test.go`
4. 跑 cross-provider 測試（[chat-spec.md 跨 Provider 帶歷史的相容性](./chat-spec.md#跨-provider-帶歷史的相容性)）確認新 prompt 在每個 provider 都被正確序列化

### 不允許

- ❌ 直接改 `CustomerSupportSystemTemplate` 不加測試
- ❌ 加 tool 不寫 `Parse...Input` 函式
- ❌ 在 dispatcher 內 hardcode tool name（一律 `tools.EscalateToHumanName` 常數）

---

## 與其他模組的關係

| 模組 | 關係 |
|------|------|
| [foundation/17-llm.md](./foundation/17-llm.md) | `prompts` 套件回傳 `llm.Tool` / system prompt string，是 LLM provider 的純資料輸入 |
| [chat-spec.md](./chat-spec.md) | dispatcher 從 `prompts.CustomerSupportTools` 取得工具清單；偵測 `EscalateToHumanName` 觸發 handoff |
| [tech-stack/contextkey.md](../tech-stack/contextkey.md) | system prompt 內若需動態填入 user 資訊，從 ctx 取（不在 prompts 套件內讀 ctx） |

---

## 未來擴充考量

| 場景 | 升級路徑 |
|------|--------|
| Prompt 量大、需要協作者非工程師編輯 | 引入 i18n 工具（`go-i18n`）或抽 prompt 到 `.tmpl` 檔（仍 build-time embed） |
| 需要 A/B test 兩版 prompt | 在 `CustomerSupportSystem` 增加 variant 參數，由 Registry decorator 控制 |
| Multi-tenant 不同品牌不同 prompt | 將 `SystemPromptParams` 擴充為品牌特性的完整集合（語氣、政策連結、營業時間），由 service 層自 `Tenant` repo 讀取後注入；env-based 的 `CHAT_BRAND_NAME` 退場 |
| 動態調 prompt 不重 deploy | 評估 ConfigMap reload 或抽出 prompt store；屆時再決策 |

**v1 不需要這些**。

---

## 相關文件

- [chat-spec.md](./chat-spec.md) — Dispatcher 如何呼叫 tools
- [foundation/17-llm.md](./foundation/17-llm.md) — LLM Provider 介面
- [api-spec.md](./api-spec.md) — Chat 端點
