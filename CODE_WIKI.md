# New API Code Wiki

## 项目概述

New API 是一个基于 Go 语言开发的新一代 LLM 网关和 AI 资源管理系统，作为 One API 的分支项目，提供了更强大的 AI 模型聚合和统一接入能力。

**项目地址**: https://github.com/Calcium-Ion/new-api

**技术栈**:
- 后端: Go 1.25+
- Web框架: Gin
- 数据库: SQLite/MySQL/PostgreSQL
- 前端: React + TypeScript
- ORM: GORM

---

## 目录结构

```
/workspace/
├── main.go                 # 应用入口
├── common/                 # 公共工具和常量
├── constant/               # 常量定义
├── controller/             # 控制器层
├── dto/                    # 数据传输对象
├── i18n/                   # 国际化
├── logger/                 # 日志模块
├── middleware/             # 中间件
├── model/                  # 数据模型层
├── oauth/                  # OAuth认证
├── pkg/                    # 公共包
│   ├── billingexpr/        # 计费表达式引擎
│   ├── cachex/             # 缓存管理
│   ├── ionet/              # IO.NET集成
│   └── perf_metrics/       # 性能指标
├── relay/                  # 中继/转发核心
│   ├── channel/            # 渠道适配器
│   ├── common/             # 中继公共模块
│   ├── common_handler/     # 通用处理器
│   └── constant/           # 中继常量
├── router/                 # 路由配置
├── service/                # 业务服务层
├── setting/                # 配置管理
├── types/                 # 类型定义
├── web/                    # 前端资源
│   ├── classic/           # 经典UI
│   └── default/           # 新版UI
└── electron/              # Electron桌面应用
```

---

## 核心模块详解

### 1. 入口文件 (main.go)

**位置**: `/workspace/main.go`

**主要功能**:
- 初始化资源和服务
- 配置Gin HTTP服务器
- 注册路由
- 启动后台任务

**关键初始化流程**:
```go
func InitResources() error {
    - godotenv.Load(".env")           // 加载环境变量
    - logger.SetupLogger()            // 初始化日志
    - ratio_setting.InitRatioSettings()// 初始化费率设置
    - service.InitHttpClient()        // 初始化HTTP客户端
    - service.InitTokenEncoders()     // 初始化Token编码器
    - model.InitDB()                  // 初始化数据库
    - common.InitRedisClient()        // 初始化Redis
    - i18n.Init()                     // 初始化国际化
    - oauth.LoadCustomProviders()     // 加载自定义OAuth提供商
}
```

**后台任务**:
- `model.SyncChannelCache()` - 同步渠道缓存
- `model.SyncOptions()` - 热更新配置
- `model.UpdateQuotaData()` - 更新配额数据
- `controller.AutomaticallyTestChannels()` - 自动测试渠道
- `service.StartCodexCredentialAutoRefreshTask()` - Codex凭证自动刷新
- `service.StartSubscriptionQuotaResetTask()` - 订阅配额重置

### 2. 路由系统 (router/)

**位置**: `/workspace/router/`

#### 2.1 主路由 (main.go)
```go
func SetRouter(router *gin.Engine, assets ThemeAssets)
```
设置所有子路由:
- API路由 (`/api`)
- Dashboard路由
- 中继路由 (`/v1/*`)
- 视频路由
- Web路由

#### 2.2 API路由 (api-router.go)
包含以下路由组:

| 路由组 | 前缀 | 认证要求 | 说明 |
|--------|------|----------|------|
| 用户认证 | `/api/user/*` | UserAuth | 注册、登录、登出 |
| 用户管理 | `/api/user/self/*` | UserAuth | 用户个人信息管理 |
| 管理员 | `/api/user/admin/*` | AdminAuth | 用户CRUD |
| 渠道管理 | `/api/channel/*` | AdminAuth | 渠道CRUD |
| Token管理 | `/api/token/*` | UserAuth | API Token管理 |
| 订阅管理 | `/api/subscription/*` | UserAuth | 订阅套餐 |
| OAuth | `/api/oauth/*` | - | OAuth认证 |
| 性能监控 | `/api/performance/*` | RootAuth | 性能统计 |
| 模型管理 | `/api/models/*` | AdminAuth | 模型元数据 |

### 3. 数据模型层 (model/)

**位置**: `/workspace/model/`

