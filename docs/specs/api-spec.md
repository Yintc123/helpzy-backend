# API 規格書

> Backend 提供給 Next.js 前端的 RESTful API 規格。

## 路由設計

### 結構

```go
// cmd/server/main.go
r := chi.NewRouter()

// 全域 Middleware
r.Use(middleware.Logger)
r.Use(middleware.CORS)
r.Use(middleware.Recoverer)

// 公開路由（不需要 JWT）
r.Get("/api/health", handlers.HealthCheck)
r.Post("/api/v1/auth/login", userHandler.Login)

// 受保護路由（需要 JWT）
r.Group(func(r chi.Router) {
    r.Use(jwtMiddleware.Verify)
    r.Use(middleware.Timeout(5 * time.Second))

    r.Get("/api/v1/users/{id}", userHandler.GetUser)
    r.Put("/api/v1/users/{id}", userHandler.UpdateUser)
})
```

### 路由清單

| Method | Path | JWT | 說明 |
|--------|------|-----|------|
| GET | `/api/health` | 否 | 健康檢查 |
| POST | `/api/v1/auth/register` | 否 | 註冊 |
| POST | `/api/v1/auth/login` | 否 | 登入 |
| POST | `/api/v1/auth/refresh` | 否 | 刷新 access token（rotation） |
| POST | `/api/v1/auth/logout` | **是** | 登出（撤銷 access + refresh） |
| GET | `/api/v1/users/{id}` | **是** | 取得使用者資料 |
| PUT | `/api/v1/users/{id}` | **是** | 更新使用者資料 |

---

## 端點規格

> TODO：各端點的 request body、response body、欄位驗證、錯誤碼對照待定義。

### `POST /api/v1/auth/login`

- Request body: TODO
- Response body: TODO
- 錯誤碼: TODO

### `GET /api/v1/users/{id}`

- Path param: `id` — TODO（UUID? 數字?）
- Response body: TODO
- 錯誤碼: TODO

### `PUT /api/v1/users/{id}`

- Path param: `id` — TODO
- Request body: TODO（可更新欄位、驗證規則）
- Response body: TODO
- 錯誤碼: TODO

---

## 相關文件

- [Backend Foundation 規格](./backend-foundation-spec.md)
- [JWT 跨服務契約](./jwt-auth-spec.md)
