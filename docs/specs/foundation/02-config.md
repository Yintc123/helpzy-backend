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
    App        AppConfig
    DB         DBConfig
    Redis      RedisConfig
    JWT        JWTConfig
    RateLimit  RateLimitConfig
    OTel       OTelConfig
    BcryptCost int
}

type AppConfig struct {
    Env               string        // local / production / test
    Port              int
    LogLevel          string
    AllowedOrigin     string
    ShutdownTimeout   time.Duration
    ReadHeaderTimeout time.Duration
    ReadTimeout       time.Duration
    WriteTimeout      time.Duration
    IdleTimeout       time.Duration
    RequestTimeout    time.Duration
    MaxBodyBytes      int64
}

type DBConfig struct {
    Host            string
    Port            int
    User            string
    Password        string
    Name            string
    SSLMode         string
    MaxConns        int
    MinConns        int
    MaxConnLifetime time.Duration
    MaxConnIdleTime time.Duration
}

type RedisConfig struct {
    URL string
}

type JWTConfig struct {
    Secret     string
    AccessTTL  time.Duration
    RefreshTTL time.Duration
}

// RateLimitConfig：見 [09-middleware.md](./09-middleware.md#ratelimit-middleware)
type RateLimitConfig struct {
    LoginPerMinute    int  // /auth/login 每分鐘 per IP
    RegisterPerHour   int  // /auth/register 每小時 per IP
    RefreshPerMinute  int  // /auth/refresh 每分鐘 per IP
    AuthedPerMinute   int  // 已驗證路由 per user_id 每分鐘
}