#### 3.1 数据库初始化 (main.go)

```go
func InitDB() error
func InitLogDB() error
func migrateDB() error
```

**支持的数据库**:
- SQLite (默认)
- MySQL (需要设置 `SQL_DSN`)
- PostgreSQL (需要设置 `SQL_DSN`)

**数据库连接池配置**:
- `SQL_MAX_IDLE_CONNS`: 最大空闲连接数 (默认100)
- `SQL_MAX_OPEN_CONNS`: 最大打开连接数 (默认1000)
- `SQL_MAX_LIFETIME`: 连接最大生命周期 (默认60秒)

#### 3.2 核心数据模型

| 模型名 | 文件 | 说明 |
|--------|------|------|
| Channel | channel.go | AI渠道配置 |
| Token | token.go | API访问令牌 |
| User | user.go | 用户账户 |
| Ability | ability.go | 模型能力映射 |
| Log | log.go | 请求日志 |
| TopUp | topup.go | 充值记录 |
| Task | task.go | 异步任务 |
| SubscriptionPlan | subscription.go | 订阅套餐 |
| CustomOAuthProvider | custom_oauth_provider.go | 自定义OAuth提供商 |

#### 3.3 渠道缓存 (channel_cache.go)

```go
type ChannelCache struct {
    Items      map[int64]*ChannelCacheItem  // channelId -> item
    ByGroup    map[string][]int64            // group -> channelIds
    ByModel    map[string][]int64           // model -> channelIds
    Priority   map[int64]int64              // channelId -> priority
    LastUpdate time.Time
}
```

**功能**:
- 内存缓存渠道配置
- 按分组和模型快速查找
- 自动同步 (`SyncChannelCache`)

### 4. 中继系统 (relay/)

**位置**: `/workspace/relay/`

#### 4.1 中继适配器接口 (channel/adapter.go)

```go
type Adaptor interface {
    Init(info *relaycommon.RelayInfo)
    GetRequestURL(info *relaycommon.RelayInfo) (string, error)
    SetupRequestHeader(c *gin.Context, req *http.Header, info *relaycommon.RelayInfo) error
    ConvertOpenAIRequest(...) (any, error)
    ConvertClaudeRequest(...) (any, error)
    ConvertGeminiRequest(...) (any, error)
    DoRequest(...) (any, error)
    DoResponse(...) (usage any, err *types.NewAPIError)
    GetModelList() []string
    GetChannelName() string
}
```

#### 4.2 支持的渠道类型 (constant/channel.go)

```go
const (
    ChannelTypeOpenAI = 1
    ChannelTypeMidjourney = 2
    ChannelTypeAzure = 3
    ChannelTypeOllama = 4
    ChannelTypeAnthropic = 14
    ChannelTypeGemini = 24
    ChannelTypeDeepSeek = 43
    // ... 共50+种渠道类型
)
```

#### 4.3 渠道适配器列表 (relay/channel/)

| 渠道 | 包名 | 说明 |
|------|------|------|
| OpenAI兼容 | openai | 标准OpenAI API |
| Claude | claude | Anthropic Claude |
| Gemini | gemini | Google Gemini |
| Azure | azure | Microsoft Azure |
| DeepSeek | deepseek | 深度求索 |
| 阿里云 | ali | 通义千问 |
| 百度 | baidu/baidu_v2 | 文心一言 |
| 腾讯 | tencent | 混元 |
| Cohere | cohere | Embedding/Rerank |
| Ollama | ollama | 本地模型 |
| MiniMax | minimax | 语音/图像 |
| 火山引擎 | volcengine | 语音合成 |
| ... | ... | 更多渠道 |

#### 4.4 中继信息 (relay/common/relay_info.go)

```go
type RelayInfo struct {
    ChannelId           int64
    ChannelType         int
    ChannelBaseUrl      string
    ApiKey              string
    OriginModelName     string        // 用户请求的模型名
    UpstreamModelName   string        // 上游实际的模型名
    RelayMode           int           // 中继模式
    RelayFormat         string        // 中继格式
    IsStream            bool
    StartTime           time.Time
    UserId              int64
    TokenId             int64
    UsingGroup          string
    // ... 更多字段
}
```

#### 4.5 中继模式 (relay/constant/relay_mode.go)

