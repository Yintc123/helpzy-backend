# 02 — 設定管理（Config）

## 目的

集中管理所有 env，提供型別安全的 `Config` struct，並在啟動時統一驗證。任何模組需要設定一律由 `main.go` 從 `Config` 傳入，**禁止散落呼叫 `os.Getenv`**。

## 設計原則

- **三種環境**：`local` / `production` / `test`，由 `LoadConfig(env)` 外部傳入
- 對應載入 `.env.local` / `.env` / `.env.test`（用 `joho/godotenv`）
- **手寫 helpers**（`getEnv` 系列），不使用第三方 env-tag 框架——保留簡單、顯式、易讀
- **嚴格 fail-fast**：必填缺漏、格式錯誤（int / duration parse 失敗）、跨欄位驗證失敗 → 一律 `return error` 讓 `main.go` 退出，不靜默落回預設值
- **只放「會隨環境改變的設定」**：應用層常數（如服務名、Driver、固定列舉清單）放 `const` 或 `internal/constants`，**禁止混入 `Config`**——避免讓人誤以為可調整
- **敏感欄位自動遮蔽**：`Password` / `Secret` / 含密碼的 URL 透過 `String()` 方法在 log 中印 `***`
- 不可變：載入後不再修改；測試以建構函式注入假值
- `normalizeEnv()` 容錯不合法輸入（落回 `local`）

---

## Config Struct

```go
// internal/config/config.go
type Config struct {
    App   AppConfig
    DB    DBConfig
    Redis RedisConfig
    JWT   JWTConfig
}

type AppConfig struct {
    Env             string        // local / production / test
    Port            int
    LogLevel        string
    AllowedOrigin   string
    ShutdownTimeout time.Duration
}

type DBConfig struct {
    Host     string
    Port     int
    User     string
    Password string
    Name     string
    SSLMode  string
}

type RedisConfig struct {
    URL string
}

type JWTConfig struct {
    Secret string
}
```

---

## Env Helpers

```go
// internal/config/config.go

// 必填：缺漏即 return error
func getEnv(key string) (string, error) {
    v := os.Getenv(key)
    if v == "" {
        return "", fmt.Errorf("missing required env: %s", key)
    }
    return v, nil
}

// 可選字串
func getEnvWithDefault(key, defaultValue string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return defaultValue
}

// 可選整數；解析失敗即 error（不靜默落回 default）
func getEnvInt(key string, defaultValue int) (int, error) {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue, nil
    }
    v, err := strconv.Atoi(raw)
    if err != nil {
        return 0, fmt.Errorf("env %s: invalid int %q: %w", key, raw, err)
    }
    return v, nil
}

// 可選 Duration；解析失敗即 error
func getEnvDuration(key string, defaultValue time.Duration) (time.Duration, error) {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue, nil
    }
    v, err := time.ParseDuration(raw)
    if err != nil {
        return 0, fmt.Errorf("env %s: invalid duration %q: %w", key, raw, err)
    }
    return v, nil
}

func normalizeEnv(env string) string {
    switch strings.ToLower(env) {
    case "production", "local", "test":
        return strings.ToLower(env)
    default:
        return "local"
    }
}
```

---

## LoadConfig

```go
// internal/config/config.go

func LoadConfig(env string) (*Config, error) {
    env = normalizeEnv(env)

    // 1) 依環境載入對應 .env
    switch env {
    case "production":
        _ = godotenv.Load(".env") // 容器內通常由 orchestrator 注入，找不到也 OK
    case "local":
        if err := godotenv.Load(".env.local"); err != nil {
            slog.Warn(".env.local not loaded", "err", err)
        }
    case "test":
        if err := godotenv.Load(".env.test"); err != nil {
            return nil, fmt.Errorf("load .env.test: %w", err)
        }
    }

    // 2) 必填字串
    dbHost,        err := getEnv("DB_HOST");        if err != nil { return nil, err }
    dbUser,        err := getEnv("DB_USER");        if err != nil { return nil, err }
    dbPassword,    err := getEnv("DB_PASSWORD");    if err != nil { return nil, err }
    redisURL,      err := getEnv("REDIS_URL");      if err != nil { return nil, err }
    jwtSecret,     err := getEnv("JWT_SECRET");     if err != nil { return nil, err }
    allowedOrigin, err := getEnv("ALLOWED_ORIGIN"); if err != nil { return nil, err }

    // 3) 型別 envs（解析失敗即 fail-fast）
    dbPort, err := getEnvInt("DB_PORT", 5432)
    if err != nil { return nil, err }

    shutdownTimeout, err := getEnvDuration("SHUTDOWN_TIMEOUT", 15*time.Second)
    if err != nil { return nil, err }

    defaultPort := 8080
    if env == "production" {
        defaultPort = 80
    }
    appPort, err := getEnvInt("PORT", defaultPort)
    if err != nil { return nil, err }

    // 4) 組裝
    cfg := &Config{
        App: AppConfig{
            Env:             env,
            Port:            appPort,
            LogLevel:        getEnvWithDefault("LOG_LEVEL", ""),
            AllowedOrigin:   allowedOrigin,
            ShutdownTimeout: shutdownTimeout,
        },
        DB: DBConfig{
            Host:     dbHost,
            Port:     dbPort,
            User:     dbUser,
            Password: dbPassword,
            Name:     getEnvWithDefault("DB_NAME", "helpzy"),
            SSLMode:  getEnvWithDefault("DB_SSLMODE", "disable"),
        },
        Redis: RedisConfig{URL: redisURL},
        JWT:   JWTConfig{Secret: jwtSecret},
    }

    // 5) 跨欄位驗證
    if err := cfg.Validate(); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

---

## Validate（跨欄位 / 環境相依規則）

單一欄位的格式 / 必填檢查放在 `getEnv*`；**跨欄位、與環境相關的規則**集中於 `Validate()`，於 `LoadConfig` 最後一步呼叫：

```go
// internal/config/config.go
func (c *Config) Validate() error {
    if len(c.JWT.Secret) < 32 {
        return errors.New("JWT_SECRET must be at least 32 chars")
    }

    if c.App.Env == "production" {
        if c.DB.SSLMode == "disable" {
            return errors.New("DB_SSLMODE must not be 'disable' in production")
        }
        if c.DB.Password == "" {
            return errors.New("DB_PASSWORD must not be empty in production")
        }
        if strings.HasPrefix(c.Redis.URL, "redis://") &&
           !strings.Contains(c.Redis.URL, "@") {
            return errors.New("REDIS_URL appears to have no credentials in production")
        }
    }

    return nil
}
```

**規則放這裡 vs. 放 `getEnv*` 的判準**：
- 「單獨檢查就能判定」（必填、格式）→ `getEnv*` 內
- 「需要看其他欄位 / 環境」→ `Validate()`

---

## 敏感欄位遮蔽（String 方法）

```go
// internal/config/config.go
func (c DBConfig) String() string {
    return fmt.Sprintf("DBConfig{Host:%s Port:%d User:%s Password:*** Name:%s SSLMode:%s}",
        c.Host, c.Port, c.User, c.Name, c.SSLMode)
}

