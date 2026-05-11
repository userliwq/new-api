# New API 项目代码文档

## 项目概述

New API 是一个基于 Go 语言构建的下一代 LLM 网关和 AI 资产管理平台。项目地址：https://github.com/QuantumNous/new-api

该项目聚合了 40+ 个上游 AI 提供商（OpenAI、Claude、Gemini、Azure、AWS Bedrock 等），提供统一的 API 接口，集成了用户管理、计费、速率限制和管理员仪表板等功能。

## 1. 项目整体架构

### 1.1 技术栈

| 层级 | 技术选型 |
|------|----------|
| 后端框架 | Go 1.25+，Gin Web 框架，GORM v2 ORM |
| 前端 | React 19，TypeScript，Rsbuild，Base UI，Tailwind CSS |
| 数据库 | SQLite，MySQL ≥ 5.7.8，PostgreSQL ≥ 9.6 |
| 缓存 | Redis (go-redis) + 内存缓存 |
| 认证 | JWT，WebAuthn/Passkeys，OAuth |
| 前端包管理 | Bun |

### 1.2 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Router Layer (router/)                  │
│         API路由 / Dashboard路由 / Relay路由 / Web路由         │
├─────────────────────────────────────────────────────────────┤
│                   Controller Layer (controller/)             │
│              请求处理、业务流程编排、参数验证                   │
├─────────────────────────────────────────────────────────────┤
│                     Service Layer (service/)                  │
│                核心业务逻辑、第三方集成、业务规则              │
├─────────────────────────────────────────────────────────────┤
│                      Model Layer (model/)                     │
│              数据模型定义、数据库访问、GORM 操作               │
├─────────────────────────────────────────────────────────────┤
│                   Relay Layer (relay/)                       │
│         AI API 中继与适配，支持 40+ 提供商                    │
├─────────────────────────────────────────────────────────────┤
│                      Common Layer (common/)                  │
│              通用工具、JSON 处理、加密、Redis、限流            │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 目录结构

```
new-api/
├── main.go                 # 应用入口
├── common/                 # 通用工具模块
│   ├── json.go            # JSON 编解码封装
│   ├── crypto.go          # 加密工具
│   ├── redis.go           # Redis 客户端
│   ├── rate-limit.go      # 限流器
│   └── utils.go           # 通用工具函数
├── constant/              # 常量定义
│   ├── api_type.go        # API 类型常量
│   ├── channel.go         # 渠道类型常量
│   └── endpoint_type.go   # 端点类型常量
├── controller/            # 控制器层
│   ├── channel.go         # 渠道管理
│   ├── user.go            # 用户管理
│   ├── token.go           # Token 管理
│   ├── billing.go         # 计费管理
│   └── relay.go           # 中继请求处理
├── service/               # 服务层
│   ├── channel.go         # 渠道服务
│   ├── quota.go           # 配额服务
│   ├── billing.go         # 计费服务
│   └── task.go            # 任务服务
├── model/                 # 数据模型层
│   ├── user.go            # 用户模型
│   ├── channel.go         # 渠道模型
│   ├── token.go           # Token 模型
│   └── main.go            # 数据库初始化
├── relay/                 # AI 中继层
│   ├── adaptor.go         # 适配器接口
│   ├── channel/           # 各提供商适配器
│   │   ├── openai/        # OpenAI 适配器
│   │   ├── claude/        # Claude 适配器
│   │   ├── gemini/        # Gemini 适配器
│   │   └── ...            # 其他 30+ 适配器
│   └── common/            # 中继公共逻辑
├── middleware/            # 中间件
│   ├── auth.go            # 认证中间件
│   ├── rate-limit.go      # 限流中间件
│   ├── cors.go            # 跨域中间件
│   └── logger.go          # 日志中间件
├── setting/               # 配置管理
│   ├── ratio_setting/     # 比率配置
│   ├── model_setting/     # 模型配置
│   └── operation_setting/ # 运营配置
├── dto/                   # 数据传输对象
├── router/                # 路由定义
├── oauth/                 # OAuth 认证
├── pkg/                   # 内部包
│   ├── billingexpr/       # 计费表达式引擎
│   └── cachex/           # 缓存封装
├── i18n/                  # 国际化
└── web/                   # 前端
    ├── default/          # 新版前端 (React 19)
    └── classic/          # 经典前端 (React 18)
```