```go
const (
    RelayModeUnknown = iota
    RelayModeChatCompletion
    RelayModeEmbedding
    RelayModeImage
    RelayModeAudio
    RelayModeRealtime
    RelayModeResponses
    RelayModeResponsesCompact
    RelayModeRerank
    // ...
)
```

### 5. 业务服务层 (service/)

**位置**: `/workspace/service/`

#### 5.1 渠道服务 (channel.go)

```go
func DisableChannel(channelError types.ChannelError, reason string)
func EnableChannel(channelId int, usingKey string, channelName string)
func ShouldDisableChannel(err *types.NewAPIError) bool
func ShouldEnableChannel(newAPIError *types.NewAPIError, status int) bool
```

**自动禁用逻辑**:
- 检查 `AutomaticDisableChannelEnabled` 配置
- 判断是否为渠道错误 (`IsChannelError`)
- 检查HTTP状态码 (`ShouldDisableByStatusCode`)
- 匹配禁用关键词

#### 5.2 配额服务 (quota.go)

```go
type QuotaInfo struct {
    InputDetails  TokenDetails
    OutputDetails TokenDetails
    ModelName     string
    UsePrice      bool
    ModelPrice    float64
    ModelRatio    float64
    GroupRatio    float64
}

func PreConsumeTokenQuota(relayInfo *relaycommon.RelayInfo, quota int) error
func PostConsumeQuota(relayInfo *relaycommon.RelayInfo, quota int, ...) error
func CalculateAudioQuota(info QuotaInfo) int
```

**配额计算公式**:
```go
quota = (input_text_tokens + 
         output_text_tokens * completion_ratio + 
         input_audio_tokens * audio_ratio + 
         output_audio_tokens * audio_ratio * audio_completion_ratio) 
        * group_ratio * model_ratio
```

#### 5.3 HTTP客户端 (http_client.go)

```go
var GlobalHttpClient *http.Client

func InitHttpClient() {
    GlobalHttpClient = &http.Client{
        Timeout: 30 * time.Minute,  // 流式请求支持30分钟
        Transport: &http.Transport{
            MaxIdleConns:        1024,
            MaxIdleConnsPerHost: 256,
        },
    }
}
```

### 6. DTO数据传输对象 (dto/)

**位置**: `/workspace/dto/`

#### 6.1 OpenAI请求 (openai_request.go)

```go
type GeneralOpenAIRequest struct {
    Model               string
    Messages            []Message
    Stream              *bool
    MaxTokens           *uint
    MaxCompletionTokens *uint
    Temperature         *float64
    TopP                *float64
    Functions           json.RawMessage
    Tools               []ToolCallRequest
    // ... 更多字段
}

type Message struct {
    Role             string
    Content          any            // string 或 []MediaContent
    ReasoningContent *string        // 思考内容
    ToolCalls        json.RawMessage
}

type MediaContent struct {
    Type       string
    Text       string
    ImageUrl   any
    InputAudio any
    File       any
    VideoUrl   any
}
```

#### 6.2 Claude请求 (claude.go)

```go
type ClaudeRequest struct {
    Model       string
    Messages    []ClaudeMessage
    MaxTokens   int
    System      string
    Tools       []Tool
    // Claude特有字段
}
```

#### 6.3 Gemini请求 (gemini.go)

```go
type GeminiChatRequest struct {
    Contents        []GeminiContent
    SystemInstruction string
    GenerationConfig *GeminiGenerationConfig
    SafetySettings  []GeminiSafetySetting
}
```

### 7. 中间件 (middleware/)

**位置**: `/workspace/middleware/`

#### 7.1 认证中间件 (auth.go)

```go
func UserAuth() func(c *gin.Context)      // 普通用户认证
func AdminAuth() func(c *gin.Context)      // 管理员认证
func RootAuth() func(c *gin.Context)       // 超级管理员认证
func TokenAuth() func(c *gin.Context)      // API Token认证
func TokenOrUserAuth() func(c *gin.Context) // Token或用户认证
func TokenAuthReadOnly() func(c *gin.Context) // 只读Token认证
```

**Token认证流程**:
1. 从Header获取 `Authorization: Bearer sk-xxx`
2. 解析key (支持 `sk-` 前缀)
3. 验证Token有效性 (`model.ValidateUserToken`)
4. 检查用户状态和IP限制
5. 设置上下文变量

