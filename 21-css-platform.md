# 二十一、CSS 平台新能力体系

2023-2026 年是 CSS 能力爆发的几年。浏览器标准化了一系列此前需要 JS 库才能实现的功能——从响应式布局到动画编排，从组件级样式作用域到原生弹出层定位。这些新能力不仅减少了 JS 体积，更重要的是它们在**浏览器合成层运行**，性能远超 JS 实现。

---

## 核心挑战

### 1. Cascade Layers 级联层

`@layer` 是 CSS 优先级管理的革命：开发者可以显式定义样式层级，彻底告别选择器权重肉搏。

- **基本原理**：
  ```css
  @layer reset, base, components, utilities;
  /* utilities 永远覆盖 components，components 永远覆盖 base，与选择器具体权重无关 */
  ```
  `@layer` 内的选择器权重低于 `@layer` 外的选择器。层顺序决定最终覆盖关系。

- **应用场景**：
  - **设计系统分层**：reset → base → components → overrides → utilities。
  - **第三方样式隔离**：`@layer third-party { @import 'antd.css'; }` → 第三方样式在一个低优先级的层中，不会意外覆盖你的样式。
  - **替代 BEM/ITCSS**：不再需要靠 `.block__element--modifier` 的命名规范来管理优先级。

- **与现有架构整合**：
  - 与 CSS Module 共存：CSS Module 的哈希类名在 `@layer` 内部仍然有效。
  - 与 Tailwind CSS 整合：Tailwind 的 utilities 层天然适合放在最高优先级层。

### 2. Container Queries 容器查询

Container Queries 是继 Media Query 之后最重要的响应式设计变革——组件基于**自身容器**而非视口大小自适应。

- **语法**：
  ```css
  .card-container { container-type: inline-size; container-name: card; }
  @container card (min-width: 400px) {
    .card { display: grid; grid-template-columns: 1fr 1fr; }
  }
  ```
  当 `.card-container` 宽度 ≥ 400px（而非整个视口）时，`.card` 变为两栏布局。

- **组件级自适应**：
  - 同一个 `.card` 组件放在宽的 main 区域和窄的 sidebar 中，自动适应不同的内部布局。
  - 组件真正的**独立性和可复用性**：不需要知道自己在页面的哪个位置。

- **Container Query Units**：
  - `cqw/cqh`：容器宽/高的 1%。
  - `cqi/cqb`：容器 inline/block 尺寸的 1%（RTL 友好）。
  - 比 `vw/vh` 更精确控制组件内部元素的尺寸。

- **Container Queries vs Media Queries**：
  - Media Queries = 页面级自适应（整页布局切换）。
  - Container Queries = 组件级自适应（每个组件独立响应）。
  - 两者互补，不互相替代。

### 3. View Transitions API 视图过渡

浏览器原生支持跨页面/同页面的声明式过渡动画，无需 JS 动画库。

- **跨页面过渡（MPA）**：
  ```js
  document.startViewTransition(() => updateDOM());
  // 浏览器自动：截图旧状态 → 执行回调 → 截图新状态 → 过渡动画
  ```
  MPA（多页面应用）也能有 SPA 般的过渡效果。

- **同页面过渡（SPA）**：
  - 列表中添加/删除/移动项目、详情页切换、模态框打开/关闭。
  - 浏览器自动为移动的元素做 transform 动画（FLIP 动画的技术细节由浏览器处理）。

- **CSS 动画控制**：
  ```css
  ::view-transition-old(root) { animation: fadeOut 0.3s; }
  ::view-transition-new(root) { animation: fadeIn 0.3s; }
  ```
  完全用 CSS 控制过渡的动画类型、时长、缓动。

- **DOM 状态控制**：
  - `view-transition-name`：给特定元素命名 → 浏览器对该元素单独做过渡（而非整个页面）。
  - 过渡期间旧 DOM 仍存在（视觉快照），新 DOM 已渲染但隐藏。过渡完成后旧快照销毁、新 DOM 显示。

### 4. Scroll-driven Animations 滚动驱动动画

动画进度直接绑定滚动位置偏移，不再需要 JS 监听 scroll 事件。

