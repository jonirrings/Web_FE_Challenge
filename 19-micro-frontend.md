# 十九、微前端架构深入

微前端将微服务的思想带入前端：一个大应用被拆分为多个独立开发、独立部署的子应用，由一个主框架统一调度。核心挑战不是能跑起来，而是**在隔离和共享之间找到最佳平衡点**。

---

## 核心挑战

### 1. 样式隔离

多个子应用运行在同一页面，CSS 全局作用域相互污染是头号问题。

- **方案对比**：
  | 方案 | 隔离强度 | 成本 | 适用场景 |
  |------|----------|------|----------|
  | CSS Module | 中 | 低 | 构建时方案，类名 hash |
  | Scoped CSS (Vue) | 中 | 低 | Vue 生态 |
  | Shadow DOM | 强 | 中 | 浏览器原生隔离 |
  | CSS-in-JS | 中 | 低 | React 生态 |
  | BEM 命名空间 | 弱 | 低 | 手动约定，易被破坏 |
  | @layer | 中 | 低 | CSS 原生优先级控制 |

- **Shadow DOM 的穿墙问题**：
  - 弹窗/Drawer/Dropdown 组件通常 `appendChild` 到 `document.body`（为了突破父容器的 overflow: hidden）→ 这些元素在 Shadow DOM 之外 → Shadow DOM 的样式无法覆盖它们 → 样式隔离被打破。
  - 解决方案：
    - 自定义 Portal 目标：弹窗渲染到 Shadow Root 内的指定容器。
    - 样式投射（`::part()`）：允许外部定义某些组件的内部样式。
    - 双层样式：Shadow DOM 内定义组件基础样式，主应用 CSS 定义主题变量。

- **CSS Module + Scoped 的局限性**：
  - CSS Module 通过哈希类名隔离，但第三方组件库（Ant Design、Element Plus）的全局样式不在哈希范围内。
  - 需要为每个子应用的第三方组件库包裹专属的样式前缀。

### 2. 应用间通信

多个独立子应用之间需要通信，但不应该强耦合。

- **事件总线**：
  - 主应用创建一个共享的 EventEmitter → 子应用通过它发布/订阅事件。
  - 优势：松耦合，子应用不需要知道对方的存在。
  - 劣势：事件名冲突、事件泛滥后难以追踪（"谁在什么时候发了这个事件？"）。

- **消息队列模式**：
  - 比事件总线更有结构：定义消息类型 + 载荷格式（如 `{ type: 'ORDER_CREATED', payload: {...} }`）。
  - 每个子应用声明自己关心的消息类型，建立消息订阅表。

- **共享状态**：
  - 主应用维护一个全局状态存储（如简化的 Store）→ 子应用可以读取/修改。
  - 类似 Redux 的单一数据源，但放在主应用层。
  - 风险：子应用 A 修改了状态 → 子应用 B 出现 bug → 排查困难。

- **通信原则**：
  - 优先通过消息传递而非共享状态。
  - 每个消息带 sender 标识，方便追踪。
  - 避免循环事件（A 发消息 → B 响应后也发消息 → A 再次响应 → ...）。

### 3. 依赖共享与加载策略

多个子应用可能使用相同的基础库（React/Vue/Ant Design/axios），重复加载浪费带宽。

- **公共依赖管理**：
  - **方案 A：External 共享** — 主应用提供 React/Vue/antd 作为 external（如 `window.React`），子应用不打包这些库。
    - 优势：只加载一次，全局共享。
    - 劣势：版本强绑定——所有子应用必须使用同一版本。React 18 的子应用无法与 React 17 的共享库一起工作。
  - **方案 B：各自打包** — 每个子应用独立打包所有依赖。
    - 优势：版本自由。
    - 劣势：用户可能要下载 N 个 React 副本。
  - **折中方案**：核心框架（React/Vue）共享，UI 库各自打包。

- **Module Federation（Webpack 5 / Rspack）**：
  - 运行时跨应用共享模块：主应用暴露 React → 子应用声明 `shared: ['react']` → 运行时从主应用获取 React。
  - 版本协商：Singleton（全局唯一实例）或 StrictVersion（兼容版本范围）。

- **缓存策略**：
  - 文件名哈希：`react.abc123.js`，内容不变则哈希不变 → 长期缓存。
  - 主应用统一管理缓存：公共依赖的缓存更新策略（如主应用升级 → 失效所有子应用用户缓存）。

### 4. 预加载与性能优化

微前端架构引入了额外的加载开销——主应用 + 子应用的加载。

- **预加载子应用资源**：
  - 用户当前在子应用 A → 在空闲时预加载子应用 B、C 的 JS/CSS → 切换到子应用 B 时可以瞬间渲染。
  - `requestIdleCallback` 中调用 `import()` 预加载。
  - `<link rel="prefetch">` 预加载子应用的入口 chunk。

- **按需加载**：
  - 首屏只加载主应用 + 当前子应用，其他子应用延迟加载。
  - 子应用内部再按路由代码分割（lazy loading routes）。

- **子应用生命周期管理**：
  - Mount：加载 HTML + JS + CSS → 渲染到容器。
  - Update：主应用传递新的 props/状态。
  - Unmount：卸载子应用 → 清理事件监听、定时器、全局样式 → 回收内存。

---

## 深入学习路线

```
入门（1-2月）
├── qiankun 微前端框架：主应用 + 子应用配置
├── single-spa 原理：子应用注册、挂载、卸载
├── 理解样式隔离的基本方案
└── 最小微前端 Demo：主应用 + 两个子应用

进阶（3-4月）
├── 样式隔离深度：Shadow DOM vs CSS Module vs @layer
├── 应用间通信设计：事件总线 + 消息队列
├── 共享依赖管理：external + Module Federation
├── 子应用预加载 + 缓存策略
└── 实战：微前端改造——单体 SPA 拆分为微前端

深入（6月+）
├── Module Federation：运行时模块共享完整方案
├── 无容器微前端：ESM import maps + 原生浏览器方案
├── 微前端性能优化：加载时序、公共依赖去重
└── 运行时沙箱：Proxy 沙箱原理 + 隔离边界设计
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [qiankun](https://qiankun.umijs.org/) | 阿里微前端框架，基于 single-spa |
| [single-spa](https://single-spa.js.org/) | 微前端框架标准 |
| [Module Federation](https://webpack.js.org/concepts/module-federation/) | Webpack 5 运行时模块共享 |
| [wujie](https://wujie-micro.github.io/) | 无界微前端框架，基于 iframe |
| [Micro-app](https://zeroing.jd.com/micro-app/) | 京东微前端框架 |
| [Garfish](https://bytedance.github.io/garfish/) | 字节跳动微前端框架 |