// OTelConfig：見 [16-observability.md](./16-observability.md)
type OTelConfig struct {
    Enabled        bool
    Endpoint       string  // OTLP collector，例 otel-collector:4318
    ServiceName    string
    ServiceVersion string  // build 時 ldflags 注入
    SampleRatio    float64 // 0.0–1.0
}
```

> `App.BcryptCost` 也屬於 auth 相關設定，但因為它只是個 int、不歸 `JWTConfig` 管，放在 `AppConfig` 或頂層 `Config` 都可。本規格放在 `Config.BcryptCost` 頂層欄位。

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

// 可選秒數（int）→ time.Duration；解析失敗即 error。
// 所有 duration env 都統一用「整數秒」格式（與 JWT exp / OAuth2 expires_in 對齊），
// 內部以 time.Duration 型別流通。
func getEnvSeconds(key string, defaultSec int) (time.Duration, error) {
    sec, err := getEnvInt(key, defaultSec)
    if err != nil {
        return 0, err
    }
    return time.Duration(sec) * time.Second, nil
}

func normalizeEnv(env string) string {
    switch strings.ToLower(env) {
    case "production", "local", "test":
        return strings.ToLower(env)
    default:
        return "local"
    }
}

// 可選浮點數；解析失敗即 error
func getEnvFloat(key string, defaultValue float64) (float64, error) {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue, nil
    }
    v, err := strconv.ParseFloat(raw, 64)
    if err != nil {
        return 0, fmt.Errorf("env %s: invalid float %q: %w", key, raw, err)
    }
    return v, nil
}

// production 預設較低取樣率以控制 collector 流量
func defaultSampleRatio(env string) float64 {
    if env == "production" {
        return 0.1
    }
    return 1.0
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

    shutdownTimeout, err := getEnvSeconds("SHUTDOWN_TIMEOUT", 15)        // 15s
    if err != nil { return nil, err }

    readHeaderTimeout, err := getEnvSeconds("READ_HEADER_TIMEOUT", 5)    // 5s（Slowloris 防護）
    if err != nil { return nil, err }

    readTimeout, err := getEnvSeconds("READ_TIMEOUT", 30)                // 30s（整個 request 讀取上限）
    if err != nil { return nil, err }

    writeTimeout, err := getEnvSeconds("WRITE_TIMEOUT", 30)              // 30s（response 寫入上限）
    if err != nil { return nil, err }

    idleTimeout, err := getEnvSeconds("IDLE_TIMEOUT", 120)               // 120s（keep-alive 閒置上限）
    if err != nil { return nil, err }

    requestTimeout, err := getEnvSeconds("REQUEST_TIMEOUT", 10)          // 10s（單一請求最長執行時間）
    if err != nil { return nil, err }

    maxBodyBytes, err := getEnvInt("MAX_BODY_BYTES", 1<<20)              // 1 MiB
    if err != nil { return nil, err }

    dbMaxConns, err := getEnvInt("DB_MAX_CONNS", 20)
    if err != nil { return nil, err }

    dbMinConns, err := getEnvInt("DB_MIN_CONNS", 5)
    if err != nil { return nil, err }

    dbMaxConnLifetime, err := getEnvSeconds("DB_MAX_CONN_LIFETIME", 3600) // 1h
    if err != nil { return nil, err }

    dbMaxConnIdleTime, err := getEnvSeconds("DB_MAX_CONN_IDLE_TIME", 1800) // 30min
    if err != nil { return nil, err }

    accessTTL, err := getEnvSeconds("JWT_ACCESS_TTL", 24*60*60)           // 1d
    if err != nil { return nil, err }

    refreshTTL, err := getEnvSeconds("JWT_REFRESH_TTL", 30*24*60*60)      // 30d
    if err != nil { return nil, err }

    bcryptCost, err := getEnvInt("BCRYPT_COST", 12)
    if err != nil { return nil, err }

    rlLogin, err := getEnvInt("RATE_LIMIT_LOGIN_PER_MIN", 5)
    if err != nil { return nil, err }
    rlRegister, err := getEnvInt("RATE_LIMIT_REGISTER_PER_HOUR", 3)
    if err != nil { return nil, err }
    rlRefresh, err := getEnvInt("RATE_LIMIT_REFRESH_PER_MIN", 60)
    if err != nil { return nil, err }
    rlAuthed, err := getEnvInt("RATE_LIMIT_AUTHED_PER_MIN", 600)
    if err != nil { return nil, err }

    otelEnabled := strings.EqualFold(getEnvWithDefault("OTEL_ENABLED", "false"), "true")
    otelEndpoint := getEnvWithDefault("OTEL_EXPORTER_OTLP_ENDPOINT", "")
    if otelEnabled && otelEndpoint == "" {
        return nil, errors.New("OTEL_EXPORTER_OTLP_ENDPOINT required when OTEL_ENABLED=true")
    }
    otelSampleRatio, err := getEnvFloat("OTEL_SAMPLE_RATIO", defaultSampleRatio(env))
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
            Env:               env,
            Port:              appPort,
            LogLevel:          getEnvWithDefault("LOG_LEVEL", ""),
            AllowedOrigin:     allowedOrigin,
            ShutdownTimeout:   shutdownTimeout,
            ReadHeaderTimeout: readHeaderTimeout,
            ReadTimeout:       readTimeout,
            WriteTimeout:      writeTimeout,
            IdleTimeout:       idleTimeout,
            RequestTimeout:    requestTimeout,
            MaxBodyBytes:      int64(maxBodyBytes),
        },
        DB: DBConfig{
            Host:            dbHost,
            Port:            dbPort,
            User:            dbUser,
            Password:        dbPassword,
            Name:            getEnvWithDefault("DB_NAME", "helpzy"),
            SSLMode:         getEnvWithDefault("DB_SSLMODE", "disable"),
            MaxConns:        dbMaxConns,
            MinConns:        dbMinConns,
            MaxConnLifetime: dbMaxConnLifetime,
            MaxConnIdleTime: dbMaxConnIdleTime,
        },
        Redis: RedisConfig{URL: redisURL},
        JWT: JWTConfig{
            Secret:     jwtSecret,
            AccessTTL:  accessTTL,
            RefreshTTL: refreshTTL,
        },
        RateLimit: RateLimitConfig{
            LoginPerMinute:   rlLogin,
            RegisterPerHour:  rlRegister,
            RefreshPerMinute: rlRefresh,
            AuthedPerMinute:  rlAuthed,
        },
        OTel: OTelConfig{
            Enabled:        otelEnabled,
            Endpoint:       otelEndpoint,
            ServiceName:    getEnvWithDefault("OTEL_SERVICE_NAME", "helpzy-backend"),
            ServiceVersion: getEnvWithDefault("OTEL_SERVICE_VERSION", "dev"),
            SampleRatio:    otelSampleRatio,
        },
        BcryptCost: bcryptCost,
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
    if c.BcryptCost < 10 || c.BcryptCost > 14 {
        return errors.New("BCRYPT_COST must be in [10, 14]")
    }
    if c.JWT.AccessTTL > c.JWT.RefreshTTL {
        return errors.New("JWT_ACCESS_TTL must not exceed JWT_REFRESH_TTL")
    }
    if c.DB.MaxConns <= 0 {
        return errors.New("DB_MAX_CONNS must be > 0")
    }
    if c.DB.MinConns < 0 {
        return errors.New("DB_MIN_CONNS must be >= 0")
    }
    if c.DB.MinConns > c.DB.MaxConns {
        return errors.New("DB_MIN_CONNS must not exceed DB_MAX_CONNS")
    }
    if c.App.ReadTimeout < c.App.RequestTimeout {
        return errors.New("READ_TIMEOUT must be >= REQUEST_TIMEOUT")
    }
    if c.App.WriteTimeout < c.App.RequestTimeout {
        return errors.New("WRITE_TIMEOUT must be >= REQUEST_TIMEOUT")
    }
    if c.App.MaxBodyBytes <= 0 {
        return errors.New("MAX_BODY_BYTES must be > 0")
    }
    if c.RateLimit.LoginPerMinute <= 0 || c.RateLimit.RegisterPerHour <= 0 ||
        c.RateLimit.RefreshPerMinute <= 0 || c.RateLimit.AuthedPerMinute <= 0 {
        return errors.New("RATE_LIMIT_* must be > 0")
    }
    if c.OTel.SampleRatio < 0 || c.OTel.SampleRatio > 1 {
        return errors.New("OTEL_SAMPLE_RATIO must be in [0.0, 1.0]")
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

### `configs/.env.example` 範本

```env
# ─── Application ────────────────────────────────────────────────
ENV=local                            # local / production / test
PORT=8080                            # production 預設 80
LOG_LEVEL=                           # 空字串時依 ENV 自動判斷
SHUTDOWN_TIMEOUT=15                  # 秒
READ_HEADER_TIMEOUT=5                # 秒（Slowloris 防護）
READ_TIMEOUT=30                      # 秒（整個 request 讀取上限）
WRITE_TIMEOUT=30                     # 秒（response 寫入上限）
IDLE_TIMEOUT=120                     # 秒（keep-alive 閒置上限）
REQUEST_TIMEOUT=10                   # 秒（單一請求 handler 執行時間）
MAX_BODY_BYTES=1048576               # bytes（1 MiB）
ALLOWED_ORIGIN=http://localhost:3000 # Next.js 前端位址