## 2. 主要模块职责

### 2.1 路由层 (router/)

| 文件 | 职责 |
|------|------|
| main.go | 主路由设置入口 |
| api-router.go | API 路由定义 |
| relay-router.go | 中继请求路由 |
| dashboard.go | 管理后台路由 |
| web-router.go | 前端页面路由 |

**核心路由配置**：
```go
func SetRouter(router *gin.Engine, assets ThemeAssets) {
    SetApiRouter(router)      // API 接口路由
    SetDashboardRouter(router) // 管理后台路由
    SetRelayRouter(router)    // AI 中继路由
    SetVideoRouter(router)    // 视频生成路由
    SetWebRouter(router, assets) // 前端页面路由
}
```

### 2.2 控制器层 (controller/)

| 模块 | 职责 |
|------|------|
| channel.go | 渠道 CRUD 操作、测试、状态管理 |
| user.go | 用户注册、登录、权限管理 |
| token.go | API Key 生成、验证、配额管理 |
| billing.go | 充值、订阅、计费记录 |
| midjourney.go | Midjourney 任务管理 |
| relay.go | AI 请求中转处理 |

**主要控制器**：
- `channel.go`: 管理系统中的 AI 渠道，支持添加、编辑、测试渠道
- `user.go`: 用户认证、权限控制、个人信息管理
- `token.go`: API Token 管理、模型限制、配额查询
- `billing.go`: 支付集成、充值、配额消耗统计
- `ratio_sync.go`: 渠道比率同步

### 2.3 服务层 (service/)

| 模块 | 职责 |
|------|------|
| channel.go | 渠道业务逻辑、负载均衡选择 |
| quota.go | 配额管理、消费记录 |
| billing.go | 计费计算、套餐管理 |
| task.go | 异步任务管理 |
| text_quota.go | 文本配额处理 |
| subscription_reset_task.go | 订阅重置定时任务 |

**关键服务**：
- `channel_select.go`: 智能选择最佳渠道
- `pre_consume_quota.go`: 消费前配额预扣
- `tiered_settle.go`: 分层结算
- `token_counter.go`: Token 计数与估算

### 2.4 模型层 (model/)

| 模型 | 用途 |
|------|------|
| User | 用户信息、角色、状态 |
| Channel | AI 渠道配置、模型映射 |
| Token | API Token、分组限制 |
| Ability | 渠道能力映射 |
| Subscription | 订阅计划管理 |
| Log | 请求日志记录 |

**数据库初始化流程** (model/main.go)：
```go
func InitDB() error {
    db, err := chooseDB("SQL_DSN", false)
    // 支持 SQLite、MySQL、PostgreSQL
    DB = db
    return migrateDB()
}
```

### 2.5 中继层 (relay/)

中继层是核心模块，负责将客户端请求转发到上游 AI 提供商，并进行格式转换。

**适配器架构**：
```
                    ┌─────────────────┐
                    │  Relay Adaptor  │
                    │  (relay_adaptor.go) │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  OpenAI       │    │   Claude      │    │   Gemini      │
│  Adaptor      │    │   Adaptor     │    │   Adaptor     │
└───────────────┘    └───────────────┘    └───────────────┘
        │                    │                    │
        ▼                    ▼                    ▼
  OpenAI API            Claude API            Gemini API
```

**支持的 AI 提供商** (40+)：