- **Scroll Progress Timeline**：
  ```css
  .progress-bar {
    animation: fill 1s linear;
    animation-timeline: scroll(root);
  }
  @keyframes fill { from { width: 0%; } to { width: 100%; } }
  ```
  页面滚动进度直接映射为进度条的宽度动画——全程在合成层运行，不阻塞主线程。

- **View Progress Timeline**：
  ```css
  .fade-in {
    animation: fade 1s linear;
    animation-timeline: view();
    animation-range: entry 0% entry 100%;
  }
  ```
  元素进入/离开视口的过程驱动动画（替代 Intersection Observer + requestAnimationFrame）。

- **性能优势**：
  - 动画在合成器线程执行，独立于主线程 JS。
  - 即使用户快速滚动，动画也能保持流畅（不受 JS 阻塞影响）。

### 5. Anchor Positioning 锚点定位

CSS 原生锚点定位，替代 Popper.js / Floating UI 等定位库。

- **基本语法**：
  ```css
  .tooltip { position-anchor: --trigger; position-area: top; }
  .trigger { anchor-name: --trigger; }
  ```
  `.tooltip` 自动定位在 `.trigger` 的上方，不需要 JS 计算位置。

- **溢出自动翻转（@position-fallback）**：
  ```css
  @position-fallback --tooltip-fallback {
    @try { position-area: top; }
    @try { position-area: bottom; }
    @try { position-area: left; }
  }
  .tooltip { position-fallback: --tooltip-fallback; }
  ```
  顶部空间不够 → 自动尝试底部 → 左边，完全替代 Popper 的 `flip` 行为。

- **应用场景**：
  - Tooltip / Popover / 下拉菜单 / Select 弹出层 / 右键菜单。
  - 所有"附着在另一个元素旁边的浮层"都可以用 CSS 锚点定位替代 JS 库。

### 6. Scoped CSS 与 @scope

浏览器原生样式作用域，不依赖 Shadow DOM 或 CSS Module 的编译步骤。

- **@scope 语法**：
  ```css
  @scope (.component) {
    p { color: blue; } /* 只对 .component 内部的 p 生效 */
    .child { font-size: 1rem; }
  }
  @scope (.component) to (.inner) {
    /* 到 .inner 为止，不进入 .inner 内部 */
  }
  ```
  比后代选择器更精确控制样式传播边界。

- **与应用场景**：
  - 微前端场景：子应用 A 的样式不渗透到子应用 B。
  - 设计系统：组件样式精确隔离，不依赖 Shadow DOM 的隔离成本。
  - 样式渗透控制：`@scope` 的 `to` 语法可以定义"样式传播到这一层就打住"。

- **vs Shadow DOM**：
  - Shadow DOM 更强（完全隔离，JS 和 DOM 都隔离），但有穿墙问题和 SEO 影响。
  - @scope 只隔离样式，DOM 和 JS 仍然共享——更轻量。

---

## 深入学习路线

```
入门（1-2月）
├── MDN CSS 新特性文档通读
├── Container Queries 实践：改造现有响应式组件
├── @layer 级联层：重构现有 CSS 架构
└── 最小 Demo：用 Container Queries 实现自适应卡片

进阶（3-4月）
├── View Transitions API：跨页面 + 同页面过渡动画
├── Scroll-driven Animations：替代 JS 滚动方案
├── Anchor Positioning：替代 Popper.js / Floating UI
├── @scope 样式作用域实践
└── 实战：用纯 CSS 新能力实现组件库

深入（6月+）
├── CSS 新能力与设计系统架构重新设计
├── 浏览器合成层深度：CSS 动画 vs JS 动画性能
├── Polyfill 与渐进增强：不支持的环境如何回退
└── 参与 CSSWG 提案讨论：影响 CSS 标准演化
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [MDN CSS](https://developer.mozilla.org/en-US/docs/Web/CSS) | CSS 参考文档 |
| [Can I Use](https://caniuse.com/) | 浏览器特性支持查询 |
| [CSS Container Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_container_queries) | MDN 容器查询文档 |
| [View Transitions API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API) | MDN 视图过渡文档 |
| [Anchor Positioning](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_anchor_positioning) | MDN 锚点定位文档 |
| [web.dev 新 CSS](https://web.dev/tags/css/) | Google 新 CSS 特性教程 |
| [PostCSS](https://postcss.org/) | CSS 编译工具（可处理新特性的 polyfill） |
