# BuildCraft — 低代码搭建平台

> 类 Retool/Appsmith，拖拽组件生成页面，支持自定义插件扩展

---

## 核心功能

- 可视化拖拽：组件拖入画布，自由布局
- JSON Schema 渲染：画布内容序列化为 JSON，运行时渲染引擎解析执行
- 表达式引擎：组件属性支持 JS 表达式，实时求值
- 自定义插件：第三方开发者上传组件包，动态加载
- 多端预览：同一 Schema 在 Web/H5/小程序预览
- 自动化测试：拖拽操作 E2E 测试、组件视觉回归

---

## 挑战落地

### 07 低代码 — 引擎内核

- **可视化拖拽**：拖拽引擎（自研或基于 dnd-kit），支持自由布局/流式布局切换
- **JSON Schema 渲染**：定义 Schema 规范，Renderer 递归解析渲染组件树
- **沙箱隔离**：表达式引擎在沙箱内执行，禁止访问 DOM/网络（eval in sandbox）
- **代码生成**：JSON Schema → React 代码导出，可直接部署

**练习要点**：
- Schema 设计：组件类型 / 属性 / 事件 / 样式 / 数据绑定
- Renderer 实现：递归渲染 + 懒加载
- 沙箱实现：Proxy 拦截 + whiteList 或 iframe sandbox
- 撤销/重做：Command Pattern + History Stack

### 16 构建工程化 — 动态打包

- **Vite 插件开发**：自定义 Vite 插件处理 Schema → 代码转换
- **Tree-shaking**：按 Schema 中使用的组件按需打包，未使用组件不进入 Bundle
- **Bundleless**：开发模式下组件按需加载（ESM），不打包全量
- **自定义插件打包**：用户上传的组件包独立编译为 UMD/ESM，运行时动态 import

**练习要点**：
- Vite 插件 API（resolveId / load / transform）
- Rollup Tree-shaking 原理（ESM 静态分析）
- 动态 import() + importmap
- 组件包的版本管理与依赖去重

### 18 微前端 — 插件架构

- **插件独立部署**：每个自定义插件是一个微应用，独立开发、独立部署
- **Module Federation**：主应用与插件共享 React/Zustand 等依赖，避免重复加载
- **样式隔离**：Shadow DOM 或 CSS Scope 隔离插件样式
- **应用间通信**：Event Bus + Shared State 实现主应用与插件通信

**练习要点**：
- Module Federation 配置（exposes / remotes / shared）
- Shadow DOM 样式穿透问题
- 通信方案对比（CustomEvent / Shared Worker / BroadcastChannel）
- 插件生命周期管理（加载 → 挂载 → 卸载）

### 11 跨端 — 一套 Schema 多端渲染

- **统一引擎**：自研轻量虚拟 DOM（不依赖 React），核心 < 5KB
- **多端渲染器**：Web Renderer（DOM）、H5 Renderer（DOM + 触摸适配）、小程序 Renderer（WXML）
- **条件渲染**：Schema 支持平台条件表达式，不同平台渲染不同组件
- **Flutter Web 实验**：可选 Flutter Web 渲染器，探索 Flutter 渲染管线

**练习要点**：
- 虚拟 DOM 实现（createElement / diff / patch）
- 渲染器抽象层设计（Renderer Interface）
- 小程序适配层（WXML 生成 + wx API 代理）
- 跨端样式差异处理（rpx / vw / rem）

### 23 测试工程化 — 自动化保障

- **E2E 测试**：Playwright 模拟拖拽操作，验证画布渲染结果
- **组件视觉回归**：截图对比（pixelmatch），组件变更后自动检测 UI 偏差
- **Schema 测试**：模糊测试（Fuzz Testing）随机生成 Schema，验证 Renderer 不崩溃
- **覆盖率门禁**：CI 流程中核心模块行覆盖率 > 80%，否则阻断合并

**练习要点**：
- Playwright 拖拽操作模拟（page.mouse API）
- 视觉回归测试框架（Playwright screenshot + pixelmatch）
- Fuzz Testing 工具（fast-check）
- 覆盖率配置（Istanbul / c8 + GitHub Actions）

### 21 CSS 新能力 — 自适应与动效

- **Container Queries**：组件面板根据容器宽度自适应（窄容器 → 紧凑模式）
- **View Transitions**：编辑模式 ↔ 预览模式切换时的平滑过渡
- **Cascade Layers**：@layer 管理样式优先级（base < components < plugins < overrides）

**练习要点**：
- `@container` 规则与 `container-type: inline-size`
- `startViewTransition()` API 使用
- `@layer` 声明顺序与优先级规则

---

## 技术栈建议

```
前端框架：React 18+
拖拽引擎：dnd-kit / 自研
Schema 渲染：自研 Renderer
表达式引擎：jsep + 自定义求值器 / mathjs
沙箱：iframe sandbox / Proxy
微前端：Module Federation（Webpack 5 / Vite Federation）
构建工具：Vite + 自定义插件
测试：Playwright + Vitest
```
