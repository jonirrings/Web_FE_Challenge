# 二十四、前端测试工程化

前端测试不是"写几个 Jest 用例"那么简单。真正的测试工程化需要覆盖**单元测试、集成测试、E2E 端到端测试、可视化回归测试**四层金字塔，并与 CI/CD 流水线深度集成，形成质量门禁。

---

## 核心挑战

### 1. 测试策略金字塔

测试有不同的粒度和成本，需要合理分配投入。

- **测试金字塔**：
  ```
      /\
     /  \  E2E 测试（少而精，核心链路）
    /____\
   /      \  集成测试（组件组合、API 集成）
  /________\
 /          \  单元测试（多而快，纯函数、工具类）
/____________\
  ```

- **单元测试（Unit Test）**：
  - 测试对象：纯函数、工具类、状态管理逻辑。
  - 特点：速度快（毫秒级）、确定性高、易于定位问题。
  - 覆盖率目标：核心业务逻辑 > 80%。
  - 框架：Jest、Vitest、Node.js Test Runner。

- **集成测试（Integration Test）**：
  - 测试对象：组件组合、页面模块、API 集成。
  - 特点：验证多个单元协作是否正常。
  - 场景：表单提交（组件 + 验证逻辑 + API）、购物车（状态 + 组件 + 计算）。
  - 框架：React Testing Library、Vue Test Utils、MSW（Mock Service Worker）。

- **E2E 端到端测试（End-to-End）**：
  - 测试对象：完整用户链路，从页面打开到业务完成。
  - 特点：最接近真实用户行为，但速度慢、不稳定（受网络/环境因素影响）。
  - 场景：登录 → 下单 → 支付 → 查看订单。
  - 框架：Playwright、Cypress、Selenium。

### 2. 现代单元测试架构

- **Vitest vs Jest**：
  - **Jest**：生态最成熟，但基于 Babel，启动较慢。
  - **Vitest**：原生 ESM 支持，与 Vite 生态无缝集成，速度更快，API 兼容 Jest。
  - 选型建议：Vite 项目首选 Vitest，存量 Webpack 项目可用 Jest。

- **测试替身（Test Doubles）**：
  - **Mock**：完全模拟依赖行为，返回固定值。
  - **Stub**：提供预定义响应，无实际逻辑。
  - **Spy**：包装真实函数，记录调用信息但不改变行为。
  - **Fake**：有实际工作的简化实现（如内存数据库替代真实数据库）。

- **快照测试（Snapshot Testing）**：
  - 将复杂输出（如组件渲染结果）序列化保存，后续对比差异。
  - 适用：UI 组件结构、API 响应格式。
  - 注意：快照更新需要人工审查，避免盲目更新掩盖问题。

- **属性测试（Property-Based Testing）**：
  - 不测试特定输入，而是定义"属性"（如"排序后数组长度不变"）。
  - 工具：fast-check，自动生成大量随机输入验证属性。

### 3. 组件测试与 Testing Library 哲学

- **Testing Library 核心原则**：
  - **测试行为，而非实现**：不测试 state 值，测试用户能看到什么。
  - **查询优先级**：优先使用 `getByRole`、`getByLabelText`（符合无障碍），避免 `getByTestId`（与实现耦合）。
  - **用户中心**：模拟真实用户操作（点击、输入），而非直接调用组件方法。

- **组件测试模式**：
  ```typescript
  // ❌ 测试实现细节
  expect(wrapper.vm.isOpen).toBe(true);
  
  // ✅ 测试用户可见行为
  expect(screen.getByText('确认删除')).toBeVisible();
  ```

- **异步测试**：
  - `waitFor`：等待异步操作完成。
  - `findBy*`：自动重试的查询方法，内置等待。
  - `act`：包装触发状态更新的操作，确保断言在更新后执行。

- **Mock Service Worker（MSW）**：
  - 在浏览器/Node 中拦截 HTTP 请求，返回模拟响应。
  - 优势：不修改业务代码即可 Mock API，支持 REST 和 GraphQL。
  - 场景：前端独立开发（后端未就绪）、测试稳定（不受真实 API 影响）。

### 4. E2E 测试工程化

- **Playwright（推荐）**：
  - 微软出品，支持 Chromium/Firefox/WebKit 三引擎。
  - 自动等待：操作前自动等待元素可见/可交互，减少 `sleep`。
  - 追踪（Trace）：失败时生成完整操作录像、DOM 快照、网络日志。
  - 并行执行：多 worker 并行，大幅缩短测试时间。

- **测试数据管理**：
  - **动态数据**：使用 Factory 模式生成测试数据（如 faker.js）。
  - **数据隔离**：每个测试用例独立的数据集，避免相互影响。
  - **数据清理**：测试后清理创建的数据，保持环境干净。

- **测试环境策略**：
  - **本地开发**：快速反馈，运行核心用例。
  - **CI 环境**：完整回归，多浏览器、多分辨率。
  - **预发布**：生产数据镜像，验证真实场景。