func (c JWTConfig) String() string {
    return "JWTConfig{Secret:***}"
}

// Redis URL 可能為 redis://user:pass@host:6379；只遮蔽 userinfo 部分
func (c RedisConfig) String() string {
    u, err := url.Parse(c.URL)
    if err != nil {
        return "RedisConfig{URL:***}"
    }
    if u.User != nil {
        u.User = url.UserPassword(u.User.Username(), "***")
    }
    return fmt.Sprintf("RedisConfig{URL:%s}", u.String())
}
```

---

## .env 檔規範

| 檔案 | 環境 | 是否進版控 |
|------|------|-----------|
| `.env.local` | local 開發 | ✗（`.gitignore`） |
| `.env` | production（容器內注入） | ✗ |
| `.env.test` | 整合測試 | ✓ 提供範本，但不含真實密碼 |
| `configs/.env.example` | 範本，列出所有 env key | ✓ |

---

## main.go 載入

```go
env := os.Getenv("ENV") // 唯一直接讀 env 的地方
cfg, err := config.LoadConfig(env)
if err != nil {
    slog.Error("config load failed", "err", err)
    os.Exit(1)
}
```

---

## 環境變數

| 變數 | 必填 | 預設 | 說明 |
|------|------|------|------|
| `ENV` | ✗ | `local` | 執行環境（`local` / `production` / `test`） |
| `PORT` | ✗ | `8080`（prod `80`） | 伺服器埠號 |
| `LOG_LEVEL` | ✗ | 依 `ENV` | 覆寫 log 等級（`debug` / `info` / `warn` / `error`） |
| `SHUTDOWN_TIMEOUT` | ✗ | `15s` | 優雅關閉等待 in-flight 請求的最長時間 |
| `ALLOWED_ORIGIN` | ✓ | — | 允許的 CORS Origin（BFF 位址） |
| `DB_HOST` | ✓ | — | PostgreSQL host |
| `DB_PORT` | ✗ | `5432` | PostgreSQL port |
| `DB_USER` | ✓ | — | PostgreSQL 使用者 |
| `DB_PASSWORD` | ✓ | — | PostgreSQL 密碼 |
| `DB_NAME` | ✗ | `helpzy` | PostgreSQL 資料庫名稱 |
| `DB_SSLMODE` | ✗ | `disable` | PostgreSQL SSL 模式 |
| `REDIS_URL` | ✓ | — | Redis 連線字串 |
| `JWT_SECRET` | ✓ | — | JWT 簽署金鑰（與 BFF 共享，min 32 chars） |

---

## 測試策略

- `LoadConfig` 以 `t.Setenv` 設定環境變數，斷言：
  - 必填缺漏 → 回傳對應 error message
  - `DB_PORT=abc` / `SHUTDOWN_TIMEOUT=xxx` → 解析錯誤直接 fail-fast，不靜默走 default
  - 可選欄位走 default
  - `production` / `local` Port 預設值差異
- `Validate()` 為純方法，table-driven 覆蓋：
  - JWT secret 長度不足
  - production + `DB_SSLMODE=disable` → 拒絕
  - production + 空 `DB_PASSWORD` → 拒絕
  - local 環境放寬上述規則
- `String()` 斷言 `Password` / `Secret` / Redis 密碼欄位被替換為 `***`
- `normalizeEnv` / `getEnvInt` / `getEnvDuration` 以 table-driven test 覆蓋邊界
