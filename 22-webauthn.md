# 二十二、WebAuthn 现代身份认证

WebAuthn（Web Authentication API）是 W3C 和 FIDO 联盟制定的**无密码认证标准**，被 Apple、Google、Microsoft 共同推动。它的核心思想是：用设备内置的生物识别（指纹/面容）或硬件安全密钥替代密码，私钥永不离开用户设备，从根本上消灭密码泄露和钓鱼攻击。

---

## 核心挑战

### 1. WebAuthn 协议落地

WebAuthn 基于非对称加密——用户设备生成密钥对，私钥存储在设备安全区（TPM/Secure Enclave），公钥注册到服务端。

- **注册流程（Registration / Create）**：
  ```
  1. 用户点击"注册 Passkey"
  2. 前端调用 navigator.credentials.create({ publicKey: options })
     - options 包含：challenge（服务端生成的随机值）、rp（Relying Party 信息）、user（用户标识）
  3. 浏览器弹出系统认证 UI（指纹/面容/安全密钥/PIN）
  4. 认证器生成公私钥对 → 用私钥签名 challenge → 返回公钥 + 签名
  5. 前端将公钥和签名发送到服务端 → 服务端验证签名并存储公钥
  ```

- **认证流程（Authentication / Get）**：
  ```
  1. 用户点击"登录"
  2. 前端调用 navigator.credentials.get({ publicKey: options })
     - options 包含：challenge、allowCredentials（允许的凭证 ID 列表，或留空使用 discoverable credentials）
  3. 浏览器弹出系统认证 UI → 用户验证身份
  4. 认证器用私钥签名 challenge → 返回签名
  5. 前端将签名发送到服务端 → 服务端用注册时的公钥验证签名 → 登录成功
  ```

- **Challenge-Response 机制**：
  - 每次注册/认证，服务端生成**随机且一次性**的 challenge（通常 32 字节随机数）。
  - 防重放攻击：攻击者截获签名无法用于另一次登录（challenge 已变化）。
  - Challenge 应有时效性（如 5 分钟过期）。

- **RP（Relying Party）服务端**：
  - RP = 你的网站/应用服务器。
  - 需要存储：用户 ID、Credential ID、公钥（COSE 格式）、签名计数器。
  - 推荐使用成熟的 RP 库：SimpleWebAuthn（Node.js）、Duo WebAuthn（Python）。

### 2. Passkeys 无密码方案

Passkeys 是 WebAuthn 的进化——**可发现凭证（Discoverable Credentials）** + **跨设备同步**。

- **可发现凭证**：
  - 传统 WebAuthn：服务端存储用户的所有 Credential ID → 认证时告诉浏览器允许哪些凭证。
  - Passkeys：凭证信息（Credential ID + 用户名 + 显示名）存储在认证器内部 → 浏览器可以自主"发现"可用凭证 → 无需服务端先提供 allowCredentials。
  - 结果：用户在登录页面只需输入用户名（或自动填充），不需要事先注册设备。

- **跨设备同步**：
  - iCloud Keychain：Apple 设备间的 Passkeys 通过 iCloud 端到端加密同步。
  - Google Password Manager：Android/Chrome 设备间同步。
  - 1Password / Dashlane：第三方密码管理器也支持 Passkeys。
  - **意义**：用户换手机不会丢失所有账号——这与传统硬件安全密钥的根本区别。

- **跨设备认证（Hybrid Transport / QR Code + Bluetooth）**：
  - 场景：在 Windows 桌面浏览器上登录，用 iPhone 扫码 → iPhone 通过蓝牙验证身份 → 桌面端登录成功。
  - 协议：FIDO 联盟的 **Cross-Device Authentication Flow** (formerly CABLE)。
  - 前提：两台设备在物理上接近（蓝牙范围）。

### 3. 多因子认证编排

WebAuthn 可以独立使用，也可以与传统认证方式混合编排。

- **WebAuthn + WebOTP（短信验证码）**：
  - WebOTP API：`navigator.credentials.get({ otp: { transport: ['sms'] } })` → 浏览器自动从短信中提取验证码（无需手动输入）。
  - 场景：WebAuthn 作为主因子（生物识别），WebOTP 作为第二因子（短信验证码）→ 达到多因子认证。

- **降级策略**：
  - 设备不支持 WebAuthn（如旧版浏览器、无生物识别硬件的桌面）→ 回退到传统密码 + 短信/邮件验证码。
  - 渐进式引入：先为高端用户（iPhone/Windows Hello 设备）开启 Passkeys，其他用户保持原密码登录。

