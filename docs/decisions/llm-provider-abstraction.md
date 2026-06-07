# 架構決策：LLM Provider 抽象

**決定**：在 `internal/services/llm/` 定義 provider-neutral 的 `Provider` interface 與 canonical 訊息格式；具體 vendor（Claude / OpenAI / 本地推論）為子 package 實作。對話歷史以 canonical schema 持久化。

## 採用的理由

1. **可替換性**：換 vendor = 換 env，不動 dispatcher / repository / DB schema
2. **可擴充性**：新增 provider = 寫一個子 package，主流程零修改
3. **資料中立**：對話歷史不綁特定 vendor 格式，未來分析、replay、A/B test 不被綁死
4. **Decorator 友善**：fallback / routing / recording 都用「provider 包 provider」實作，疊起來都通

## 核心設計選擇

### 1. Event stream，不只是 text chunk

`Stream` 回傳 `<-chan Event`，事件包含 `TextDelta` / `ToolUseStart` / `ToolUseDelta` / `ToolUseEnd` / `Done`。

**理由**：客服系統的 handoff 觸發走 tool call。若 interface 只回 text chunk，dispatcher 無法分辨 LLM 是在打字還是在呼叫工具，被綁死在「純文字 LLM」。

### 2. Canonical message schema 持久化

DB 存的訊息採用 provider-neutral 的 block 陣列格式（`text` / `tool_use` / `tool_result`），對應 `llm.ContentBlock`。

**理由**：
- 顧客在 Claude 開始對話 → 客服接手後想丟 OpenAI 摘要 → 必須能用同一份歷史
- Dashboard / 分析 / 稽核都不該寫 if-else 處理多家格式
- 換 vendor 不該觸發 DB migration

**抽象掉的部分**：role、content blocks、tool 結構、token usage  
**保留原貌的部分**：stop_reason、cache 命中、provider 專屬旗標 → 存 `provider_meta JSONB`

### 3. 不引入官方 SDK

各 provider 子 package 使用 `net/http` + 手寫 SSE 解析。

**理由**：
- 依賴更乾淨（官方 SDK 常帶大量無關套件）
- API surface 小，自寫成本低
- 對 SSE 與重試行為有完整掌控

### 4. 不過早抽象 fallback / routing

首版只實作單一 provider。decorator 模式（`FallbackProvider`、`RouterProvider`、`RecordingProvider`）皆能事後追加而不破壞 interface。

**理由**：第二個 provider 出現前，無法判斷 interface 的真正痛點；先求可跑，痛點出現再迭代。

## 不採用的替代方案

| 方案 | 不採用理由 |
|------|----------|
| 直接呼叫 Anthropic SDK | dispatcher 會被 Claude API 型別污染，未來替換成本高 |
| 各 provider 各存原始格式 | 換 vendor 即破表，無法統一查詢與分析 |
| 抽象出獨立 LLM 微服務（Python） | 邏輯不複雜，多一個服務只增加延遲與維運成本 |
| 用 LangChain / LiteLLM 等中介層 | 引入大型依賴與其抽象觀念，鎖死生態；Go 端規模還不需要 |

## 相關規格

- [foundation/17-llm.md](../specs/foundation/17-llm.md) — interface、Registry、Provider 實作要求
- [chat-spec.md](../specs/chat-spec.md) — Canonical message schema 在對話系統的使用
