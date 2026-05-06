# 十、浏览器安全攻防体系

前端安全不是"加个 CSP 头就好了"，而是一个多层次的纵深防御体系。攻击面远不止 XSS——从网络传输层到 JS 执行层，从用户输入到第三方依赖，每一层都有被攻破的可能性。

---

## 核心挑战

### 1. XSS 深度防御

跨站脚本攻击（XSS）是最古老也最持久的前端安全威胁。

- **存储型 XSS**：恶意脚本被持久化到数据库，其他用户访问页面时触发。
  - 典型场景：评论区输入 `<script>` 标签 → 存储到 DB → 所有访问者加载时执行。
  - 防御：服务端输出编码（HTML Entity 转义）+ 前端二次校验。

- **反射型 XSS**：恶意脚本在 URL 参数中，服务器直接拼入响应 HTML。
  - 典型场景：`https://site.com/search?q=<script>alert(1)</script>` → 服务器在页面中直接渲染 `q` 参数。
  - 防御：永远不要将 URL 参数未经转义地插入 HTML。

- **DOM 型 XSS**：纯前端 JS 漏洞，恶意数据通过 DOM API 注入页面。
  - 典型场景：`element.innerHTML = userInput`（不经过服务端）。
  - 防御：
    - **永远不要用 `innerHTML` / `document.write` 处理不可信数据**。
    - 使用 `textContent` 设置文本内容、`setAttribute` 设置属性。
    - 如果必须使用 `innerHTML`，先用 DOMPurify 做 HTML 消毒（sanitization）。

- **富文本防注入**：
  - 富文本编辑器允许用户提交 HTML（如粗体/斜体/链接），但必须截断危险标签和属性。
  - DOMPurify：白名单过滤 HTML，允许安全标签（b/i/a）但移除 `<script>`、`onerror`、`onclick` 等。
  - 服务端二次过滤必不可少（前端过滤可以被绕过）。

- **CSP（Content Security Policy）**：
  - HTTP 响应头 `Content-Security-Policy: script-src 'self'` 阻止加载外部脚本。
  - CSP 不是 XSS 的"完全阻止"，而是"纵深防御"——即使 XSS 注入成功，CSP 也能阻止攻击脚本执行。
  - `nonce`：为每个合法脚本生成一次性 nonce，只有带正确 nonce 的脚本才能执行。
  - `strict-dynamic`：由合法脚本动态创建的 `<script>` 也被信任。

### 2. 前端加密与 Web Crypto API

- **Web Crypto API**：
  - `crypto.subtle` 提供低级加密操作：RSA-OAEP（非对称加密）、AES-GCM/CBC（对称加密）、HMAC（消息认证）、ECDSA（数字签名）。
  - SHA 系列哈希摘要（`crypto.subtle.digest`）。
  - PBKDF2 密钥派生（从密码派生加密密钥）。

- **前端加密的局限**：
  - **JavaScript 加密不可信**：攻击者如果控制了前端（XSS 或恶意扩展），可以替换加密函数，获取明文。前端加密仅保护"传输过程"（TLS 之外额外加密），不能替代服务端加密。
  - 国密算法（SM2/SM4）：浏览器不支持（Web Crypto 仅有国际标准算法），JS 实现存在侧信道攻击风险。正确做法是后端处理或由硬件安全模块（HSM）提供。

- **端到端加密（E2EE）**：
  - 消息发送者加密 → 只有接收者能解密，服务端存储的是密文。
  - Web Crypto API 可以在浏览器生成密钥对，私钥不离开设备。

### 3. Cookie / Token 安全管控

- **Cookie 安全属性**：
  - `HttpOnly`：JS 无法读取，防止 XSS 脚本窃取 Cookie。
  - `Secure`：仅 HTTPS 传输。
  - `SameSite`：防止 CSRF——`Strict`（同站请求才带 Cookie）、`Lax`（默认，允许 GET 导航）、`None`（跨站也带，须配合 Secure）。

- **Token 存储策略**：
  - **不要**将敏感 token 存在 localStorage（可被 XSS 脚本读取）。
  - 推荐：HttpOnly Cookie + 短期 Access Token（如 15 分钟有效）+ Refresh Token。Access Token 失窃影响有限（很快过期），Refresh Token 有 HttpOnly 保护。

