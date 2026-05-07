# GlobalPay — 全球化支付钱包

> 跨境支付钱包，多币种、多语言、强安全、强合规

---

## 核心功能

- 多币种账户：查看/转换/收付款
- 交易记录：历史列表、详情、导出
- 跨境转账：汇率查询、费用预估、提交转账
- 安全认证：Passkeys 登录、支付密码、多因子认证
- 无障碍：全键盘操作、屏幕阅读器兼容
- 离线查看：断网查看交易记录与账户余额

---

## 挑战落地

### 10 安全 — 端到端防护

- **XSS 防御**：所有用户输入转义，DOMPurify 清洗富文本，CSP 禁止 inline script
- **CSRF 防御**：SameSite Cookie + Double Submit Token
- **Web Crypto**：交易签名（ECDSA），敏感数据加密存储（AES-GCM）
- **Token 安全**：Access Token 短效 + Refresh Token 轮转，存储在 httpOnly Cookie
- **反调试**：生产环境检测 DevTools，关键逻辑混淆

**练习要点**：
- CSP 策略配置与 violation 监控
- Web Crypto API 签名/验签流程
- Token 安全存储方案对比（Cookie vs localStorage vs SessionStorage）
- 安全头部完整配置

### 22 WebAuthn — 现代认证

- **Passkeys 注册**：注册流程引导用户创建 Passkey（指纹/Face ID/PIN）
- **Passkeys 登录**：条件 UI 自动提示，一键登录
- **支付确认**：大额转账二次确认，Passkey 作为第二因子
- **安全模型**：FIDO2 协议，凭证 ID + 公钥存储在服务端，私钥永远不出设备

**练习要点**：
- WebAuthn 注册/认证流程
- Conditional UI 体验优化
- 跨设备 Passkey 同步机制
- 降级方案（设备不支持 Passkey → TOTP）

### 20 国际化 — 全球适配

- **RTL 布局**：阿拉伯语/希伯来语镜像布局（margin/padding/border-radius 自动翻转）
- **多币种格式化**：Intl.NumberFormat 按地区展示货币符号、千分位、小数位
- **动态语言包**：按需加载当前语言，首屏不加载全部翻译
- **CJK 排版**：中文/日文特殊断行规则、字体回退链

**练习要点**：
- CSS 逻辑属性（margin-inline-start 替代 margin-left）
- Intl API 完整使用（NumberFormat / DateTimeFormat / Segmenter）
- i18n 框架（react-intl）集成
- RTL 测试自动化

### 19 无障碍 — 包容性设计

- **WCAG 2.1 AA 合规**：颜色对比度 ≥ 4.5:1、焦点可见、文本可缩放 200%
- **屏幕阅读器**：ARIA 标签完善，交易列表可读性优化（role="row" + aria-label）
- **键盘导航**：Tab 顺序合理，转账流程全键盘可操作，Skip Link 跳过导航
- **视觉无障碍**：高对比度模式、大字体模式、减少动画偏好

**练习要点**：
- ARIA 属性正确使用（aria-label / aria-live / aria-expanded）
- 键盘焦点管理（roving tabindex）
- axe / Lighthouse 自动化无障碍检测
- 屏幕阅读器测试（VoiceOver / NVDA）

### 13 性能优化 — 极致首屏

- **首屏 FCP < 1s**：Critical CSS 内联 + JS 延迟加载 + 预连接 API 域名
- **交易列表虚拟滚动**：万级交易记录只渲染可见行（react-window / 自研）
- **内存泄漏监控**：长时间停留在交易列表页，监听组件卸载后 EventListener/Timer 是否清理
- **主线程调度**：汇率刷新等低优先级任务放 requestIdleCallback

**练习要点**：
- Critical Path 分析与优化
- 虚拟滚动实现（固定行高 / 动态行高）
- Chrome DevTools Memory 面板泄漏排查
- PerformanceObserver 采集 Long Task

### 06 PWA — 离线可用

- **离线查看**：Service Worker 缓存交易记录与账户余额，断网可查看
- **后台同步**：离线期间标记的交易状态变更，联网后 Background Sync 同步
- **推送通知**：交易到账、汇率变动 Push Notification

**练习要点**：
- Service Worker 缓存策略（NetworkFirst for API / CacheFirst for static）
- IndexedDB 存储离线数据
- Background Sync API
- Push API + Notification API

### 24 Serverless — 边缘加速

- **边缘渲染**：多区域边缘 SSR，就近返回首屏 HTML（Vercel Edge / Cloudflare Workers）
- **边缘函数**：汇率查询缓存到边缘 KV Store，减少中心 API 调用
- **前端直连数据库**：边缘函数代理查询交易记录，前端无需自建后端

**练习要点**：
- Next.js Edge Runtime 开发
- 边缘缓存策略（stale-while-revalidate）
- 边缘 KV Store 读写

---

## 技术栈建议

```
前端框架：React 18+
状态管理：Zustand
路由：React Router v6
样式：CSS Modules + CSS Variables（设计 Token）
国际化：react-intl
认证：WebAuthn API + @simplewebauthn
离线：Workbox（Service Worker）+ IndexedDB
边缘：Vercel Edge Functions / Cloudflare Workers
构建工具：Vite
测试：Playwright + Vitest + axe-core
```