| 类型 | 提供商 |
|------|--------|
| 通用兼容 | OpenAI, Claude, Gemini, Azure, AWS Bedrock |
| 国内厂商 | 阿里云、百度文心、腾讯混元、讯飞星火、智谱 GLM |
| 开源部署 | Ollama, Xinference, Dify |
| 商业平台 | Cohere, Jina, Replicate, SiliconFlow |
| 图像生成 | Midjourney, Suno, Kling, Jimeng |
| 其他 | Perplexity, Mistral, Moonshot, DeepSeek |

**获取适配器**：
```go
func GetAdaptor(apiType int) channel.Adaptor {
    switch apiType {
    case constant.APITypeOpenAI:
        return &openai.Adaptor{}
    case constant.APITypeAnthropic:
        return &claude.Adaptor{}
    case constant.APITypeGemini:
        return &gemini.Adaptor{}
    // ... 更多提供商
    }
    return nil
}
```

### 2.6 中间件层 (middleware/)

| 中间件 | 功能 |
|--------|------|
| auth.go | JWT 认证、API Key 验证 |
| rate-limit.go | 模型级别限流 |
| cors.go | 跨域资源共享 |
| logger.go | 请求日志记录 |
| stats.go | 性能统计 |
| recover.go | 恐慌恢复 |

### 2.7 通用模块 (common/)

| 模块 | 功能 |
|------|------|
| json.go | JSON 编解码封装（必须使用此模块） |
| crypto.go | 加密解密工具 |
| redis.go | Redis 连接与操作 |
| rate-limit.go | 限流器实现 |
| utils.go | 通用工具函数 |
| disk_cache.go | 磁盘缓存 |
| email.go | 邮件发送 |

**JSON 使用规范**：
```go
// 必须使用 common/json.go 中的封装
data, err := common.Marshal(v)           // 编码
err = common.Unmarshal(data, &v)         // 解码
err = common.DecodeJson(reader, &v)      // 流式解码
```

### 2.8 配置模块 (setting/)

| 目录 | 职责 |
|------|------|
| ratio_setting/ | 渠道比率配置 |
| model_setting/ | 模型配置与映射 |
| operation_setting/ | 运营配置 |
| system_setting/ | 系统设置 |

## 3. 关键类与函数说明

### 3.1 应用入口 (main.go)

```go
func main() {
    // 1. 初始化资源
    err := InitResources()
    
    // 2. 初始化缓存
    if common.MemoryCacheEnabled {
        model.InitChannelCache()
        go model.SyncChannelCache(common.SyncFrequency)
    }
    
    // 3. 启动定时任务
    go model.SyncOptions(common.SyncFrequency)
    go model.UpdateQuotaData()
    
    // 4. 启动 Gin 服务器
    server := gin.New()
    // ... 中间件配置
    router.SetRouter(server, assets)
    server.Run(":" + port)
}
```

**初始化流程**：
1. 加载环境变量
2. 设置日志
3. 初始化数据库
4. 初始化缓存
5. 初始化 HTTP 客户端
6. 初始化 i18n
7. 注册路由

### 3.2 渠道管理 (controller/channel.go)

**主要函数**：

| 函数 | 功能 |
|------|------|
| `GetChannels()` | 获取渠道列表 |
| `CreateChannel()` | 创建新渠道 |
| `UpdateChannel()` | 更新渠道配置 |
| `DeleteChannel()` | 删除渠道 |
| `TestChannel()` | 测试渠道连通性 |
| `AutomaticallyTestChannels()` | 自动测试所有渠道 |

**渠道测试逻辑**：
```go
func (c *Channel) Test() error {
    // 1. 构建测试请求
    req := c.BuildTestRequest()
    // 2. 发送请求到上游
    resp, err := c.SendRequest(req)
    // 3. 验证响应
    return c.ValidateResponse(resp)
}
```

### 3.3 中继适配器 (relay/channel/adapter.go)

**适配器接口定义**：

