# 二十四、Serverless 与边缘计算

前端开发正从"纯浏览器端"向"全栈化"演进。借助 Serverless 和边缘计算，前端可以直接编写服务端逻辑、操作数据库、实现边缘渲染——无需运维服务器，全球就近执行，冷启动毫秒级。

---

## 核心挑战

### 1. 边缘函数架构

边缘函数（Edge Functions）运行在 CDN 节点上，比传统服务器更接近用户。

- **边缘 vs 传统 Serverless**：
  | 维度 | 传统 Serverless（Lambda/FC） | 边缘函数（Workers/Edge） |
  |------|---------------------------|------------------------|
  | 运行位置 | 中心云区域 | 全球 CDN 节点 |
  | 冷启动 | 100ms - 数秒 | < 1ms |
  | 运行时长 | 最长 15 分钟 | 通常 < 50ms - 数秒 |
  | 适用场景 | 重计算、长任务 | 轻量逻辑、请求改写、渲染 |

- **主流平台**：
  - **Cloudflare Workers**：V8 Isolates 技术，零冷启动，全球 300+ 节点。
  - **Vercel Edge Functions**：基于 Cloudflare，与 Next.js 深度集成。
  - **Deno Deploy**：V8 + Rust，原生 TypeScript，边缘优先。
  - **AWS Lambda@Edge**：CloudFront 集成，但冷启动较高。

- **边缘函数限制**：
  - CPU 时间限制（如 Workers 免费版 10ms）。
  - 内存限制（通常 128MB - 1GB）。
  - 无文件系统、无原生模块（部分平台支持 Wasm）。

### 2. 边缘渲染（Edge SSR/ESR）

在边缘节点执行服务端渲染，比中心云更快。

- **边缘 SSR 模式**：
  - 用户请求 → 边缘节点 → 执行 React/Vue SSR → 返回 HTML。
  - 相比传统 SSR（请求 → 中心云 → SSR → 返回），减少了网络往返。

- **流式渲染（Streaming SSR）**：
  - 边缘节点流式输出 HTML：先返回骨架屏 → 再返回主要内容 → 最后返回次要内容。
  - 浏览器可以边接收边解析，首字节时间（TTFB）大幅降低。

- **边缘缓存策略**：
  - 渲染结果缓存到边缘：相同请求直接返回缓存，无需重复渲染。
  - 增量静态再生（ISR）：后台异步更新缓存，前端始终返回最新缓存。

- **平台方案**：
  - Next.js on Vercel：`export const runtime = 'edge'` 标记边缘运行。
  - Nuxt on Cloudflare：`nitro: { preset: 'cloudflare-pages' }`。
  - Remix on Deno Deploy：原生边缘支持。

### 3. 前端直连数据库

前端绕过 BFF 直接操作数据库，简化架构。

- **BaaS（Backend as a Service）**：
  - **Firebase**：Google 出品，Realtime Database + Firestore + Auth。
  - **Supabase**：开源 Firebase 替代，PostgreSQL + 实时订阅 + Auth。
  - **Appwrite**：开源 BaaS，功能全面，可自托管。

- **数据库即服务（DBaaS）直连**：
  - **PlanetScale**：MySQL 兼容，Serverless 定价，边缘友好。
  - **Neon**：PostgreSQL 分离存储与计算，按需扩展。
  - **Turso**：SQLite 边缘化，全球复制，超低延迟。

- **安全模型**：
  - **行级安全（RLS）**：数据库层面限制用户只能访问自己的数据。
  - **JWT 验证**：前端携带 JWT，边缘函数验证后操作数据库。
  - **直连风险**：前端代码暴露，必须通过权限控制防止越权。

- **ORM 与查询**：
  - **Prisma**：支持边缘运行时（Edge-compatible），类型安全。
  - **Drizzle**：轻量 ORM，边缘友好，SQL-like API。
  - **Kysely**：类型安全 SQL 构建器，无运行时开销。

### 4. 边缘状态与实时同步

在边缘维护状态，实现低延迟实时应用。

- **边缘 KV 存储**：
  - **Cloudflare KV**：全球分布式键值存储，最终一致性。
  - **Vercel Edge Config**：低延迟配置存储，用于特性开关。
  - **Upstash Redis**：Serverless Redis，全球复制，边缘可访问。

- **实时同步架构**：
  - **WebSocket at Edge**：Durable Objects（Cloudflare）实现有状态边缘。
  - **Pub/Sub**：Ably、Pusher 等实时服务与边缘函数集成。
  - **Server-Sent Events（SSE）**：边缘函数推送实时更新。