#### 7.2 其他中间件

| 中间件 | 文件 | 功能 |
|--------|------|------|
| RateLimit | rate-limit.go | 请求速率限制 |
| Cors | cors.go | 跨域资源共享 |
| Logger | logger.go | 请求日志 |
| Recover | recover.go | 异常恢复 |
| Cache | cache.go | 响应缓存 |
| Stats | stats.go | 流量统计 |
| Performance | performance.go | 性能监控 |

### 8. 配置管理 (setting/)

**位置**: `/workspace/setting/`

#### 8.1 比率配置 (ratio_setting/)

```go
func GetGroupRatio(group string) float64
func GetModelRatio(modelName string) (ratio float64, hit bool, byGroup bool)
func GetCompletionRatio(modelName string) float64
func GetAudioRatio(modelName string) float64
func GetAudioCompletionRatio(modelName string) float64
```

#### 8.2 渠道亲和性 (channel_affinity.go)

智能选择最优渠道:
```go
type ChannelAffinityRule struct {
    Priority    int
    Conditions   []AffinityCondition
    TargetChannel string
}

type AffinityCondition struct {
    ModelPattern  string
    UserGroup     string
    TimeRange     string
}
```

#### 8.3 运营设置 (operation_setting/)

```go
// 自动禁用配置
var AutomaticDisableKeywords []string  // 禁用关键词
var StatusCodeRanges []StatusCodeRange  // 禁用状态码范围

type StatusCodeRange struct {
    Start int
    End   int
    Disable bool
}
```

### 9. 计费表达式引擎 (pkg/billingexpr/)

**位置**: `/workspace/pkg/billingexpr/`

支持复杂的阶梯计费表达式:
```go
// 表达式示例
"if P < 1000 { C * 1.0 } else if P < 5000 { C * 0.8 } else { C * 0.6 }"
```

**编译和执行**:
```go
func Compile(source string) (*Program, error)
func Run(program *Program, env *Environment) (any, error)
func Settle(params TokenParams, expr string) (*TieredResult, error)
```

### 10. 国际化 (i18n/)

**位置**: `/workspace/i18n/`

**支持语言**:
- `en` - English
- `zh-CN` - 简体中文
- `zh-TW` - 繁體中文

**消息Key示例**:
```go
const (
    MsgAuthNotLoggedIn = "auth_not_logged_in"
    MsgTokenInvalid = "token_invalid"
    MsgDatabaseError = "database_error"
)
```

---

## API接口文档

### 用户认证

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/user/register` | 用户注册 |
| POST | `/api/user/login` | 用户登录 |
| GET | `/api/user/logout` | 登出 |
| POST | `/api/user/login/2fa` | 2FA登录 |

### 渠道管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/channel/` | 获取所有渠道 |
| POST | `/api/channel/` | 创建渠道 |
| PUT | `/api/channel/` | 更新渠道 |
| DELETE | `/api/channel/:id` | 删除渠道 |
| GET | `/api/channel/test/:id` | 测试渠道 |
| POST | `/api/channel/fetch_models/:id` | 获取上游模型 |

### Token管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/token/` | 获取所有Token |
| POST | `/api/token/` | 创建Token |
| PUT | `/api/token/` | 更新Token |
| DELETE | `/api/token/:id` | 删除Token |

### 中继API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/v1/chat/completions` | 聊天完成 |
| POST | `/v1/embeddings` | 向量嵌入 |
| POST | `/v1/images/generations` | 图像生成 |
| POST | `/v1/audio/transcriptions` | 语音转文字 |
| POST | `/v1/audio/speech` | 文字转语音 |
| POST | `/v1/responses` | OpenAI Responses |
| POST | `/v1/realtime` | 实时对话 |
| POST | `/v1/rerank` | Rerank |

---

## 关键流程

### 1. 请求中继流程

```
客户端请求
    ↓
[TokenAuth中间件] - 验证API Key
    ↓
[Relay路由] - 解析请求路径
    ↓
[获取渠道] - 从缓存/数据库获取可用渠道
    ↓
[选择适配器] - 根据渠道类型选择适配器
    ↓
[转换请求] - 转换为上游API格式
    ↓
[发送请求] - 调用上游API
    ↓
[处理响应] - 转换响应格式
    ↓
[计费] - 计算并扣除配额
    ↓
返回响应
```