```go
type Adaptor interface {
    // 请求转换
    ConvertRequest() error
    
    // 发送请求到上游
    SendRequest() error
    
    // 转换响应
    ConvertResponse() error
    
    // 获取 API 类型
    GetAPIType() int
    
    // 获取模型列表
    GetModels() ([]string, error)
}
```

**流式响应处理**：
```go
type StreamAdaptor interface {
    ConvertStreamRequest() error
    HandleStreamEvent(event *Event) (string, error)
}
```

### 3.4 计费表达式 (pkg/billingexpr/)

**表达式引擎**：

| 文件 | 功能 |
|------|------|
| compile.go | 表达式编译 |
| run.go | 表达式执行 |
| types.go | 类型定义 |

**表达式语法示例**：
```
// 输入 Token 计费
p * 0.03

// 输出 Token 计费
c * 0.06

// 缓存命中折扣
p * 0.03 + c * 0.01
```

### 3.5 用户模型 (model/user.go)

**用户结构**：
```go
type User struct {
    ID          uint      `gorm:"primaryKey"`
    Username    string    `gorm:"uniqueIndex;size:64"`
    Password    string    `gorm:"size:128"`
    DisplayName string    `gorm:"size:128"`
    Role        int       `gorm:"default:1"` // 角色：0-root, 1-admin, 2-user
    Status      int       `gorm:"default:1"` // 状态：0-禁用, 1-启用
    Quota       float64   `gorm:"default:0"` // 配额
    Group       string    `gorm:"size:64"` // 分组
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

### 3.6 Token 模型 (model/token.go)

**Token 结构**：
```go
type Token struct {
    ID           uint      `gorm:"primaryKey"`
    UserId       uint      `gorm:"index"`
    Key          string    `gorm:"uniqueIndex;size:64"` // API Key
    Name         string    `gorm:"size:128"`
    Group        string    `gorm:"size:64"`
    ModelLimits  string    `gorm:"type:text"` // 模型限制
    Status       int       `gorm:"default:1"`
    UsedQuota    float64   `gorm:"default:0"` // 已使用配额
    CreatedAt    time.Time
    ExpiresAt    *time.Time
}
```

### 3.7 渠道模型 (model/channel.go)

**渠道结构**：
```go
type Channel struct {
    ID            uint      `gorm:"primaryKey"`
    Name          string    `gorm:"size:128"`
    Type          int       `gorm:"index"` // 渠道类型
    Key           string    `gorm:"size:512"` // API Key
    BaseURL       string    `gorm:"size:512"` // 自定义 endpoint
    Models         string    `gorm:"type:text"` // 模型列表
    ModelMapping  string    `gorm:"type:text"` // 模型映射
    Status        int       `gorm:"default:1"`
    Priority      int       `gorm:"default:1"` // 优先级
    Weight        int       `gorm:"default:1"` // 权重
    CreatedAt     time.Time
    UpdatedAt     time.Time
}
```

### 3.8 配额服务 (service/quota.go)

**配额消费流程**：
```go
func PreConsumeQuota(tokenId uint, model string, promptTokens, completionTokens int) error {
    // 1. 验证 Token 有效性
    token := GetToken(tokenId)
    if !token.IsValid() {
        return ErrInvalidToken
    }
    
    // 2. 计算配额消耗
    cost := CalculateCost(model, promptTokens, completionTokens)
    
    // 3. 预扣配额
    return DeductQuota(token.UserId, cost)
}
```

## 4. 依赖关系

### 4.1 Go 核心依赖

| 依赖 | 版本 | 用途 |
|------|------|------|
| gin-gonic/gin | v1.9.1 | HTTP Web 框架 |
| gorm.io/gorm | v1.25.2 | ORM |
| gorm.io/driver/mysql | v1.4.3 | MySQL 驱动 |
| gorm.io/driver/postgres | v1.5.2 | PostgreSQL 驱动 |
| glebarez/sqlite | v1.9.0 | SQLite 驱动 |
| go-redis/redis | v8.11.5 | Redis 客户端 |
| golang-jwt/jwt | v5.3.0 | JWT 认证 |
| go-webauthn/webauthn | v0.14.0 | Passkey 支持 |

### 4.2 第三方服务集成

| 服务 | 集成方式 |
|------|----------|
| Stripe | 支付订阅 |
| EPay | 支付通道 |
| Waffo | 支付服务 |
| Creem | 支付服务 |
| Umami | 网站分析 |
| Google Analytics | 流量分析 |
| Pyroscope | 性能监控 |

### 4.3 依赖图

```
main.go
├── common/          # 通用工具
│   ├── json.go     # JSON 编解码
│   ├── redis.go    # Redis 客户端
│   └── crypto.go   # 加密工具
├── model/           # 数据模型
│   ├── main.go     # 数据库初始化
│   ├── user.go     # 用户模型
│   ├── channel.go  # 渠道模型
│   └── token.go    # Token 模型
├── controller/      # 控制器
│   ├── channel.go  # 渠道管理
│   ├── user.go     # 用户管理
│   └── relay.go    # 中继处理
├── service/         # 业务服务
│   ├── channel.go  # 渠道服务
│   └── quota.go    # 配额服务
├── relay/           # 中继层
│   ├── adaptor.go  # 适配器接口
│   └── channel/    # 各提供商适配器
├── middleware/      # 中间件
│   ├── auth.go     # 认证
│   └── rate-limit.go # 限流
└── router/         # 路由
```

## 5. 项目运行方式

### 5.1 环境要求

| 组件 | 要求 |
|------|------|
| Go | 1.25+ |
| Node.js | 18+ |
| Bun | 最新版 |
| Docker | 20.10+ |
| 数据库 | SQLite/MySQL/PostgreSQL |

### 5.2 本地开发

**后端开发**：
```bash
# 1. 克隆项目
git clone https://github.com/QuantumNous/new-api.git
cd new-api