- **边缘缓存一致性**：
  - 边缘缓存 TTL 策略：平衡新鲜度与性能。
  - 缓存失效：主动清除（Purge API）vs 被动过期。
  - 原子更新：使用 KV 的版本控制实现缓存一致性。

### 5. 全栈框架一体化

现代框架将前端、API、数据库整合为统一开发体验。

- **Next.js 全栈**：
  - App Router：服务端组件默认，自动优化数据获取。
  - API Routes：`/app/api/*` 定义后端接口。
  - Server Actions：组件内直接调用服务端逻辑，无需显式 API。

- **Nuxt 全栈**：
  - Nitro 引擎：统一的开发/生产运行时，支持多平台部署。
  - Server API：`/server/api/*` 定义接口。
  - 自动导入：服务端代码也享受自动导入便利。

- **SvelteKit**：
  - 表单 Actions：原生表单提交 + 渐进增强。
  - 适配器架构：一套代码部署到 Node、Vercel、Cloudflare 等。

- **Remix**：
  - 数据加载模式：`loader`（GET）、`action`（POST/PUT/DELETE）。
  - 渐进增强：无 JS 也能工作，有 JS 体验更好。

### 6. 边缘安全与认证

边缘节点执行认证，保护后端资源。

- **边缘认证模式**：
  - JWT 验证：边缘节点验证 JWT，无效请求直接拦截。
  - Session 验证：边缘 KV 存储 session，快速校验。
  - OAuth 回调：边缘处理 OAuth 回调，设置 Cookie。

- **零信任架构**：
  - 不信任任何请求，每次都在边缘验证身份。
  - 最小权限原则：边缘函数只能访问必要资源。

- **DDoS 防护**：
  - 边缘天然具备 DDoS 清洗能力（Cloudflare 等）。
  - 速率限制：边缘层实现请求限流，保护源站。

### 7. 成本优化与可观测性

Serverless 按需付费，需要精细化成本管理。

- **成本模型**：
  - 按请求数计费：注意避免无限循环调用。
  - 按执行时间计费：优化代码减少 CPU 时间。
  - 按出流量计费：压缩响应、启用边缘缓存。

- **冷启动优化**：
  - 保持函数"温热"：定时 ping（非生产环境慎用）。
  - 减小 bundle 体积：Tree-shaking、按需引入。
  - 使用边缘函数：零冷启动，适合延迟敏感场景。

- **可观测性**：
  - 分布式追踪：OpenTelemetry 在边缘的支持。
  - 日志聚合：边缘日志实时回传中心。
  - 性能监控：边缘执行时间、缓存命中率。

---

## 深入学习路线

```
入门（1-2月）
├── Cloudflare Workers / Vercel Edge Functions 入门
├── 边缘 SSR：Next.js/Nuxt 边缘运行时配置
├── 基础 KV 存储：Cloudflare KV / Upstash Redis
├── 简单 API：边缘函数处理 CRUD
└── 实战：边缘部署一个带缓存的 API

进阶（3-4月）
├── 前端直连数据库：Supabase/PlanetScale + Prisma
├── 行级安全（RLS）配置与权限设计
├── 实时同步：SSE / WebSocket at Edge
├── 全栈框架深度：Next.js Server Actions / Nuxt Nitro
├── 边缘认证：JWT 验证、Session 管理
└── 实战：边缘部署实时协作应用

深入（6月+）
├── 边缘数据库：Turso、Neon 架构原理
├── Durable Objects：有状态边缘计算
├── 边缘缓存策略：一致性模型、失效机制
├── 成本优化：Bundle 体积、执行时间优化
├── 可观测性：边缘日志、分布式追踪
└── 开源贡献：Workers 生态、全栈框架
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Cloudflare Workers](https://workers.cloudflare.com/) | 边缘函数平台 |
| [Vercel Edge Functions](https://vercel.com/docs/functions/edge-functions) | Vercel 边缘函数 |
| [Deno Deploy](https://deno.com/deploy) | Deno 边缘运行时 |
| [Next.js](https://nextjs.org/) | React 全栈框架 |
| [Nuxt](https://nuxt.com/) | Vue 全栈框架 |
| [SvelteKit](https://kit.svelte.dev/) | Svelte 全栈框架 |
| [Remix](https://remix.run/) | 现代 Web 框架 |
| [Supabase](https://supabase.com/) | 开源 Firebase 替代 |
| [PlanetScale](https://planetscale.com/) | Serverless MySQL |
| [Turso](https://turso.tech/) | 边缘 SQLite |
| [Prisma](https://www.prisma.io/) | 类型安全 ORM |
| [Drizzle](https://orm.drizzle.team/) | 轻量 ORM |
| [Upstash](https://upstash.com/) | Serverless Redis |
| [Hono](https://hono.dev/) | 边缘优先 Web 框架 |