### 2. 渠道选择流程

```go
func SelectChannel(modelName, group string) (*Channel, error) {
    // 1. 检查缓存
    cached := channelCache.GetByModel(modelName, group)
    if cached != nil {
        return cached, nil
    }
    
    // 2. 按优先级排序
    channels := sortByPriority(channels)
    
    // 3. 检查模型能力
    for _, ch := range channels {
        if ch.SupportsModel(modelName) {
            // 4. 检查渠道余额
            if ch.Balance > 0 {
                return ch, nil
            }
        }
    }
    
    // 5. 跨分组重试
    if crossGroupRetry {
        return selectFromOtherGroups(modelName)
    }
    
    return nil, ErrNoChannelAvailable
}
```

### 3. 配额扣费流程

```go
func ConsumeQuota(relayInfo *RelayInfo, usage *Usage) {
    // 1. 预扣费 (可选)
    if preConsumeEnabled {
        quota := estimateQuota(relayInfo, usage)
        PreConsumeTokenQuota(relayInfo, quota)
    }
    
    // 2. 计算实际配额
    actualQuota := CalculateQuota(relayInfo, usage)
    
    // 3. 阶梯计费
    if tieredBillingEnabled {
        tieredQuota, _ := TryTieredSettle(relayInfo, params)
        actualQuota = tieredQuota
    }
    
    // 4. 扣除配额
    PostConsumeQuota(relayInfo, actualQuota, preConsumed, true)
    
    // 5. 记录日志
    RecordConsumeLog(relayInfo, actualQuota)
}
```

---

## 依赖关系

### Go模块依赖

```go
require (
    github.com/gin-gonic/gin v1.9.1          // HTTP框架
    gorm.io/gorm v1.25.2                     // ORM
    gorm.io/driver/mysql v1.4.3              // MySQL驱动
    gorm.io/driver/postgres v1.5.2           // PostgreSQL驱动
    github.com/glebarez/sqlite v1.9.0        // SQLite驱动
    github.com/redis/go-redis/v8 v8.11.5     // Redis客户端
    github.com/golang-jwt/jwt/v5 v5.3.0     // JWT认证
    github.com/go-webauthn/webauthn v0.14.0  // Passkey支持
    github.com/samber/lo v1.52.0             // 工具库
    github.com/tidwall/gjson v1.18.0         // JSON处理
    github.com/expr-lang/expr v1.17.8        // 表达式引擎
)
```

### 前端依赖

```json
{
    "dependencies": {
        "react": "^18.x",
        "react-router-dom": "^6.x",
        "@tanstack/react-table": "^8.x",
        "zustand": "^4.x",
        "i18next": "^23.x",
        "dayjs": "^1.x"
    }
}
```

---

## 环境变量

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `SQL_DSN` | 数据库连接字符串 | SQLite |
| `REDIS_CONN_STRING` | Redis连接字符串 | - |
| `SESSION_SECRET` | Session密钥 | - |
| `CRYPTO_SECRET` | 加密密钥 | - |
| `PORT` | HTTP端口 | 3000 |
| `GIN_MODE` | Gin运行模式 | release |
| `DEBUG` | 调试模式 | false |
| `STREAMING_TIMEOUT` | 流式超时(秒) | 300 |
| `LOG_SQL_DSN` | 日志数据库 | 主数据库 |
| `UMAMI_WEBSITE_ID` | Umami分析ID | - |
| `GOOGLE_ANALYTICS_ID` | GA4 ID | - |

---

## 运行方式

### 开发模式

```bash
# 编译前端
cd web/default && npm install && npm run build

# 编译后端
go build -o new-api .

# 运行
./new-api
```

### Docker部署

```bash
# 使用SQLite
docker run --name new-api -d \
  -p 3000:3000 \
  -v ./data:/data \
  calciumion/new-api:latest

# 使用MySQL
docker run --name new-api -d \
  -p 3000:3000 \
  -e SQL_DSN="root:123456@tcp(host.docker.internal:3306)/newapi" \
  -v ./data:/data \
  calciumion/new-api:latest
```

### Docker Compose

```yaml
version: '3.8'
services:
  new-api:
    image: calciumion/new-api:latest
    ports:
      - "3000:3000"
    environment:
      - SQL_DSN=root:password@tcp(mysql:3306)/newapi
      - REDIS_CONN_STRING=redis:6379
      - SESSION_SECRET=your-secret-key
    volumes:
      - ./data:/data
```

