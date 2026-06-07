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
    WSTicket   WSTicketConfig
    WS         WSConfig
    Visitor    VisitorConfig
    LLM        LLMConfig
    Chat       ChatConfig
    RateLimit  RateLimitConfig
    OTel       OTelConfig
    BcryptCost int
}

type AppConfig struct {
    Env               string        // local / production / test
    Port              int
    LogLevel          string
    AllowedOrigins    []string      // CORS 允許的 Origin 白名單（逗號分隔，精確比對；不支援萬用字元）
    ShutdownTimeout   time.Duration
    ReadHeaderTimeout time.Duration
    ReadTimeout       time.Duration
    WriteTimeout      time.Duration
    IdleTimeout       time.Duration
    RequestTimeout    time.Duration // 一般受保護端點的 per-request timeout
    MaxBodyBytes      int64

    // 分群 timeout：不同端點群組的 SLA 不同，獨立調整
    AuthRequestTimeout      time.Duration // /auth/* (login/refresh/...)
    AnonymousRequestTimeout time.Duration // /anonymous/* (匿名 ws-ticket)
    HealthPingTimeout       time.Duration // /healthz / /readyz 內 Pinger 的 timeout

    // 安全 / CORS header 參數（值會隨環境變動）
    HSTSMaxAge int // SecureHeaders 的 Strict-Transport-Security max-age（秒）
    CORSMaxAge int // CORS preflight cache（秒）
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

// WSTicketConfig：機制層見 [15-auth-mechanism.md](./15-auth-mechanism.md)；
// 業務用途見 [../features/auth-feature-spec.md](../features/auth-feature-spec.md)
type WSTicketConfig struct {
    Secret string        // IP hash 用的 secret，min 32 chars
    TTL    time.Duration // 預設 30 秒
    BindIP bool          // 是否啟用 IP fingerprint 綁定
}

// VisitorConfig：政策層型別與 cookie 屬性見 [../features/auth-feature-spec.md](../features/auth-feature-spec.md)
type VisitorConfig struct {
    CookieSecret string        // visitor cookie HMAC secret，min 32 chars
    TTL          time.Duration // 預設 30 天
}

// WSConfig：WebSocket 升級與連線層設定，見 [chat-spec.md](../chat-spec.md)
type WSConfig struct {
    AllowedOrigins []string      // 升級時驗 Origin 的白名單（精確比對；不支援萬用字元）
    PingInterval   time.Duration
    ReadTimeout    time.Duration
}

// LLMConfig：見 [17-llm.md](./17-llm.md)
type LLMConfig struct {
    Provider           string        // Registry fallback 名稱（claude / openai / mock 等）
    RequestTimeout     time.Duration
    MaxOutputTokens    int           // 單次回應最大 token 數（成本上限 / 換 model 時需調）
    ClaudeAPIKey       string
    ClaudeDefaultModel string
    ClaudeAPIBaseURL   string        // 測試 / proxy 用；預設 https://api.anthropic.com
}

// ChatConfig：見 [../chat-spec.md](../chat-spec.md)
type ChatConfig struct {
    EscalationKeywords []string      // 觸發 handoff 的關鍵字
    AgentQueueTTL      time.Duration
    MaxHistoryTokens   int           // 餵給 LLM 的歷史最大 token 數，超過截斷舊訊息
    SessionIdleTTL     time.Duration // Session 雙端離線後 in-memory 保留時間
    BrandName          string        // system prompt 內品牌名（multi-tenant / 白標時必調）
    InboundChanBuffer  int           // Session inbound chan 緩衝；高並發 throughput tuning
}

// RateLimitConfig：見 [09-middleware.md](./09-middleware.md#ratelimit-middleware)
type RateLimitConfig struct {
    LoginPerMinute           int // /auth/login 每分鐘 per IP
    RegisterPerHour          int // /auth/register 每小時 per IP
    RefreshPerMinute         int // /auth/refresh 每分鐘 per IP
    AuthedPerMinute          int // 已驗證路由 per user_id 每分鐘
    AnonymousTicketPerMinute int // /anonymous/ws-ticket 每分鐘 per IP（防匿名訪客濫發）
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

// 可選 bool（true / false / 1 / 0，不分大小寫）；空字串走 default；其他值 → error
func getEnvBool(key string, defaultValue bool) (bool, error) {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue, nil
    }
    switch strings.ToLower(raw) {
    case "true", "1":
        return true, nil
    case "false", "0":
        return false, nil
    default:
        return false, fmt.Errorf("env %s: invalid bool %q (expect true/false/1/0)", key, raw)
    }
}

// 可選逗號分隔字串清單；空字串走 default；自動 trim 各段空白與略過空項。
// 用於 WS_ALLOWED_ORIGINS、CHAT_ESCALATION_KEYWORDS 等。
func getEnvStringList(key string, defaultValue []string) []string {
    raw := os.Getenv(key)
    if raw == "" {
        return defaultValue
    }
    parts := strings.Split(raw, ",")
    out := make([]string, 0, len(parts))
    for _, p := range parts {
        if t := strings.TrimSpace(p); t != "" {
            out = append(out, t)
        }
    }
    if len(out) == 0 {
        return defaultValue
    }
    return out
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
    allowedOrigins := getEnvStringList("ALLOWED_ORIGINS", nil)
    if len(allowedOrigins) == 0 {
        return nil, fmt.Errorf("missing required env: ALLOWED_ORIGINS")
    }

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

    authReqTimeout, err := getEnvSeconds("AUTH_REQUEST_TIMEOUT", 10)     // 10s（/auth/* 群組）
    if err != nil { return nil, err }

    anonReqTimeout, err := getEnvSeconds("ANONYMOUS_REQUEST_TIMEOUT", 5) // 5s（/anonymous/* 群組）
    if err != nil { return nil, err }

    healthPingTimeout, err := getEnvSeconds("HEALTH_PING_TIMEOUT", 2)    // 2s（Pinger ctx 上限）
    if err != nil { return nil, err }

    hstsMaxAge, err := getEnvInt("HSTS_MAX_AGE", 31536000)               // 1 年
    if err != nil { return nil, err }

    corsMaxAge, err := getEnvInt("CORS_MAX_AGE", 300)                    // 5 分鐘 preflight cache
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
    rlAnonTicket, err := getEnvInt("RATE_LIMIT_ANONYMOUS_TICKET_PER_MIN", 30)
    if err != nil { return nil, err }

    // WSTicket
    wsTicketSecret, err := getEnv("WS_TICKET_SECRET")
    if err != nil { return nil, err }
    wsTicketTTL, err := getEnvSeconds("WS_TICKET_TTL", 30)             // 30s
    if err != nil { return nil, err }
    wsTicketBindIP, err := getEnvBool("WS_TICKET_BIND_IP", true)
    if err != nil { return nil, err }

    // WS server-side
    wsAllowedOrigins := getEnvStringList("WS_ALLOWED_ORIGINS", nil)
    wsPingInterval, err := getEnvSeconds("WS_PING_INTERVAL", 30)        // 30s
    if err != nil { return nil, err }
    wsReadTimeout, err := getEnvSeconds("WS_READ_TIMEOUT", 60)          // 60s
    if err != nil { return nil, err }

    // Visitor cookie（匿名訪客）
    visitorSecret, err := getEnv("VISITOR_COOKIE_SECRET")
    if err != nil { return nil, err }
    visitorTTL, err := getEnvSeconds("VISITOR_TTL", 30*24*60*60) // 30d
    if err != nil { return nil, err }

    // LLM
    llmProvider := getEnvWithDefault("LLM_PROVIDER", "claude")
    llmRequestTimeout, err := getEnvSeconds("LLM_REQUEST_TIMEOUT", 60)  // 60s
    if err != nil { return nil, err }
    llmMaxOutputTokens, err := getEnvInt("LLM_MAX_OUTPUT_TOKENS", 2048)
    if err != nil { return nil, err }
    claudeAPIKey := os.Getenv("CLAUDE_API_KEY") // 至少一個 provider key，於 Validate 強制
    claudeDefaultModel := getEnvWithDefault("CLAUDE_DEFAULT_MODEL", "claude-haiku-4-5")
    claudeAPIBaseURL := getEnvWithDefault("CLAUDE_API_BASE_URL", "https://api.anthropic.com")

    // Chat
    chatKeywords := getEnvStringList("CHAT_ESCALATION_KEYWORDS",
        []string{"找真人", "客服", "人工"})
    chatAgentQueueTTL, err := getEnvSeconds("CHAT_AGENT_QUEUE_TTL", 300) // 5m
    if err != nil { return nil, err }
    chatMaxHistTokens, err := getEnvInt("CHAT_MAX_HISTORY_TOKENS", 8000)
    if err != nil { return nil, err }
    chatSessionIdleTTL, err := getEnvSeconds("CHAT_SESSION_IDLE_TTL", 600) // 10m
    if err != nil { return nil, err }
    chatBrandName := getEnvWithDefault("CHAT_BRAND_NAME", "Helpzy")
    chatInboundBuf, err := getEnvInt("CHAT_INBOUND_CHAN_BUFFER", 64)
    if err != nil { return nil, err }

    otelEnabled := strings.EqualFold(getEnvWithDefault("OTEL_ENABLED", "false"), "true")
    otelEndpoint := getEnvWithDefault("OTEL_EXPORTER_OTLP_ENDPOINT", "")
    if otelEnabled && otelEndpoint == "" {
        return nil, errors.New("OTEL_EXPORTER_OTLP_ENDPOINT required when OTEL_ENABLED=true")
    }
    // 反向情境：endpoint 設了但 enabled=false。
    // 不 fail-fast（無害；常見於開發者複製 prod env 後忘記改 OTEL_ENABLED），
    // 但記一條 warn 提醒 — 避免「為什麼我的 trace 都沒送出」的困惑。
    if !otelEnabled && otelEndpoint != "" {
        slog.Warn("OTEL_EXPORTER_OTLP_ENDPOINT 已設定但 OTEL_ENABLED=false，將忽略 endpoint")
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
            Env:                     env,
            Port:                    appPort,
            LogLevel:                getEnvWithDefault("LOG_LEVEL", ""),
            AllowedOrigins:          allowedOrigins,
            ShutdownTimeout:         shutdownTimeout,
            ReadHeaderTimeout:       readHeaderTimeout,
            ReadTimeout:             readTimeout,
            WriteTimeout:            writeTimeout,
            IdleTimeout:             idleTimeout,
            RequestTimeout:          requestTimeout,
            MaxBodyBytes:            int64(maxBodyBytes),
            AuthRequestTimeout:      authReqTimeout,
            AnonymousRequestTimeout: anonReqTimeout,
            HealthPingTimeout:       healthPingTimeout,
            HSTSMaxAge:              hstsMaxAge,
            CORSMaxAge:              corsMaxAge,
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
        WSTicket: WSTicketConfig{
            Secret: wsTicketSecret,
            TTL:    wsTicketTTL,
            BindIP: wsTicketBindIP,
        },
        WS: WSConfig{
            AllowedOrigins: wsAllowedOrigins,
            PingInterval:   wsPingInterval,
            ReadTimeout:    wsReadTimeout,
        },
        Visitor: VisitorConfig{
            CookieSecret: visitorSecret,
            TTL:          visitorTTL,
        },
        LLM: LLMConfig{
            Provider:           llmProvider,
            RequestTimeout:     llmRequestTimeout,
            MaxOutputTokens:    llmMaxOutputTokens,
            ClaudeAPIKey:       claudeAPIKey,
            ClaudeDefaultModel: claudeDefaultModel,
            ClaudeAPIBaseURL:   claudeAPIBaseURL,
        },
        Chat: ChatConfig{
            EscalationKeywords: chatKeywords,
            AgentQueueTTL:      chatAgentQueueTTL,
            MaxHistoryTokens:   chatMaxHistTokens,
            SessionIdleTTL:     chatSessionIdleTTL,
            BrandName:          chatBrandName,
            InboundChanBuffer:  chatInboundBuf,
        },
        RateLimit: RateLimitConfig{
            LoginPerMinute:           rlLogin,
            RegisterPerHour:          rlRegister,
            RefreshPerMinute:         rlRefresh,
            AuthedPerMinute:          rlAuthed,
            AnonymousTicketPerMinute: rlAnonTicket,
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
        c.RateLimit.RefreshPerMinute <= 0 || c.RateLimit.AuthedPerMinute <= 0 ||
        c.RateLimit.AnonymousTicketPerMinute <= 0 {
        return errors.New("RATE_LIMIT_* must be > 0")
    }
    if c.OTel.SampleRatio < 0 || c.OTel.SampleRatio > 1 {
        return errors.New("OTEL_SAMPLE_RATIO must be in [0.0, 1.0]")
    }

    // WSTicket
    if len(c.WSTicket.Secret) < 32 {
        return errors.New("WS_TICKET_SECRET must be at least 32 chars")
    }
    if c.WSTicket.TTL <= 0 || c.WSTicket.TTL > 5*time.Minute {
        return errors.New("WS_TICKET_TTL must be in (0, 300] seconds")
    }

    // Visitor
    if len(c.Visitor.CookieSecret) < 32 {
        return errors.New("VISITOR_COOKIE_SECRET must be at least 32 chars")
    }
    if c.Visitor.TTL <= 0 {
        return errors.New("VISITOR_TTL must be > 0")
    }

    // WS server
    if c.WS.PingInterval <= 0 {
        return errors.New("WS_PING_INTERVAL must be > 0")
    }
    if c.WS.ReadTimeout < c.WS.PingInterval {
        return errors.New("WS_READ_TIMEOUT must be >= WS_PING_INTERVAL")
    }

    // LLM：至少一個 provider 有 key
    switch c.LLM.Provider {
    case "claude":
        if c.LLM.ClaudeAPIKey == "" {
            return errors.New("CLAUDE_API_KEY required when LLM_PROVIDER=claude")
        }
    case "mock":
        // 測試用，允許無 key
    default:
        return fmt.Errorf("unknown LLM_PROVIDER: %q", c.LLM.Provider)
    }
    if c.LLM.RequestTimeout <= 0 {
        return errors.New("LLM_REQUEST_TIMEOUT must be > 0")
    }

    // Chat
    if c.Chat.MaxHistoryTokens <= 0 {
        return errors.New("CHAT_MAX_HISTORY_TOKENS must be > 0")
    }
    if c.Chat.AgentQueueTTL <= 0 {
        return errors.New("CHAT_AGENT_QUEUE_TTL must be > 0")
    }
    if c.Chat.SessionIdleTTL <= 0 {
        return errors.New("CHAT_SESSION_IDLE_TTL must be > 0")
    }
    if strings.TrimSpace(c.Chat.BrandName) == "" {
        return errors.New("CHAT_BRAND_NAME must not be empty")
    }
    if c.Chat.InboundChanBuffer <= 0 {
        return errors.New("CHAT_INBOUND_CHAN_BUFFER must be > 0")
    }

    // LLM (續)
    if c.LLM.MaxOutputTokens <= 0 {
        return errors.New("LLM_MAX_OUTPUT_TOKENS must be > 0")
    }

    // App (新增分群 timeout / header 參數)
    if c.App.AuthRequestTimeout <= 0 {
        return errors.New("AUTH_REQUEST_TIMEOUT must be > 0")
    }
    if c.App.AnonymousRequestTimeout <= 0 {
        return errors.New("ANONYMOUS_REQUEST_TIMEOUT must be > 0")
    }
    if c.App.HealthPingTimeout <= 0 {
        return errors.New("HEALTH_PING_TIMEOUT must be > 0")
    }
    if c.App.HSTSMaxAge < 0 {
        // 允許 0（明確不送 HSTS），但不可為負
        return errors.New("HSTS_MAX_AGE must be >= 0")
    }
    if c.App.Env == "production" && c.App.HSTSMaxAge == 0 {
        return errors.New("HSTS_MAX_AGE must be > 0 in production")
    }
    if c.App.CORSMaxAge < 0 {
        return errors.New("CORS_MAX_AGE must be >= 0")
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
        if len(c.WS.AllowedOrigins) == 0 {
            return errors.New("WS_ALLOWED_ORIGINS must not be empty in production")
        }
        for _, o := range c.WS.AllowedOrigins {
            if !strings.HasPrefix(o, "https://") && !strings.HasPrefix(o, "wss://") {
                return fmt.Errorf("WS_ALLOWED_ORIGINS entry %q must use https/wss in production", o)
            }
        }
        // CORS：production 必須走 https
        for _, o := range c.App.AllowedOrigins {
            if !strings.HasPrefix(o, "https://") {
                return fmt.Errorf("ALLOWED_ORIGINS entry %q must use https in production", o)
            }
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

func (c WSTicketConfig) String() string {
    return fmt.Sprintf("WSTicketConfig{Secret:*** TTL:%s BindIP:%t}",
        c.TTL, c.BindIP)
}

func (c VisitorConfig) String() string {
    return fmt.Sprintf("VisitorConfig{CookieSecret:*** TTL:%s}", c.TTL)
}

func (c LLMConfig) String() string {
    return fmt.Sprintf(
        "LLMConfig{Provider:%s RequestTimeout:%s ClaudeAPIKey:*** ClaudeDefaultModel:%s ClaudeAPIBaseURL:%s}",
        c.Provider, c.RequestTimeout, c.ClaudeDefaultModel, c.ClaudeAPIBaseURL,
    )
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
AUTH_REQUEST_TIMEOUT=10              # 秒（/auth/* 群組；DB 慢時可調長）
ANONYMOUS_REQUEST_TIMEOUT=5          # 秒（/anonymous/* 群組；冷啟動可調長）
HEALTH_PING_TIMEOUT=2                # 秒（/healthz / /readyz Pinger）
HSTS_MAX_AGE=31536000                # 秒（1 年）；首次上線安全期建議 300 逐步調高
CORS_MAX_AGE=300                     # 秒（preflight 快取）
MAX_BODY_BYTES=1048576               # bytes（1 MiB）
ALLOWED_ORIGINS=http://localhost:3000 # 逗號分隔；production 必填且每項須為 https://

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
RATE_LIMIT_ANONYMOUS_TICKET_PER_MIN=30 # /anonymous/ws-ticket 每分鐘 per IP（防匿名濫發）

# ─── OpenTelemetry ──────────────────────────────────────────────
OTEL_ENABLED=false                   # local 通常 false；staging / production = true
OTEL_EXPORTER_OTLP_ENDPOINT=         # OTEL_ENABLED=true 時必填，例 otel-collector:4318
OTEL_SERVICE_NAME=helpzy-backend
OTEL_SERVICE_VERSION=dev             # 通常 build 時 -ldflags 注入 commit SHA
OTEL_SAMPLE_RATIO=                   # 空字串時 production=0.1、其他=1.0

# ─── WebSocket Ticket ───────────────────────────────────────────
WS_TICKET_SECRET=                    # IP fingerprint hash 用；min 32 chars
WS_TICKET_TTL=30                     # ticket 有效期（秒，上限 300）
WS_TICKET_BIND_IP=true               # 雲端 LB / 多層 CDN 可關閉

# ─── WebSocket Server ───────────────────────────────────────────
WS_ALLOWED_ORIGINS=http://localhost:3000   # 逗號分隔；production 必填 https/wss
WS_PING_INTERVAL=30                  # 心跳間隔（秒）
WS_READ_TIMEOUT=60                   # 讀取逾時（秒，>= WS_PING_INTERVAL）

# ─── Visitor Cookie（匿名訪客）───────────────────────────────────
VISITOR_COOKIE_SECRET=               # HMAC secret，min 32 chars
VISITOR_TTL=2592000                  # 30 天（秒）

# ─── LLM Provider ───────────────────────────────────────────────
LLM_PROVIDER=claude                  # registry fallback：claude / mock（未來：openai / ollama）
LLM_REQUEST_TIMEOUT=60               # 單次請求逾時（秒）
LLM_MAX_OUTPUT_TOKENS=2048           # 單次回應 token 上限（成本與延遲控制）
CLAUDE_API_KEY=                      # 當 LLM_PROVIDER=claude 必填
CLAUDE_DEFAULT_MODEL=claude-haiku-4-5
CLAUDE_API_BASE_URL=https://api.anthropic.com  # 測試 / proxy 用

# ─── Chat ───────────────────────────────────────────────────────
CHAT_ESCALATION_KEYWORDS=找真人,客服,人工  # 逗號分隔，空字串走預設
CHAT_AGENT_QUEUE_TTL=300              # 待接手對話逾時（秒）
CHAT_MAX_HISTORY_TOKENS=8000          # 餵 LLM 的歷史 token 上限
CHAT_SESSION_IDLE_TTL=600             # Session 雙端離線後 in-memory 保留時間（秒）
CHAT_BRAND_NAME=Helpzy                # system prompt 內品牌名（multi-tenant / 白標時需調）
CHAT_INBOUND_CHAN_BUFFER=64           # Session inbound chan 緩衝大小
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
| `AUTH_REQUEST_TIMEOUT` | ✗ | `10` | `/auth/*` 群組 timeout（秒） |
| `ANONYMOUS_REQUEST_TIMEOUT` | ✗ | `5` | `/anonymous/*` 群組 timeout（秒） |
| `HEALTH_PING_TIMEOUT` | ✗ | `2` | `/healthz` / `/readyz` 內 Pinger ctx timeout（秒） |
| `HSTS_MAX_AGE` | ✗ | `31536000` | HSTS `max-age`（秒，production 必 > 0） |
| `CORS_MAX_AGE` | ✗ | `300` | CORS preflight cache `Access-Control-Max-Age`（秒） |
| `MAX_BODY_BYTES` | ✗ | `1048576` | request body 大小上限（bytes，預設 1 MiB） |
| `ALLOWED_ORIGINS` | ✓ | — | 允許的 CORS Origin 白名單（逗號分隔；production 每項須為 `https://`） |
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
| `RATE_LIMIT_ANONYMOUS_TICKET_PER_MIN` | ✗ | `30` | `/anonymous/ws-ticket` 每分鐘 per IP |
| `OTEL_ENABLED` | ✗ | `false` | 是否啟用 OpenTelemetry |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTEL_ENABLED=true 時必填 | — | OTLP collector 位址 |
| `OTEL_SERVICE_NAME` | ✗ | `helpzy-backend` | service.name resource attribute |
| `OTEL_SERVICE_VERSION` | ✗ | `dev` | build 時 ldflags 注入 commit SHA |
| `OTEL_SAMPLE_RATIO` | ✗ | `1.0`（local）/ `0.1`（prod） | trace 取樣率 [0.0, 1.0] |
| `WS_TICKET_SECRET` | ✓ | — | WS Ticket IP hash 用 secret，min 32 chars |
| `WS_TICKET_TTL` | ✗ | `30` | WS Ticket 有效期（秒，上限 300） |
| `WS_TICKET_BIND_IP` | ✗ | `true` | 是否啟用 IP fingerprint 綁定 |
| `WS_ALLOWED_ORIGINS` | production 必填 | — | WS 升級允許的 Origin 白名單（逗號分隔） |
| `WS_PING_INTERVAL` | ✗ | `30` | WebSocket 心跳間隔（秒） |
| `WS_READ_TIMEOUT` | ✗ | `60` | WebSocket 讀取逾時（秒，>= ping interval） |
| `VISITOR_COOKIE_SECRET` | ✓ | — | Visitor cookie HMAC secret，min 32 chars |
| `VISITOR_TTL` | ✗ | `2592000`（30 天） | Visitor cookie 有效期（秒） |
| `LLM_PROVIDER` | ✗ | `claude` | LLM Registry fallback 名稱 |
| `LLM_REQUEST_TIMEOUT` | ✗ | `60` | LLM 單次請求逾時（秒） |
| `LLM_MAX_OUTPUT_TOKENS` | ✗ | `2048` | LLM 單次回應 token 上限 |
| `CLAUDE_API_KEY` | `LLM_PROVIDER=claude` 時必填 | — | Anthropic API key |
| `CLAUDE_DEFAULT_MODEL` | ✗ | `claude-haiku-4-5` | Claude 預設模型 |
| `CLAUDE_API_BASE_URL` | ✗ | `https://api.anthropic.com` | Claude API base URL（測試 / proxy 用） |
| `CHAT_ESCALATION_KEYWORDS` | ✗ | `找真人,客服,人工` | 觸發 handoff 的關鍵字（逗號分隔） |
| `CHAT_AGENT_QUEUE_TTL` | ✗ | `300` | 待接手對話逾時（秒） |
| `CHAT_MAX_HISTORY_TOKENS` | ✗ | `8000` | 餵給 LLM 的歷史最大 token 數 |
| `CHAT_SESSION_IDLE_TTL` | ✗ | `600` | Session 雙端離線後 in-memory 保留時間（秒） |
| `CHAT_BRAND_NAME` | ✗ | `Helpzy` | system prompt 內品牌名 |
| `CHAT_INBOUND_CHAN_BUFFER` | ✗ | `64` | Session inbound chan 緩衝（高並發 throughput tuning） |

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
- `String()` 斷言 `Password` / `Secret` / Redis 密碼欄位 / `CLAUDE_API_KEY` / `WS_TICKET_SECRET` 被替換為 `***`
- `normalizeEnv` / `getEnvInt` / `getEnvSeconds` / `getEnvBool` / `getEnvStringList` 以 table-driven test 覆蓋邊界
  - `getEnvBool`：`"true"` / `"TRUE"` / `"1"` → true、`"false"` / `"0"` → false、`""` → default、其他 → error
  - `getEnvStringList`：逗號分隔 trim、空項略過、整段空 → default、CSV 內含中文（UTF-8）
- `Validate()` 新增覆蓋：
  - `WS_TICKET_SECRET` < 32 chars → error
  - `WS_TICKET_TTL` = 0 / > 300 → error
  - `WS_READ_TIMEOUT` < `WS_PING_INTERVAL` → error
  - `LLM_PROVIDER=claude` 且 `CLAUDE_API_KEY=""` → error
  - `LLM_PROVIDER=mock` 即使無 key 也 OK
  - `LLM_PROVIDER=unknown` → error
  - production + 空 `WS_ALLOWED_ORIGINS` → error
  - production + `WS_ALLOWED_ORIGINS` 含 `http://` 開頭 → error
  - 空 `ALLOWED_ORIGINS` → error（必填，所有環境皆需）
  - production + `ALLOWED_ORIGINS` 含 `http://` 開頭 → error
  - local + `ALLOWED_ORIGINS=http://localhost:3000` → OK
  - `VISITOR_COOKIE_SECRET` < 32 chars → error
  - `VISITOR_TTL` = 0 → error
  - `RATE_LIMIT_ANONYMOUS_TICKET_PER_MIN` = 0 → error
  - `AUTH_REQUEST_TIMEOUT` / `ANONYMOUS_REQUEST_TIMEOUT` / `HEALTH_PING_TIMEOUT` = 0 → error
  - `HSTS_MAX_AGE` < 0 → error；production + `HSTS_MAX_AGE=0` → error；local + `HSTS_MAX_AGE=0` → OK
  - `CORS_MAX_AGE` < 0 → error；CORS_MAX_AGE = 0 → OK（明確禁用 preflight cache）
  - `LLM_MAX_OUTPUT_TOKENS` = 0 → error
  - `CHAT_BRAND_NAME` = `""` 或全空白 → error
  - `CHAT_INBOUND_CHAN_BUFFER` = 0 → error
  - `CHAT_SESSION_IDLE_TTL` = 0 → error
- OTel 警告：
  - `OTEL_ENABLED=false` + `OTEL_EXPORTER_OTLP_ENDPOINT=non-empty` → `LoadConfig` 成功（不 fail-fast），但會輸出 `slog.Warn`；用 `slog` testing handler 攔截斷言 warn 被觸發
  - `OTEL_ENABLED=true` + 空 endpoint → error（已存在的規則）
