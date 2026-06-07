# 架構決策：客服 Assignment 採用 FCFS

**決定**：客服 dashboard 顯示待接手清單（按 `pending_handoff` 時間排序），由客服**自行點選接手**；伺服器以 atomic UPDATE 防止並發搶佔。**不做自動指派、不分群組、不按 skill 路由。**

## 採用的理由

1. **規模匹配**：MVP 階段客服人數有限，視覺化清單 + 手動選取夠用
2. **零隱藏邏輯**：客服自己決定接哪個對話，不會出現「為何系統把這個給我？」的困惑
3. **實作成本最低**：一個 SQL UPDATE + 一個 dashboard 元件，無需 routing 引擎、無需 skill 標籤系統
4. **easy to upgrade**：未來若量起來要自動分派，dashboard 顯示與 atomic claim 機制都可保留，只是上層多一層「自動選 + 一鍵接受」

## 不採用的替代方案

| 方案 | 不採用理由 |
|------|----------|
| Round-robin 自動指派 | 需要追蹤每位客服當前負載、處理離線 / 拒接 / 超時等狀態；MVP 過度 |
| Skill-based routing | 需要 tag 系統、客服能力標註、對話分類器；MVP 過度 |
| 客服群組 / 部門分流 | 組織架構需求未浮現；提前抽象風險高 |
| WebSocket push 自動分派 | 與「客服自選」UX 衝突，且客服可能不在線時搶到 |

## Atomic Claim 機制

```sql
UPDATE conversations
SET assigned_agent_id = :me,
    mode = 'human',
    updated_at = NOW()
WHERE id = :id
  AND assigned_agent_id IS NULL
  AND mode = 'pending_handoff';
-- 影響列數 = 0 → 已被其他客服搶先，UI 顯示「已被其他人接手」並 refresh 清單
```

## 對應規格

- [chat-spec.md](../specs/chat-spec.md#客服-assignment-策略fcfs) — 完整規格
- [api-spec.md](../specs/api-spec.md) — `POST /api/v1/conversations/{id}/takeover` 端點