---

## 错误处理

### 错误类型 (types/error.go)

```go
type NewAPIError struct {
    Error *ErrorDetail
}

type ErrorDetail struct {
    Message string
    Type    string
    Code    string
}

const (
    ErrorTypeChannelError = "channel_error"       // 渠道错误
    ErrorTypeTokenError = "token_error"           // Token错误
    ErrorTypeQuotaError = "quota_error"           // 配额错误
    ErrorTypeAuthError = "auth_error"             // 认证错误
    ErrorTypeRateLimitError = "rate_limit_error"  // 限流错误
)
```

### 常见错误码

| 错误码 | 说明 | 处理方式 |
|--------|------|----------|
| 401 | 认证失败 | 检查API Key |
| 403 | 权限不足 | 检查Token权限 |
| 429 | 请求过于频繁 | 限流或等待 |
| 500 | 服务器内部错误 | 检查日志 |
| 503 | 服务不可用 | 检查渠道状态 |

---

## 性能优化

### 1. 渠道缓存
- 使用内存缓存渠道配置
- 支持热更新 (`SyncChannelCache`)
- 缓存命中率监控

### 2. 数据库优化
- 使用连接池 (MaxOpenConns: 1000)
- 预编译SQL语句 (PrepareStmt: true)
- 异步写入日志

### 3. HTTP客户端
- 连接池复用
- 超时配置 (30分钟支持流式)
- 自动重试

### 4. 性能指标 (pkg/perf_metrics)
```go
type RelaySample struct {
    ChannelId   int64
    ModelName   string
    Duration    time.Duration
    TokenCount  int64
    IsStream    bool
    Success     bool
}
```

---

## 安全措施

### 1. 认证
- Session认证
- JWT Token认证
- API Key认证
- 2FA (TOTP)
- Passkey (WebAuthn)

### 2. 权限控制
- 用户角色 (普通用户/管理员/超级管理员)
- 分组隔离
- IP白名单

### 3. 输入验证
- 请求体大小限制 (`MAX_REQUEST_BODY_MB`)
- SSRF防护
- SQL注入防护 (GORM)

### 4. 安全Headers
- `Auth-Version` - 防止版本冲突
- `X-Request-Id` - 请求追踪

---

## 监控和日志

### 系统监控
```go
common.StartSystemMonitor()  // CPU、内存、协程数监控
```

### 日志级别
```go
const (
    LogLevelDebug = iota
    LogLevelInfo
    LogLevelWarn
    LogLevelError
)
```

### Pyroscope集成
```go
common.StartPyroScope()  // 性能分析
```

---

## 扩展开发

### 添加新渠道

1. 在 `relay/channel/` 创建新目录
2. 实现 `Adaptor` 接口
3. 在 `constant/channel.go` 添加渠道类型
4. 在 `router/relay-router.go` 注册路由
5. 添加模型列表和常量

示例:
```go
// relay/channel/myprovider/adaptor.go
type Adaptor struct{}

func (a *Adaptor) Init(info *relaycommon.RelayInfo) { ... }
func (a *Adaptor) GetRequestURL(info *relaycommon.RelayInfo) (string, error) {
    return fmt.Sprintf("%s/v1/chat/completions", info.ChannelBaseUrl), nil
}
// ... 实现其他方法
```

### 添加新中间件

```go
func MyMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 前置处理
        c.Set("my_key", "my_value")
        
        c.Next()
        
        // 后置处理
        myValue := c.GetString("my_key")
    }
}
```

---

## 测试

### 运行测试
```bash
# 运行所有测试
go test ./...

# 运行特定包测试
go test ./model/...

# 运行带覆盖率的测试
go test -cover ./...
```

### 模拟测试
```go
// 模拟中继请求
func TestRelayChatCompletion(t *testing.T) {
    // 1. 创建测试上下文
    // 2. 构造测试请求
    // 3. 调用中继处理器
    // 4. 验证响应
}
```

---

## 文档更新日志

| 版本 | 日期 | 更新内容 |
|------|------|----------|
| 1.0 | 2026-05-12 | 初始文档 |

---

*本文档由代码分析自动生成，如有问题请联系开发者*
