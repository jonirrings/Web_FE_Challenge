# 七、低代码/零代码 引擎内核开发

低代码/零代码平台的核心不是拖拽 UI，而是一套能将**可视化配置实时编译为可运行应用**的渲染引擎。真正的难点在于引擎层的抽象设计——既要有足够的灵活性（覆盖 80% 以上的业务场景），又不能让配置复杂度过高。

---

## 核心挑战

### 1. 可视化拖拽渲染引擎

画布（Canvas）是低代码的交互入口，但不是简单的 `position: absolute`。

- **自由画布与吸附对齐**：
  - 拖拽过程中实时计算与其他组件的对齐关系：边缘对齐、中心对齐、等间距分布。
  - 吸附算法：当组件 A 的边缘与组件 B 的边缘距离 < N px 时，自动吸附到对齐位置。
  - 参考线渲染：对齐时绘制临时辅助线（类似 Figma/Sketch 的对齐指示）。

- **图层管理**：
  - 类似 Photoshop 的图层面板：管理组件的 z-index、可见性、锁定。
  - 图层嵌套：容器组件（Flex/Grid/Stack）内嵌套子组件，递归渲染。

- **布局算法**：
  - 现代低代码引擎逐渐抛弃绝对定位，转向基于 Flex/Grid 的约束布局。
  - 拖拽时实时计算布局位置——组件拖入 Flex 容器时，其他兄弟组件自动"让位"。
  - 响应式布局：在不同断点下（手机/平板/桌面），组件的布局位置和尺寸自动适配。

- **碰撞检测**：
  - 拖拽时判断组件是否进入/离开某个容器（Drop Target）。
  - 算法：`element.getBoundingClientRect()` 实时比较，或使用空间索引（如 R-Tree）加速大画布下的检测。

### 2. JSON Schema 动态渲染

低代码的页面定义不是代码，是 JSON Schema，运行时被渲染引擎解析为真实 DOM。

- **Schema 设计**：
  - 页面 = JSON Tree：`{ type: 'Page', children: [{ type: 'Container', props: {...}, children: [...] }] }`。
  - 每个组件节点的属性包括：类型、props（样式/数据/事件）、children、版本号（组件版本升级时的兼容）。

- **表单引擎**：
  - 不是简单的"JSON → 表单控件"，而是完整的表单系统：
    - **联动（Reactions）**：A 字段值变化 → B 字段显隐/options 变化/C 校验规则变化。
    - **校验（Validation）**：必填、格式、自定义函数、异步校验（如用户名查重）。
    - **布局（Layout）**：分栏、分组、步骤条式表单（Wizard）。
  - 规则引擎通常采用**事件驱动 + 表达式系统**模式。

- **动态数据绑定**：
  - 组件属性可以绑定数据源（API 返回 / State 变量 / URL 参数）。
  - 模板语法如 `{{ state.userName }}` 或 `$context.orderId`。

### 3. 自定义组件沙箱隔离

允许用户自定义组件的低代码平台必须解决安全隔离问题。

- **样式隔离**：
  - CSS Module (webpack) / Scoped CSS (Vue) 是常见方案。
  - Shadow DOM：最强隔离，但第三方组件库（Ant Design、Element Plus）的弹窗、下拉菜单常 append 到 document.body，绕过了 Shadow DOM。
  - **穿墙问题**：`<Teleport>` / Portal 组件挂载到 Shadow DOM 外部，样式隔离被破坏。需要用 `@layer` 或严格的 CSS 命名空间。

- **JS 隔离**：
  - 自定义组件如果可以直接访问全局 window → 安全风险。
  - **Sandboxed Iframe**：最强隔离，但通信成本高、性能差、样式/主题传递困难。
  - **Proxy 沙箱**：用 `new Proxy(window, handler)` 模拟 window 对象，对不安全 API（eval、fetch 到外部域名、document.cookie）做拦截。微前端框架如 qiankun 广泛使用此方案。
  - **Web Worker 沙箱**：组件逻辑在 Worker 中运行，通过消息与主线程交互。天然隔离，但无法操作 DOM。