# ─── PostgreSQL ─────────────────────────────────────────────────
DB_HOST=localhost
DB_PORT=5432
DB_USER=helpzy
DB_PASSWORD=                         # local 可填任意值；production 必填且強密碼
DB_NAME=helpzy
DB_SSLMODE=disable                   # production 必須改為 require / verify-full
DB_MAX_CONNS=20                      # 連線池上限
DB_MIN_CONNS=5                       # 暖連線下限
DB_MAX_CONN_LIFETIME=3600            # 1 小時（秒），避免 DB 端 idle timeout
DB_MAX_CONN_IDLE_TIME=1800           # 30 分鐘（秒），釋放閒置連線

# ─── Redis ──────────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379

# ─── JWT & Password ─────────────────────────────────────────────
JWT_SECRET=                          # 僅 Backend 持有；min 32 chars
JWT_ACCESS_TTL=86400                 # 1 天（秒）
JWT_REFRESH_TTL=2592000              # 30 天（秒）
BCRYPT_COST=12                       # 範圍 10–14

# ─── Rate Limit ─────────────────────────────────────────────────
RATE_LIMIT_LOGIN_PER_MIN=5           # /auth/login 每分鐘 per IP
RATE_LIMIT_REGISTER_PER_HOUR=3       # /auth/register 每小時 per IP
RATE_LIMIT_REFRESH_PER_MIN=60        # /auth/refresh 每分鐘 per IP
RATE_LIMIT_AUTHED_PER_MIN=600        # 已驗證路由 per user_id 每分鐘