- **CSRF 防御**：
  - CSRF Token（同步令牌模式）：页面加载时从服务端获取随机 token → 每个状态变更请求带上 token → 服务端校验。
  - SameSite Cookie（Lax/Strict）是最简单有效的 CSRF 防御。
  - 自定义请求头（如 X-Requested-With）：跨域请求无法设置自定义头（有预检保护）。

- **跨域身份安全**：
  - OAuth 2.0 / OIDC（OpenID Connect）在前端的实现：PKCE（Proof Key for Code Exchange）流程，防止授权码拦截攻击。
  - 前端 OAuth 只能使用 Authorization Code + PKCE，禁止 Implicit Grant（隐式授权，Token 在 URL 中易泄露）。

### 4. 第三方依赖安全

前端项目通常有数百到数千个 npm 依赖，供应链攻击面极大。

- **供应链攻击**：
  - 恶意包名抢注（typosquatting）：`react` vs `reacet`。
  - 依赖混淆：公开发布与私有包同名的包。
  - 被黑维护者：攻破 npm 账号，向流行包推送恶意代码（如 event-stream 事件）。

- **Subresource Integrity（SRI）**：
  - `<script src="https://cdn.com/lib.js" integrity="sha384-xxx" crossorigin="anonymous">`。
  - 浏览器在加载资源后校验哈希，不匹配则拒绝执行。
  - 保护从 CDN 加载的公共库不被篡改。

- **定期审计**：
  - `npm audit`：检查已知漏洞。
  - `npm ls`：审计依赖树，发现异常引入的依赖。
  - Lockfile 完整性：`package-lock.json` / `yarn.lock` 必须提交，防止 CI 安装到的版本与本地不一致。

### 5. 防调试与代码加固

- **反调试**：
  - 检测 DevTools 是否打开：`window.outerWidth - window.innerWidth > 100`（非可靠方法，仅作提醒）。
  - `debugger;` 无限循环：打开 DevTools 时大量断点引发性能问题（攻击性手段，会被浏览器保护机制限制）。
  - 源码映射保护：生产环境谨慎暴露 source map（`.map` 文件）。

- **代码混淆**：
  - 变量名缩短、控制流扁平化（If → Switch）、字符串加密。
  - 混淆 ≠ 安全，只是提高逆向成本。关键安全逻辑（加密、密钥生成）必须在服务端。

- **完整性校验**：
  - 检测页面 JS 是否被篡改：计算自身脚本的哈希并与服务端期望值对比。
  - 检测到异常 → 上报安全事件 + 重定向到安全页面。

---

## 深入学习路线

```
入门（1-2月）
├── XSS 三种类型（存储/反射/DOM）原理与防御
├── CSP 配置详解：script-src、style-src、frame-ancestors
├── Web Crypto API 基础：加密、解密、哈希
└── 最小安全审计

进阶（3-4月）
├── CSRF 攻击原理与多层防御（Token/SameSite/自定义头）
├── Token 安全架构：Access Token + Refresh Token 轮换
├── 代码混淆与反调试原理
├── 第三方依赖安全审计与 SBOM
└── 实战：前端安全审计清单

深入（6月+）
├── 浏览器安全模型深度：同源策略、沙箱、进程隔离
├── 端到端加密：Signal Protocol 思想、双棘轮
├── 零信任前端架构：不信任任何源，每次请求独立认证
├── 隐私计算：联邦学习浏览器端可行性
└── 安全工具深度使用：Burp Suite、OWASP ZAP
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [DOMPurify](https://github.com/cure53/DOMPurify) | HTML 消毒库，XSS 防御 |
| [Helmet](https://helmetjs.github.io/) | Express 安全响应头中间件 |
| [CSP Evaluator](https://csp-evaluator.withgoogle.com/) | Google CSP 配置评估工具 |
| [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) | MDN 文档 |
| [OWASP Top 10](https://owasp.org/www-project-top-ten/) | Web 安全十大风险 |
| [Snyk](https://snyk.io/) | 依赖漏洞扫描 |
| [Retire.js](https://retirejs.github.io/) | 检测使用了已知漏洞版本的 JS 库 |