# 2. 安装依赖
go mod download

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 文件

# 4. 运行
go run main.go
```

**前端开发**：
```bash
# 进入前端目录
cd web/default

# 安装依赖
bun install

# 开发模式
bun run dev

# 构建
bun run build
```

### 5.3 Docker 部署

**使用 Docker Compose**：
```yaml
version: '3.8'
services:
  new-api:
    image: calciumion/new-api:latest
    ports:
      - "3000:3000"
    environment:
      - SQL_DSN=root:password@tcp(db:3306)/newapi
      - SESSION_SECRET=your-secret
    volumes:
      - ./data:/data
    depends_on:
      - db
      - redis

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=newapi

  redis:
    image: redis:7-alpine
```

**启动**：
```bash
docker-compose up -d
```

### 5.4 环境变量配置

**必填配置**：
| 变量 | 说明 | 示例 |
|------|------|------|
| `SQL_DSN` | 数据库连接字符串 | `root:password@tcp(localhost:3306)/newapi` |
| `SESSION_SECRET` | 会话密钥（多机部署必填） | `random-secret-string` |

**可选配置**：
| 变量 | 说明 | 默认值 |
|------|------|--------|
| `REDIS_CONN_STRING` | Redis 连接 | - |
| `CRYPTO_SECRET` | 加密密钥 | - |
| `PORT` | 服务端口 | 3000 |
| `GIN_MODE` | 运行模式 | release |
| `MEMORY_CACHE_ENABLED` | 启用内存缓存 | false |
| `SYNC_FREQUENCY` | 同步频率（秒） | 60 |

**第三方服务**：
| 变量 | 说明 |
|------|------|
| `STRIPE_API_KEY` | Stripe API Key |
| `STRIPE_WEBHOOK_SECRET` | Stripe Webhook 密钥 |
| `UMAMI_WEBSITE_ID` | Umami 网站 ID |
| `GOOGLE_ANALYTICS_ID` | GA4 ID |

### 5.5 数据库迁移

数据库迁移在应用启动时自动执行，支持的数据库：

**SQLite（默认）**：
```bash
# 数据存储在 /data 目录
docker run -v ./data:/data calciumion/new-api:latest
```

**MySQL**：
```bash
docker run -e SQL_DSN="root:password@tcp(localhost:3306)/newapi" \
           calciumion/new-api:latest