# ─── OpenTelemetry ──────────────────────────────────────────────
OTEL_ENABLED=false                   # local 通常 false；staging / production = true
OTEL_EXPORTER_OTLP_ENDPOINT=         # OTEL_ENABLED=true 時必填，例 otel-collector:4318
OTEL_SERVICE_NAME=helpzy-backend
OTEL_SERVICE_VERSION=dev             # 通常 build 時 -ldflags 注入 commit SHA
OTEL_SAMPLE_RATIO=                   # 空字串時 production=0.1、其他=1.0
```

> **守則**：`.env.example` **僅含 key 與註解**，不得寫入真實密碼；`DB_PASSWORD` / `JWT_SECRET` 等敏感欄位一律留空。

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
| `SHUTDOWN_TIMEOUT` | ✗ | `15` | 優雅關閉等待 in-flight 請求的最長時間（秒） |
| `READ_HEADER_TIMEOUT` | ✗ | `5` | `http.Server` 讀取請求 header 的最長時間（秒，Slowloris 防護） |
| `READ_TIMEOUT` | ✗ | `30` | `http.Server` 讀取整個 request（含 body）的最長時間（秒） |
| `WRITE_TIMEOUT` | ✗ | `30` | `http.Server` 寫入 response 的最長時間（秒） |
| `IDLE_TIMEOUT` | ✗ | `120` | `http.Server` keep-alive 連線閒置最長時間（秒） |
| `REQUEST_TIMEOUT` | ✗ | `10` | 單一請求 handler 最長執行時間（秒） |
| `MAX_BODY_BYTES` | ✗ | `1048576` | request body 大小上限（bytes，預設 1 MiB） |
| `ALLOWED_ORIGIN` | ✓ | — | 允許的 CORS Origin（Next.js 前端位址） |
| `DB_HOST` | ✓ | — | PostgreSQL host |
| `DB_PORT` | ✗ | `5432` | PostgreSQL port |
| `DB_USER` | ✓ | — | PostgreSQL 使用者 |
| `DB_PASSWORD` | ✓ | — | PostgreSQL 密碼 |
| `DB_NAME` | ✗ | `helpzy` | PostgreSQL 資料庫名稱 |
| `DB_SSLMODE` | ✗ | `disable` | PostgreSQL SSL 模式 |
| `DB_MAX_CONNS` | ✗ | `20` | 連線池上限 |
| `DB_MIN_CONNS` | ✗ | `5` | 暖連線下限 |
| `DB_MAX_CONN_LIFETIME` | ✗ | `3600` | 連線最長生命週期（秒） |
| `DB_MAX_CONN_IDLE_TIME` | ✗ | `1800` | 連線最長閒置時間（秒） |
| `REDIS_URL` | ✓ | — | Redis 連線字串 |
| `JWT_SECRET` | ✓ | — | JWT 簽署金鑰（僅 Backend 持有，min 32 chars） |
| `JWT_ACCESS_TTL` | ✗ | `86400` | Access token 有效期（秒，預設 1 天） |
| `JWT_REFRESH_TTL` | ✗ | `2592000` | Refresh token 有效期（秒，預設 30 天） |
| `BCRYPT_COST` | ✗ | `12` | 密碼 hash cost factor（範圍 10–14） |
| `RATE_LIMIT_LOGIN_PER_MIN` | ✗ | `5` | `/auth/login` 每分鐘 per IP |
| `RATE_LIMIT_REGISTER_PER_HOUR` | ✗ | `3` | `/auth/register` 每小時 per IP |
| `RATE_LIMIT_REFRESH_PER_MIN` | ✗ | `60` | `/auth/refresh` 每分鐘 per IP |
| `RATE_LIMIT_AUTHED_PER_MIN` | ✗ | `600` | 已驗證路由 per user_id 每分鐘 |
| `OTEL_ENABLED` | ✗ | `false` | 是否啟用 OpenTelemetry |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTEL_ENABLED=true 時必填 | — | OTLP collector 位址 |
| `OTEL_SERVICE_NAME` | ✗ | `helpzy-backend` | service.name resource attribute |
| `OTEL_SERVICE_VERSION` | ✗ | `dev` | build 時 ldflags 注入 commit SHA |
| `OTEL_SAMPLE_RATIO` | ✗ | `1.0`（local）/ `0.1`（prod） | trace 取樣率 [0.0, 1.0] |

---

## 測試策略

- `LoadConfig` 以 `t.Setenv` 設定環境變數，斷言：
  - 必填缺漏 → 回傳對應 error message
  - `DB_PORT=abc` / `SHUTDOWN_TIMEOUT=xxx` / `JWT_ACCESS_TTL=xxx` → 解析錯誤直接 fail-fast，不靜默走 default
  - 可選欄位走 default
  - `production` / `local` Port 預設值差異
- `Validate()` 為純方法，table-driven 覆蓋：
  - JWT secret 長度不足
  - production + `DB_SSLMODE=disable` → 拒絕
  - production + 空 `DB_PASSWORD` → 拒絕
  - local 環境放寬上述規則
- `String()` 斷言 `Password` / `Secret` / Redis 密碼欄位被替換為 `***`
- `normalizeEnv` / `getEnvInt` / `getEnvSeconds` 以 table-driven test 覆蓋邊界
