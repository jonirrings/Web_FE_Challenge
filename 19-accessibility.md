# 十九、Web 无障碍 Accessibility A11y

Web 无障碍（A11y）不是"给盲人用的"，而是让所有用户——包括视觉/听觉/运动/认知障碍用户——都能平等访问 Web 内容。在欧盟和美国（Section 508/ADA），无障碍合规是法定义务。技术上，A11y 涉及语义 HTML、ARIA 标注、焦点管理、辅助技术兼容等多个维度。

---

## 核心挑战

### 1. WCAG 标准合规

WCAG（Web Content Accessibility Guidelines）是国际通用的无障碍标准。

- **等级划分**：
  - **A**：最低要求（如"所有非文本内容有替代文本"）。不满足 → 完全不可访问。
  - **AA**：行业标准（如"文本对比度至少 4.5:1"）。大多数法律合规要求此等级。
  - **AAA**：最高要求（如"文本对比度至少 7:1"）。难以全面实现。

- **WCAG 2.1 / 2.2 关键原则**（POUR）：
  - **Perceivable（可感知）**：内容必须能被用户感知——图片有 alt、视频有字幕、颜色不是唯一传达信息的手段。
  - **Operable（可操作）**：界面必须可通过多种方式操作——键盘可导航、有足够时间阅读、无闪烁导致癫痫的内容。
  - **Understandable（可理解）**：内容和操作必须可理解——语言标识、可预测的交互行为、输入辅助。
  - **Robust（鲁棒）**：内容必须能被各种用户代理（浏览器、屏幕阅读器）正确解析——语义 HTML、ARIA。

- **常见合规问题**：
  - 图片无 `alt` 属性。
  - 色彩对比度不足（正常文本 < 4.5:1，大文本 < 3:1）。
  - 仅靠颜色传达信息（如"红色表单项为错误"缺错误图标）。
  - 表单无关联 label（`<label for="id">` 或包裹 `<input>`）。
  - 页面无标题（`<title>`）或无语义标题层级（`h1` → `h2` → `h3`）。

### 2. 屏幕阅读器适配

屏幕阅读器将网页内容转换为语音或盲文输出。

- **ARIA 标注体系**：
  - **role**：定义元素的语义角色（如 `role="button"` 告知阅读器这是一个按钮）。
  - **state**：定义元素的当前状态（如 `aria-expanded="false"` 表示折叠面板）。
  - **property**：定义元素的属性（如 `aria-label="关闭"` 给无文本的关闭按钮提供可读名称）。
  - **黄金法则**：优先使用原生语义 HTML（`<button>`、`<nav>`、`<main>`），ARIA 是补充而非替代。

- **ARIA Live Region 实时播报**：
  - `aria-live="polite"`：区域内容变化时，屏幕阅读器在当前任务完成后播报。
  - `aria-live="assertive"`：立即播报，打断当前任务（用于紧急通知）。
  - 应用场景：表单验证错误信息、搜索建议更新、聊天新消息、进度条变化。

- **不同阅读器兼容**：
  | 屏幕阅读器 | 平台 | 浏览器搭配 |
  |------------|------|------------|
  | NVDA | Windows | Chrome / Firefox |
  | JAWS | Windows | Chrome / Edge |
  | VoiceOver | macOS / iOS | Safari |
  | TalkBack | Android | Chrome |

  ARIA 在不同阅读器 + 浏览器组合下表现不一致（如某些 `aria` 属性在 VoiceOver 上有效、在 NVDA 上无效）。需要真机测试。

### 3. 键盘导航与焦点管理

键盘是运动障碍用户和高级用户（不用鼠标）的主要操作方式。

- **完整键盘操作流**：
  - Tab 键：在可聚焦元素间移动（链接、按钮、输入框、选择框、可聚焦的自定义组件）。
  - Enter/Space：激活按钮或选择项。
  - Arrow Keys：在复合组件内移动（如菜单项、Tab 面板、单选组）。
  - Escape：关闭弹窗/下拉菜单/对话框。

- **焦点陷阱（Focus Trap）**：
  - 模态框打开后，焦点应被"困"在模态框内（Tab 在最后一个元素 → 回到第一个元素，而非跑到背景页面）。
  - 模态框关闭后，焦点应返回触发它的元素（如"打开"按钮）。
  - 实现：在模态框第一个和最后一个可聚焦元素上监听 Tab，手动 `element.focus()` 转移焦点。