```

**PostgreSQL**：
```bash
docker run -e SQL_DSN="postgres://user:password@localhost/newapi" \
           calciumion/new-api:latest
```

### 5.6 验证运行

启动成功后访问：
- 管理后台：`http://localhost:3000/`
- 默认账号：`root`
- 默认密码：`123456`

### 5.7 性能优化配置

```bash
# 连接池配置
SQL_MAX_IDLE_CONNS=100
SQL_MAX_OPEN_CONNS=1000
SQL_MAX_LIFETIME=60

# 缓存配置
MEMORY_CACHE_ENABLED=true
SYNC_FREQUENCY=30

# 性能监控
ENABLE_PPROF=true
PYROSCOPE_URL=https://pyroscope.example.com
```

## 6. 高级功能

### 6.1 智能路由

项目支持多种渠道选择策略：

1. **权重随机**：按渠道权重分配请求
2. **优先级优先**：优先使用高优先级渠道
3. **自动重试**：失败时自动切换渠道
4. **模型限制**：Token 级别的模型访问控制

### 6.2 计费系统

支持多种计费模式：
- 按 Token 计费（输入/输出分开）
- 缓存计费
- 分层套餐
- 表达式计费

### 6.3 多租户

- 用户分组
- 分组配额隔离
- 模型访问控制
- 渠道分组绑定

### 6.4 安全特性

- API Key 管理
- JWT 会话认证
- WebAuthn/Passkey 支持
- OAuth 登录（GitHub, Discord, LinuxDo）
- 请求限流
- SSRF 防护

## 7. 扩展开发

### 7.1 添加新渠道

1. 在 `relay/channel/` 下创建新目录
2. 实现 `Adaptor` 接口
3. 在 `constant/api_type.go` 添加类型常量
4. 在 `relay/relay_adaptor.go` 注册适配器

### 7.2 添加新支付渠道

1. 在 `service/` 添加支付服务
2. 实现支付回调处理
3. 在 `controller/` 添加相关路由

### 7.3 自定义中间件

```go
func CustomMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 前置处理
        c.Set("custom_key", "value")
        
        c.Next()
        
        // 后置处理
        log.Println(c.Writer.Status())
    }
}
```

## 8. 最佳实践

### 8.1 数据库兼容性

- 使用 GORM 抽象接口
- 避免数据库特有语法
- 使用 `common.UsingSQLite/MySQL/PostgreSQL` 进行分支

### 8.2 JSON 处理

**必须使用** `common/json.go` 封装：
```go
// 正确
data, _ := common.Marshal(v)

// 错误
data, _ := json.Marshal(v)
```

### 8.3 错误处理

```go
// 使用 common 包记录错误
common.SysError("error message")
common.FatalLog("fatal error") // 会终止程序

// 返回结构化错误
c.JSON(400, gin.H{"error": gin.H{"message": "error"}})
```

### 8.4 日志规范

- 系统日志：`common.SysLog()`
- 错误日志：`common.SysError()`
- 致命错误：`common.FatalLog()`
- 调试日志：`common.DebugLog()` (仅在 debug 模式)

## 9. 文档与资源

- 官方文档：https://docs.newapi.pro
- GitHub：https://github.com/QuantumNous/new-api
- 问题反馈：https://github.com/QuantumNous/new-api/issues
- 官方Discord：https://discord.gg/xxxxx

---

*本文档由代码分析自动生成，如有疑问请参考官方文档或提交 Issue。*