- **Conditional UI（条件 UI / Conditional Mediation）**：
  - 用户在登录表单中点击用户名输入框 → 浏览器自动弹出 Passkey 选择器（无需先点击"登录"）。
  - 实现：在 `get()` 调用中设置 `mediation: 'conditional'`。
  - UX 优势：**免交互登录**——用户看不到"登录"按钮，开始输入用户名时自动弹出生物识别提示 → 指纹/面容验证 → 直接登录。

- **UV（User Verification） vs UP（User Presence）**：
  - UP（用户存在）：简单的点击/触摸，证明有人在操作。不需要 PIN/指纹。
  - UV（用户验证）：指纹、面容、PIN 码，证明是特定用户。
  - `userVerification: 'required'` 强制要求 UV（适合高安全场景）；`'preferred'` 优选项但允许降级；`'discouraged'` 不需要。

### 4. 安全模型深度

- **私钥不可导出**：
  - 认证器的私钥存储在硬件安全区域（TPM / Secure Enclave / Titan M），软件无法读取或导出私钥。
  - 即使攻击者完全攻破操作系统，也无法提取私钥（硬件隔离保证）。

- **RP ID 绑定防钓鱼**：
  - 私钥在创建时绑定了 RP ID（域名的 eTLD+1，如 `google.com`、`github.com`）。
  - 用户在钓鱼网站 `go0gle.com` 上登录 → 认证器找不到对应 RP ID 的私钥 → 登录失败。
  - 这是 Passkeys 相比密码的**最大安全优势**——用户不会被钓鱼骗走密码，因为根本就没有密码。

- **Attestation 认证声明**：
  - 注册时，认证器可以返回 Attestation Statement（认证证明），证明公钥确实是由某个可信的认证器生成的。
  - Attestation 类型：None（匿名，无认证器信息）、Indirect（由中间层签名）、Direct（认证器制造商签名）。
  - 高安全场景（如金融、政府）需要验证 Attestation，确保用户使用了合规的硬件认证器（如 FIPS 认证的 YubiKey）。

- **签名计数器**：
  - 认证器内置签名计数器：每次签名操作计数 +1。
  - 服务端存储上次的计数器值 → 新的签名计数器必须大于旧值 → 防止凭证克隆重放。

### 5. 多平台兼容

- **桌面端支持**：
  - Windows：Windows Hello（指纹/面容/PIN）+ 外部安全密钥（USB/NFC YubiKey）。
  - macOS：Touch ID + 外部安全密钥。
  - Linux：通常依赖外部安全密钥（USB YubiKey）。

- **移动端支持**：
  - iOS：Face ID / Touch ID + iCloud Keychain Passkeys 同步。
  - Android：指纹 / 图案 / PIN + Google Password Manager 同步。

- **浏览器差异**：
  - Chrome/Edge：完整支持，包括 Conditional UI。
  - Safari：基本支持，Passkeys 同步（iCloud Keychain）集成良好。
  - Firefox：基本支持，Passkeys 同步支持较弱。

- **Polyfill 策略**：
  - 不支持 WebAuthn 的浏览器 → 回退到传统密码认证。
  - `navigator.credentials && navigator.credentials.create` 特性检测。

---

## 深入学习路线

```
入门（1-2月）
├── WebAuthn 规范（w3c/webauthn Level 2/3）通读
├── MDN Web Authentication API 文档
├── 理解注册/认证两步的核心流程
├── SimpleWebAuthn 库使用（前端 + 服务端）
└── 最小 Demo：用户名注册 + 指纹登录

进阶（3-4月）
├── Passkeys：可发现凭证 + 跨设备同步
├── Conditional UI 免交互登录实现
├── RP Server 完整实现：Challenge 管理、公钥存储、计数器校验
├── 跨设备认证（Hybrid Transport / QR + Bluetooth）
└── 实战：WebAuthn + 传统密码双因子应用

深入（6月+）
├── FIDO2/CTAP2 协议深度：认证器与客户端的通信协议
├── Attestation 验证：Direct/Indirect/None 类型安全分析
├── 企业级身份认证架构：SSO + WebAuthn + LDAP 混合
├── 隐私保护：域内匿名凭证、不可链接认证
└── 参与 W3C WebAuthn Adoption Community Group
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [SimpleWebAuthn](https://simplewebauthn.dev/) | WebAuthn Node.js 前后端库，最易上手 |
| [WebAuthn Spec](https://www.w3.org/TR/webauthn-3/) | W3C WebAuthn Level 3 规范 |
| [FIDO Alliance](https://fidoalliance.org/) | FIDO2/Passkeys 标准制定组织 |
| [FIDO2 CTAP](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/) | 客户端到认证器协议规范 |
| [WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API) | 短信验证码自动填充 |
| [passkeys.dev](https://passkeys.dev/) | Passkeys 开发指南和最佳实践 |
| [WebAuthn.io](https://webauthn.io/) | Duo 实验室的 WebAuthn 在线 Demo |