- **复合组件键盘交互模式**（WAI-ARIA Authoring Practices）：
  - **Combobox（搜索建议）**：输入框 + 下拉建议列表，Arrow Up/Down 在建议项间移动。
  - **Listbox（多选列表）**：Arrow Up/Down 移动，Space 选中/取消，Ctrl+A 全选。
  - **Tree（树形导航）**：Arrow Up/Down 在同级移动，Arrow Right/Left 展开/折叠。
  - **Roving Tabindex**：Tab 进入组件组 → 焦点落在"激活子项" → Arrow Keys 在子项间移动（子项 `tabindex="-1"` + `focus()` 配合）。

### 4. 视觉无障碍

- **色彩对比度**：
  - WCAG AA 标准：正常文本对比度 ≥ 4.5:1，大文本（>18px 粗体或 >24px）≥ 3:1。
  - 检测工具：Chrome DevTools 的 CSS Overview / Lighthouse / axe DevTools / Stark 插件。

- **减少动效**：
  - `@media (prefers-reduced-motion: reduce)`：用户系统设置"减少动效"时，禁用或减弱 CSS 动画（`animation: none` 或更简单的过渡）。
  - 关键：不为追求视觉效果牺牲无障碍——如视差滚动、大型缩放动画可能引发眩晕。

- **暗色模式**：
  - `@media (prefers-color-scheme: dark)`：适配系统暗色模式偏好。
  - 不仅仅是颜色反转——暗色模式的对比度、饱和度需要专门设计。

- **文本缩放**：
  - 用户可能设置浏览器最小字体大小或缩放 200%。
  - 布局不能因文本放大而破坏：不固定容器高度、使用相对单位（rem/em）、弹性布局。

### 5. 自动化检测与测试

- **自动化工具**：
  - **axe-core**：Deque 开源的无障碍检测引擎，可集成到单元测试/CI。
  - **Lighthouse**：Chrome 内置审计工具，包含无障碍评分。
  - **eslint-plugin-jsx-a11y**：在代码编写阶段检测 JSX 的无障碍问题。

- **自动化 vs 手动测试**：
  - 自动化只能发现 **~30%** 的无障碍问题（如缺少 alt、对比度不足、ARIA 错误使用）。
  - 剩余 70% 需要手动判断：操作流是否合理？按钮的文本是否清楚地表达了功能？页面结构是否符合预期？
  - 屏幕阅读器实机测试非常必要——在至少一种阅读器（NVDA 或 VoiceOver）上完整走一遍核心流程。

- **CI/CD 集成**：
  - 每次 PR 触发 axe-core 检测 → 超过阈值（如新增 5 个 IA 问题）→ CI 失败。
  - 定期 Lighthouse 审计 → 无障碍分数回归告警。

---

## 深入学习路线

```
入门（1-2月）
├── WCAG 2.1 标准通读：理解 POUR 四原则
├── 语义 HTML：正确的标签（button vs div role=button）
├── ARIA 规范：常用 role/state/property
├── 键盘导航基础：Tab、Enter、Escape、Arrow Keys
└── 最小无障碍改造：为现有页面修复 10 个问题

进阶（3-4月）
├── 屏幕阅读器（NVDA/VoiceOver）操作练习
├── 复合组件 ARIA 模式：Combobox/Listbox/Tree/Tab Panel
├── 焦点管理进阶：Roving Tabindex、Focus Trap
├── 无障碍测试体系：axe-core + 手动测试
└── 实战：组件库无障碍改造

深入（6月+）
├── 无障碍自动检测引擎开发：自定义 axe 规则
├── Web 无障碍与原生（iOS/Android）对比
├── 多语言场景无障碍适配（RTL + ARIA）
└── 无障碍设计系统：从源头保证组件合规
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [WCAG 2.2](https://www.w3.org/TR/WCAG22/) | Web 无障碍标准 |
| [axe-core](https://github.com/dequelabs/axe-core) | 无障碍检测引擎 |
| [eslint-plugin-jsx-a11y](https://github.com/jsx-eslint/eslint-plugin-jsx-a11y) | JSX 无障碍 Lint |
| [WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/) | 复合组件无障碍实现指南 |
| [NVDA](https://www.nvaccess.org/) | Windows 免费屏幕阅读器 |
| [axe DevTools](https://www.deque.com/axe/devtools/) | 浏览器无障碍调试插件 |
| [Stark](https://www.getstark.co/) | 无障碍设计 + 开发工具 |