- **Flaky Test 治理**：
  - Flaky Test：时而过时而不过的测试，严重侵蚀测试可信度。
  - 成因：异步等待不足、测试间数据污染、时间/随机数依赖。
  - 治理：重试机制（临时缓解）、根因修复（推荐）、隔离标记。

### 5. 可视化回归测试（VRT）

UI 的像素级差异检测，防止"样式回退"。

- **原理**：
  - 基准截图（Baseline）：主分支的组件/页面截图。
  - 对比截图（Current）：PR 分支的截图。
  - 像素对比：使用 pixelmatch 等算法检测差异，阈值以上标记为失败。

- **工具链**：
  - **Storybook**：组件级 VRT，每个 story 生成截图。
  - **Chromatic**：Storybook 官方 VRT 服务，自动对比、云端审查。
  - **Loki**：基于 Storybook 的开源 VRT，可自建。
  - **Percy**：通用的 VRT 平台，支持任意测试框架。

- **VRT 最佳实践**：
  - 组件级优先：页面级 VRT 易受数据影响，组件级更稳定。
  - 视觉降噪：隐藏动态内容（时间、随机头像）、固定视口。
  - 人工审查：不是所有差异都是 bug，需要设计师参与审查。

### 6. 测试覆盖率与质量门禁

- **覆盖率指标**：
  - **行覆盖率（Line Coverage）**：执行过的代码行比例。
  - **分支覆盖率（Branch Coverage）**：执行过的条件分支比例（如 if/else）。
  - **函数覆盖率（Function Coverage）**：被调用过的函数比例。
  - **语句覆盖率（Statement Coverage）**：执行过的语句比例。

- **门禁策略**：
  - CI 中设置阈值：行覆盖率 > 80%，分支覆盖率 > 70%。
  - 增量覆盖：新代码覆盖率必须 > 90%，防止技术债务累积。
  - 覆盖率报告：Codecov、Coveralls 集成 PR 评论，直观展示覆盖变化。

- **覆盖率陷阱**：
  - 高覆盖率 ≠ 高质量：可能测试了所有行，但没验证正确性。
  - 不要为了覆盖率而测试：私有方法、简单 getter/setter 无需测试。
  - 关注未覆盖的关键路径：支付、权限、数据一致性。

### 7. 性能测试与基准

- **基准测试（Benchmark）**：
  - 使用 Benchmark.js 或 Vitest Bench 对比函数性能。
  - 场景：算法优化前后对比、库选型（lodash vs 原生）。

- **内存泄漏测试**：
  - 使用 Chrome DevTools Protocol 在测试中采集堆快照。
  - 对比测试前后的内存占用，检测泄漏。

- **启动性能测试**：
  - 测量冷启动时间、首屏渲染时间。
  - 防止构建优化退化（如 bundle 体积膨胀）。

---

## 深入学习路线

```
入门（1-2月）
├── Vitest/Jest 基础：describe/it/expect、生命周期钩子
├── Testing Library：查询方法、事件触发、异步等待
├── MSW：API Mock 基础
├── 覆盖率报告： Istanbul + HTML 报告解读
└── 实战：为现有组件补充单元测试

进阶（3-4月）
├── Playwright E2E：Page Object 模式、 fixtures、并行执行
├── 组件测试深度：复杂交互、表单验证、错误边界
├── 可视化回归：Storybook + Chromatic/Loki 集成
├── CI 集成：GitHub Actions/Jenkins 测试流水线
├── 质量门禁：覆盖率阈值、自动化报告
└── 实战：完整测试套件覆盖核心业务流程

深入（6月+）
├── 测试架构设计：测试数据工厂、环境管理
├── Flaky Test 治理：根因分析、稳定性提升
├── 性能基准测试：Benchmark.js、性能回归检测
├── 自定义 Testing Library 扩展：业务专用查询
├── 大型项目测试策略：分层测试、契约测试
└── 开源贡献：Vitest、Testing Library、Playwright
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Vitest](https://vitest.dev/) | 新一代单元测试框架，Vite 原生支持 |
| [Jest](https://jestjs.io/) | 最流行的 JS 测试框架 |
| [Testing Library](https://testing-library.com/) | 用户为中心的组件测试 |
| [Playwright](https://playwright.dev/) | 现代 E2E 测试框架 |
| [Cypress](https://www.cypress.io/) | 前端 E2E 测试 |
| [MSW](https://mswjs.io/) | Mock Service Worker，API Mock |
| [Storybook](https://storybook.js.org/) | 组件开发与测试 |
| [Chromatic](https://www.chromatic.com/) | Storybook 官方 VRT 服务 |
| [Loki](https://loki.js.org/) | 开源可视化回归测试 |
| [faker.js](https://fakerjs.dev/) | 测试数据生成 |
| [Codecov](https://codecov.io/) | 覆盖率报告平台 |
| [fast-check](https://github.com/dubzzz/fast-check) | 属性测试库 |
