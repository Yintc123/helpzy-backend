# ORM 決策：sqlc

**決定**：使用 `sqlc`，不使用 `gorm`、`ent`、`bun`。

## 比較

| | sqlc | gorm | ent | bun |
|---|---|---|---|---|
| 類型 | SQL → 程式碼生成 | 傳統 ORM | Schema as Code | 輕量 ORM |
| SQL 透明度 | 完全透明 | 隱藏 | 隱藏 | 部分透明 |
| 效能 | 最高 | 中 | 中 | 高 |
| Clean Architecture | 最對齊 | 普通 | 好 | 普通 |
| 學習曲線 | 中（需寫 SQL） | 低 | 高 | 低 |

## 選擇 sqlc 的理由

1. **SQL 完全透明**：查詢行為清晰，效能問題易定位，debug 成本低
2. **型別安全**：根據 SQL 生成 Go 程式碼，編譯期即發現錯誤
3. **效能最佳**：直接使用 pgx，無 reflection 魔法
4. **Clean Architecture 對齊**：生成的程式碼只出現在 repository 層，不滲透進 service 或 models
5. **高流量友善**：查詢行為可預測，利於效能調優

## 工作流程

```
寫 SQL（db/queries/）→ sqlc generate → 使用生成的程式碼（db/sqlc/）
```

## 放棄 gorm 的理由

- 大量魔法，複雜查詢難以控制
- 效能較差，高流量場景有疑慮
- 較難與 Clean Architecture 結合
