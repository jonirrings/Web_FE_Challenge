# 十八、前端监控与可观测性

前端可观测性超越了"报错发个邮件"。它是一套完整的体系：**采集 → 上报 → 聚合 → 分析 → 告警 → 优化**。目标是回答三个问题：用户实际体验如何？哪里出了问题？根因是什么？

---

## 核心挑战

### 1. 前端指标体系 RUM

RUM（Real User Monitoring，真实用户监控）是前端监控的核心方法论。

- **Google Core Web Vitals（核心指标）**：
  - **LCP（Largest Contentful Paint）**：最大内容绘制时间，衡量加载速度。目标 < 2.5 秒。
    - LCP 元素通常是首屏的大图、大段文本或视频封面。
    - 优化：预加载关键图片、使用 CDN、优化服务端响应时间。
  - **INP（Interaction to Next Paint）**：交互到下次绘制（2024 年替代 FID），衡量响应速度。目标 < 200ms。
    - 覆盖整个页面生命周期的交互延迟，比 FID 更全面。
    - 优化：长任务拆分、减少主线程阻塞。
  - **CLS（Cumulative Layout Shift）**：累积布局偏移，衡量视觉稳定性。目标 < 0.1。
    - 图片无尺寸、动态注入内容、Web Font 切换都会引起 CLS。
    - 优化：为图片/视频/广告位预留尺寸（`width`/`height` 属性或 `aspect-ratio` CSS）。

- **其他关键指标**：
  - **TTFB（Time to First Byte）**：首字节时间，衡量网络延迟和服务器响应速度。
  - **FCP（First Contentful Paint）**：首次内容绘制，衡量用户看到内容的速度。
  - **TTI（Time to Interactive）**：可交互时间，页面在视觉上渲染完毕且可以响应用户输入。
  - **FPS（Frames Per Second）**：帧率，衡量动画和滚动的流畅度。

- **指标采集方式**：
  - `PerformanceObserver`：监听 LCP/INP/CLS/FCP 等指标（web-vitals 库封装）。
  - `Performance API`：`performance.timing`（已废弃，用 `PerformanceNavigationTiming` 替代）。
  - `Long Task API`：`PerformanceObserver` 监听 `longtask` 事件（> 50ms 的任务）。

### 2. 异常捕获链路

前端异常捕获不仅仅是 `window.onerror`。

- **错误类型与捕获方式**：
  | 错误类型 | 捕获方式 | 示例 |
  |----------|----------|------|
  | 同步 JS 错误 | `window.onerror` | `throw new Error()` |
  | Promise 未捕获 | `window.onunhandledrejection` | `Promise.reject()` 无 catch |
  | 资源加载错误 | `window.addEventListener('error', ..., true)`（捕获阶段）| `<img src=404>` |
  | 静态资源 404 | 同上 | JS/CSS 文件 404 |
  | 手动上报 | `try-catch` + 手动调用上报 | API 调用异常 |

- **堆栈还原（Source Map）**：
  - 生产环境代码被压缩混淆 → 错误堆栈不可读（`at a.r (bundle.js:1:2345)`）。
  - Source Map 将压缩后的行列号映射回源文件位置。
  - **安全考虑**：source map 不应该部署到生产环境的公开目录。错误上报时带上行列号 → 服务端用私有的 source map 文件还原堆栈。

- **采样策略**：
  - 不能 100% 上报（大数据量 → 成本高 + 影响分析效率）。
  - 采样维度：按用户（随机 10%）、按错误类型（严重错误 100%、非严重错误 10%）、按频率（同一错误在时间窗口内去重）。

- **异常聚类**：
  - 同样的错误（不同用户/浏览器）应聚合为同一个 Issue。
  - 聚类方法：错误堆栈指纹（取堆栈的前 N 行做哈希）、错误消息正则归一化（把变量值替换为占位符）。

### 3. 性能监控与告警

- **SLA/SLO 定义**：
  - SLA（Service Level Agreement）：对客户的承诺（如"99.9% 用户 LCP < 2.5s"）。
  - SLO（Service Level Objective）：内部目标（如"P95 LCP < 2s"）。
  - 定义指标 + 阈值 + 时间段（如"过去 5 分钟 P95 INP > 300ms"）。

- **告警规则**：
  - 阈值告警：指标超过阈值触发。
  - 同比/环比告警：相比昨天同一时间/上一时间段恶化 X%。
  - 告警抑制：低优先级告警在已有高优先级告警时静默。

- **性能监控平台**：
  - 自建：Grafana + Prometheus + 前端 SDK → 上报 → 时序数据库存储。
  - 商业：Sentry / Datadog RUM / New Relic Browser / 阿里 ARMS。

### 4. OpenTelemetry 前端集成

OpenTelemetry 是 CNCF 的可观测性标准，正在将链路追踪引入前端。

- **分布式 Tracing**：
  - 前端请求 → BFF → 后端服务 A → 数据库。一次用户操作的完整链路。
  - `trace-id`（全局唯一）+ `span-id`（每层操作）贯穿整个链路。
  - 前端生成 trace-id → 通过 HTTP Header（`traceparent`）传递给后端 → 前后端链路串联。

- **日志与事件聚合**：
  - 用户操作路径（Click → Page View → API Call → Error）。
  - Session Replay：录制用户操作过程（DOM 快照 + 事件回放），用于错误复现。
  - 注意隐私：自动屏蔽敏感输入框、手机号/身份证号脱敏。

---

## 深入学习路线

```
入门（1-2月）
├── Performance API：PerformanceObserver、PerformanceTimeline
├── web-vitals 库集成：LCP/INP/CLS/FCP 采集
├── 错误捕获：window.onerror + unhandledrejection
└── 最小监控 Demo：指标采集 + 上报

进阶（3-4月）
├── Core Web Vitals 优化：逐个指标分析与改进
├── Source Map 堆栈还原：错误定位到源码
├── 采样策略设计：按用户/按错误类型/按频率
├── 自定义性能打点：User Timing API（performance.mark/measure）
└── 实战：前端监控平台搭建（采集 → 展示 → 告警）

深入（6月+）
├── OpenTelemetry JS：前端分布式 Tracing 集成
├── Session Replay：DOM 录制 + 事件回放
├── 异常聚类算法：堆栈指纹 + 消息归一化
├── 实时日志流处理 + 根因分析
└── 监控告警体系设计：SLO 定义 + 告警规则 + on-call 流程
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [web-vitals](https://github.com/GoogleChrome/web-vitals) | Google Core Web Vitals 采集库 |
| [Sentry](https://sentry.io/) | 错误监控 + 性能监控平台 |
| [OpenTelemetry JS](https://opentelemetry.io/docs/instrumentation/js/) | 前端 Tracing 标准 |
| [Grafana](https://grafana.com/) | 可观测性可视化平台 |
| [rrweb](https://www.rrweb.io/) | Session Replay DOM 录制库 |
| [Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API) | MDN Performance API 文档 |
| [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) | 性能审计 CI 集成 |