### 4. 表达式解析引擎

低代码的"逻辑"部分需要表达式引擎——类似 Excel 公式。

- **表达式类型**：
  - 取值：`{{ user.name }}`、`{{ state.count + 1 }}`。
  - 逻辑：`{{ visible ? '显示' : '隐藏' }}`。
  - 函数调用：`{{ formatDate(date, 'YYYY-MM-DD') }}`。

- **实现方式**：
  - **字符串替换**（最简单）：正则替换 `{{ xxx }}`，但对复杂表达式力不从心。
  - **new Function**：`new Function('context', 'return ' + expr)` 动态执行。注意：有 XSS 风险，必须对表达式做白名单校验。
  - **AST 解析**：完整解析表达式为 AST → 解释执行。安全、可控，但实现复杂。

- **建议**：不要自研完整表达式引擎。使用成熟的开源方案：
  - [formula.js](https://github.com/handsontable/formula.js)：类 Excel 公式引擎。
  - [hot-formula-parser](https://github.com/handsontable/hot-formula-parser)：Handsontable 出品的公式解析器。
  - [jexl](https://github.com/TomFrost/jexl)：JavaScript 表达式语言，安全且可扩展。

### 5. 源码导出与私有化部署

企业客户的核心诉求之一："在你平台上搭建后，把源码导出，部署到我们自己的服务器上。"

- **DSL 设计**：
  - 平台内部使用 JSON Schema（平台 DSL），导出时需要转换为目标框架代码。
  - DSL → React/Vue 代码生成（Code Generation）。

- **代码生成策略**：
  - **模板拼接**：为每个组件准备 React/Vue 代码模板，批量替换属性 → 拼接。简单但灵活度低。
  - **AST 生成**：直接构建目标语言的 AST → 输出代码。灵活度高，可以优化代码结构（如自动合并 import、提取公共变量）。
  - **编译时优化**：去除未使用组件、内联固定样式、压缩代码。

- **依赖分析**：
  - 生成的项目需要精确的依赖声明（package.json），不能多也不能少。
  - 分析哪些组件被使用 → 只引入必需的第三方库 → 避免不必要的大依赖。

---

## 深入学习路线

```
入门（1-2月）
├── React DnD / dnd-kit / pragmatic-drag-and-drop 使用
├── 核心拖拽算法：碰撞检测、Drag Overlay、Drop Animation
├── 理解 JSON Schema → 动态组件渲染模式
└── 实现：简单拖拽排序 + 画布

进阶（3-4月）
├── 虚拟画布：无限滚动 + 虚拟化渲染（只渲染可视区域的组件）
├── 组件树设计：嵌套 Schema + 递归渲染
├── 表达式解析：理解 AST 构建 + 词法分析 + 语法分析
├── 沙箱隔离方案选择与实现
└── 实战：可视化表单构建器（含字段联动和校验）

深入（6月+）
├── 运行时组件加载：动态 import + 远程组件
├── 代码生成管线：DSL → AST → 源码 → 打包
├── 编译时优化：静态分析、无用代码消除
└── 开源项目阅读：React Flow、Logic Flow、LowCodeEngine（阿里）
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [React Flow](https://reactflow.dev/) | 流程图/节点图编辑器 |
| [Logic Flow](https://site.logic-flow.cn/) | 滴滴开源流程图编辑框架 |
| [LowCodeEngine](https://github.com/alibaba/lowcode-engine) | 阿里低代码引擎，设计思路参考价值高 |
| [dnd-kit](https://dndkit.com/) | React 拖拽库，支持碰撞检测、排序、多容器 |
| [Pragmatic drag and drop](https://github.com/atlassian/pragmatic-drag-and-drop) | Atlassian 开源拖拽库，性能优秀 |
| [Formily](https://formilyjs.org/) | 阿里表单解决方案，Schema 驱动 |
| [jexl](https://github.com/TomFrost/jexl) | JavaScript 表达式语言 |
