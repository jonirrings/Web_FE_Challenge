# 十六、前端构建工程化

前端构建工程化的核心不是"会配 Webpack"，而是理解**编译器/打包器的工作原理**——AST 解析、依赖图构建、代码转换、产物优化。新一代构建工具（Vite/Rspack/Turbopack）正把前端构建推向毫秒级增量编译。

---

## 核心挑战

### 1. 构建工具原理对比

- **Webpack（传统打包器）**：
  - 架构：一切皆模块 → 构建依赖图 → 打包为 bundle。
  - 冷启动慢（大型项目 >30 秒）的原因：JS 实现的全量依赖分析 + Loader/Plugin 链。
  - 热更新（HMR）：文件变化 → 重新构建受影响的模块 → WebSocket 推送到浏览器。

- **Vite（新一代构建工具）**：
  - 开发模式：基于浏览器原生 ESM，不打包——`<script type="module" src="./App.tsx">`。
  - 按需编译：浏览器请求哪个文件 → 即时编译哪个文件（ESBuild 处理 TS/JSX）。
  - 生产模式：Rollup 打包。
  - 冷启动 < 1 秒（与项目大小无关，因为不打包）。

- **Rspack（Rust 的 Webpack）**：
  - Rust 实现的 Webpack 兼容打包器，API 兼容 Webpack 配置。
  - 编译速度比 Webpack 快 5-10 倍。
  - 适合已有 Webpack 项目无缝迁移。

- **Turbopack（Vercel 出品）**：
  - Next.js 13+ 的默认开发构建器，Rust 实现。
  - 函数级缓存：只重新编译变化的函数，而非整个文件。

### 2. Tree-shaking 深度优化

Tree-shaking 不只是"删掉未使用的导出"。

- **静态分析**：
  - 打包器通过 ES Module 的静态 `import`/`export` 构建依赖图。
  - CommonJS (`require`) 是动态的 → 无法 tree-shake。
  - 必须用 ESM（`import` / `export`）才能 tree-shake。

- **副作用标记**：
  - `package.json` 中的 `"sideEffects": false`：声明此包无副作用，打包器可以安全删除未被引用的导出。
  - 如果某文件有副作用（如 `import './polyfill.js'`），必须在 `sideEffects` 数组中声明。

- **Dead Code Elimination**：
  - Terser/SWC 压缩阶段：消除 `if (false)` 块、未被调用的函数、不可达代码。
  - 结合 `__DEV__` 等编译时常量 → 开发代码在生产环境自动去除。

- **按需加载与动态导入**：
  - `import('./heavy-module').then(...)`：代码分割点，生成独立 chunk。
  - React.lazy + Suspense / Vue defineAsyncComponent 实现组件级懒加载。

### 3. 自定义构建插件

理解 AST 转换是编写插件的前提。

- **Babel 插件**：
  - Babel 将 JS 解析为 AST（@babel/parser）→ 遍历修改（@babel/traverse）→ 重新生成代码（@babel/generator）。
  - Visitor 模式：访问特定节点类型（如 `CallExpression`）并修改。
  - Babel 插件的速度瓶颈：JS 实现的 AST 遍历。

- **SWC 插件**：
  - Rust 编写的编译器，速度比 Babel 快 20-70 倍。
  - 插件系统基于 `swc_core`（Rust crate），用 Rust 编写插件。

- **ESBuild 插件**：
  - ESBuild 本身不支持用户自定义插件（核心用 Go 编写，插件 API 是 JS）。
  - 插件通过 `onResolve` / `onLoad` 钩子修改模块解析和加载逻辑。

### 4. Bundleless 运行时优化

Bundleless 是"反打包"的思路——利用浏览器原生 ESM 能力。

- **ESM Import Maps**：
  - `<script type="importmap">`：定义裸标识符（如 `"react"`）→ CDN URL 的映射。
  - 配合 `<script type="module">`，浏览器直接加载未打包的源码。
  - 适用场景：开发环境、低复杂度应用、Deno 风格开发。

- **运行时模块加载优化**：
  - 预加载（`<link rel="modulepreload">`）：提前告知浏览器这个模块很快会需要，提前加载。
  - HTTP/2 Server Push：服务端在请求 HTML 时主动推送相关模块资源（现已被弃用，推荐用 103 Early Hints）。
  - Module Federation：Webpack 5 / Rspack 的微前端方案——运行时跨应用共享模块。

---

## 深入学习路线

```
入门（1-2月）
├── Vite 原理：ESM 开发模式 + Rollup 生产构建
├── Rollup 配置：input/output/plugins 体系
├── 理解打包器的基本流程：入口 → 依赖图 → chunks → 输出
└── 最小构建配置：TypeScript + JSX + CSS

进阶（3-4月）
├── ESBuild 原理与插件：onResolve/onLoad 钩子
├── Rspack 深度使用：Webpack 兼容迁移
├── Tree-shaking 原理：ESM 静态分析 + 副作用检测
├── 代码分割策略：路由级 + 组件级 + 公共依赖提取
└── 实战：自定义构建插件（自动导入、环境变量注入）

深入（6月+）
├── Vite/Rspack 源码阅读：Dev Server 实现、HMR 原理
├── 编译器原理：词法分析 → 语法分析 → AST → 代码生成
├── Module Federation：运行时共享模块的架构设计
└── 构建性能监控：构建耗时分析、产物大小追踪
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Vite](https://vitejs.dev/) | 新一代前端构建工具 |
| [Rspack](https://www.rspack.dev/) | Rust Webpack 兼容打包器 |
| [Turbopack](https://turbo.build/pack) | Vercel Rust 构建工具 |
| [ESBuild](https://esbuild.github.io/) | Go 编写的高性能打包器 |
| [Rollup](https://rollupjs.org/) | 专注于库打包 |
| [SWC](https://swc.rs/) | Rust 编写的 JS/TS 编译器 |
| [Babel](https://babeljs.io/) | JS 编译器 |
| [Terser](https://terser.org/) | JS 压缩混淆 |
