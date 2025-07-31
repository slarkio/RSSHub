# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码库中工作提供指导。

## 项目概览

RSSHub 是基于 TypeScript 的 RSS 聚合服务，从超过 5000 个源聚合内容。使用 Hono web 框架构建，支持多种部署方式，包括 Docker、Vercel 和传统 Node.js。

## 核心命令

### 开发
- `npm run dev` - 启动开发服务器，支持热重载和调试（端口 1200）
- `npm run dev:cache` - 启动开发服务器，启用生产环境缓存

### 构建和生产
- `npm run build` - 构建路由并编译 TypeScript，用于生产环境
- `npm start` - 启动生产服务器（需要先构建）

### 测试
- `npm test` - 运行完整测试套件，包含覆盖率和格式检查
- `npm run vitest` - 运行测试，不含覆盖率
- `npm run vitest:watch` - 在监听模式下运行测试
- `npm run vitest:fullroutes` - 运行全面的路由测试

### 代码质量
- `npm run lint` - 检查 TypeScript 和 YAML 文件
- `npm run format` - 格式化并修复所有 TypeScript、JavaScript 和 JSON 文件
- `npm run format:check` - 检查格式，不进行修复

### 工具
- `npm run build:docs` - 生成文档
- `npm run profiling` - 使用 Node.js 性能分析器运行

## 架构

### 核心应用结构
- **`lib/app.ts`** - 主应用入口，包含请求重写器初始化
- **`lib/config.ts`** - 综合配置管理，支持 100+ 服务集成
- **`lib/routes/`** - 按服务/网站组织的路由实现（5000+ 路由）
- **`lib/middleware/`** - HTTP 中间件，包含缓存、日志、认证、模板
- **`lib/utils/`** - HTTP 请求、解析、缓存、Puppeteer 等共享工具
- **`lib/views/`** - 使用 JSX 的 RSS/Atom/JSON 订阅模板

### 路由架构
路由按服务名称在 `lib/routes/{service}/` 中组织：
- `namespace.ts` - 路由元数据和配置
- `{route}.ts` - 单个路由实现
- `utils.ts` - 服务特定的工具（如需要）

### 关键技术
- **Hono** - 支持 OpenAPI 的 Web 框架
- **Puppeteer** - 用于动态内容的浏览器自动化
- **Cheerio** - 服务端 HTML 解析
- **Redis/Memory** - 可配置的缓存层
- **OpenTelemetry** - 可观测性和指标
- **Vitest** - 带覆盖率的测试框架

### 配置系统
环境变量控制：
- 服务凭证（cookies、API 密钥、令牌）
- 缓存策略（内存/Redis）
- 代理设置和故障转移
- 功能标志和限流
- 调试和遥测

## 开发指南

### 添加新路由
1. 在 `lib/routes/{service}/` 下创建目录
2. 实现包含服务元数据的 `namespace.ts`
3. 在单独文件中创建路由处理器
4. 添加全面的测试
5. 更新文档

### 测试路由
- 每个路由都应有对应的测试文件
- 使用 `vitest:fullroutes` 进行集成测试
- 适当地模拟外部 API 调用
- 测试错误处理和边界情况

### 配置访问
- 从 `@/config` 导入 `config` 来处理所有配置需求
- 使用服务特定的配置部分（如 `config.bilibili`）
- 优雅地处理缺失的凭证

### 错误处理
- 使用 `lib/errors/` 中的结构化错误类型
- 实现正确的 HTTP 状态码
- 适当记录错误以便调试

### 性能考虑
- 利用内置缓存机制
- 为外部 API 使用连接池
- 在必要时实现限流
- 优化 Puppeteer 使用，使用共享实例

## 常见模式

### 路由处理器结构
```typescript
export const route: Route = {
    path: '/path/:param',
    categories: ['category'],
    handler: async (ctx) => {
        // 实现逻辑
        return {
            title: 'Feed Title',
            items: items.map(item => ({
                title: item.title,
                link: item.link,
                description: item.description,
                pubDate: item.date,
            })),
        };
    },
};
```

### 认证模式
- 存储在配置中的基于 Cookie 的认证
- 通过请求头的 API 密钥认证
- OAuth 令牌刷新机制
- 用于 IP 限制的代理轮换

### 数据获取
- 使用 `ofetch` 进行带自动重试的 HTTP 请求
- 为 JavaScript heavy 的网站实现 Puppeteer
- 使用内置缓存工具缓存昂贵的操作
- 使用指数退避处理限流