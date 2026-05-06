# 十一、跨端统一渲染内核

跨端开发的终极形态不是"一套代码跑多个平台"，而是**构建一个统一的渲染内核**，对各平台的差异进行抽象和抹平。这需要对渲染管线有从顶至底的理解——从虚拟 DOM 到平台原生控件的映射。

---

## 核心挑战

### 1. 鸿蒙 / 小程序 / APP / H5 多端统一引擎

不同平台的渲染模型差异巨大，需要系统性地抹平差异。

- **平台渲染模型对比**：
  | 平台 | 渲染机制 | 语言 | 限制 |
  |------|----------|------|------|
  | H5 | DOM / CSSOM → Layout → Paint | JS/TS | 浏览器性能上限 |
  | 微信小程序 | WebView 双线程（渲染层 + 逻辑层） | JS + WXML/WXSS | 无法直接操作 DOM |
  | React Native | JS Bridge → Native View | JS + Native 组件 | 需熟悉原生开发 |
  | Flutter | Skia 引擎自绘（不依赖原生控件） | Dart | 体积大 |
  | 鸿蒙 ArkUI | 方舟开发框架（声明式 UI） | ArkTS | 生态初建 |

- **渲染对齐**：
  - 同一套 DSL（如 JSX 描述 UI 树）→ 不同平台生成不同的渲染输出。
  - H5：JSX → DOM 操作。
  - 小程序：JSX → 小程序模板（WXML/WXSS）。
  - 鸿蒙：JSX → ArkUI 组件树。
  - 每个平台需要独立的渲染器（Renderer），但 UI 树描述层统一。

- **样式归一**：
  - CSS 属性在不同平台支持程度不一。需要 CSS 属性映射表：如 `display: flex` → 鸿蒙 `Flex({ direction: FlexDirection.Row })`。
  - 尺寸单位转换：rpx（小程序）/ lpx（鸿蒙）/ px（H5）/ dp（Android）/ pt（iOS）。

- **事件抹平**：
  - 点击事件：H5 `click`（有 300ms 延迟）/ 移动端 `touchend`（无延迟）/ 鸿蒙 `onClick`。
  - 手势：H5 有 `touchstart/touchmove/touchend`，小程序有 `bindtouchstart`，手势识别需统一抹平。

### 2. 自研跨端渲染框架

如果要脱离 Vue/React 生态自研一套跨端方案，需要从零搭建渲染内核。

- **虚拟 DOM 实现**：
  - 核心 API：`createElement(type, props, ...children)` → 返回 VNode 对象（{ type, props, children, key }）。
  - Diff 算法：同层比较（Tree Diff），通过 key 复用节点。复杂度 O(n)（n = 节点数）。
  - 属性 Diff：className 变化 → 更新 class；style 对象变化 → 逐个属性比较更新。

- **布局引擎**：
  - 实现 Flexbox 布局算法（CSS Flexbox 规范 → 布局计算）。
  - Yoga（Facebook 开源）：C 实现的 Flexbox 布局引擎，React Native 底层使用，可编译到 Wasm 供浏览器使用。
  - 自研布局需处理：盒模型计算（margin/border/padding/content）、弹性伸缩（flex-grow/shrink）、对齐（justify-content/align-items）。

- **样式计算**：
  - 样式表的解析：CSS 文本 → AST → 规则匹配。
  - 优先级计算：specificity（内联 > ID > Class > 标签）。
  - 样式继承：某些属性（font-size、color）自动继承父节点。

### 3. 原生混合渲染 Hybrid

Hybrid 模式是 WebView 和原生控件的混合使用。

- **WebView 性能优化**：
  - WebView 冷启动慢（首次加载 WebView 引擎 + 创建 JS 上下文）：可以通过 WebView 池预加载 + 预热来优化。
  - 离线包：将 H5 资源预下载到 App 本地 → 拦截 WebView 请求 → 从本地加载（省去网络请求时间）。
  - 渲染黑/白屏：WebView 渲染前先显示原生骨架屏/loading → 渲染完成后隐藏。

- **原生与前端通信桥**：
  - JSBridge：原生注入 JS 对象 → H5 调用 `window.bridge.invoke('method', params, callback)` → 原生响应。
  - 参数序列化：复杂对象必须 JSON.stringify（有性能开销和循环引用问题）。
  - 回调管理：每个 invoke 生成唯一 callbackId → 原生执行完后通过 callbackId 回传结果。

- **渲染互通**：
  - 原生地图控件在 WebView 列表中的渲染——地图需要在 WebView 上层用原生 View 覆盖。
  - 坐标同步：WebView 滚动 → 原生 View 同步移动，保持相对位置一致。

### 4. Flutter Web / Taro / UniApp 底层渲染原理

理解主流框架的渲染差异是选型前提。

- **Flutter Web**：
  - HTML 渲染模式：Flutter Widget → HTML DOM + Canvas。性能接近原生，但包体积大（>2MB）。
  - CanvasKit 渲染模式：用 Skia 的 Wasm 版本（CanvasKit）在 Canvas 上自绘所有内容。视觉效果一致但文本不可选中、无障碍支持差。

- **Taro（小程序跨端）**：
  - Taro 2.x：重编译时——JSX 编译为 WXML 模板。编译时处理，运行时轻量但动态能力弱。
  - Taro 3.x：重运行时——在逻辑层模拟 DOM/BOM API，渲染层用小程序的模板。类似 React Native 的设计。

- **UniApp**：
  - 编译 + 运行时混合。编译时把 Vue 模板转为各平台代码，运行时提供跨平台 API（uni.xxx）。
  - Weex 内置：可编译到 Weex（Vue Native 渲染）。

---

## 深入学习路线

```
入门（1-2月）
├── 主流跨端框架使用：Taro / UniApp / React Native
├── 小程序原理：双线程模型、WXML 编译
├── 理解虚拟 DOM 的基本实现
└── 最小跨端 Demo：H5 + 小程序

进阶（3-4月）
├── 自研虚拟 DOM：createElement + Diff + Patch
├── 布局引擎：Yoga Layout 实践
├── 样式系统：CSS-in-JS 实现、属性映射表
├── 统一事件系统：事件委托 + 手势识别
└── 性能优化：启动速度分析、包体积优化

深入（6月+）
├── Flutter Web 渲染原理：CanvasKit vs HTML 后端
├── 鸿蒙方舟开发框架：ArkUI 声明式范式
├── 原生渲染桥接优化：JSBridge 零拷贝方案
└── 跨端性能对比：各框架渲染性能基准测试
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Taro](https://taro.js.org/) | 京东开源跨端框架，支持 React/Vue 转小程序 |
| [UniApp](https://uniapp.dcloud.io/) | DCloud 跨端框架，支持 Vue 转各平台 |
| [React Native](https://reactnative.dev/) | Meta 跨端框架，原生渲染 |
| [Flutter](https://flutter.dev/) | Google 跨端方案，自绘引擎 |
| [Rax](https://github.com/alibaba/rax) | 阿里跨端框架，小程序 + Web |
| [Yoga](https://yogalayout.com/) | Facebook Flexbox 布局引擎（C 实现） |
| [Remax](https://github.com/remaxjs/remax) | 运行时小程序框架，React 编写 |
