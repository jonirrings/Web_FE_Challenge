# 六、离线 PWA 渐进式网页应用

PWA（Progressive Web Application）将 Web 应用提升到接近原生应用的用户体验。核心能力：离线可用、安装到桌面、后台推送。PWA 不是一项单一技术，而是 **Service Worker + Web App Manifest + Cache API + IndexedDB + Push API** 的组合拳。

---

## 核心挑战

### 1. Service Worker 全链路缓存策略

Service Worker 是 PWA 的核心——一个运行在浏览器后台的 JS 代理，拦截所有网络请求。

- **生命周期管理**：
  - Install → Waiting → Activate → Fetch → Terminated → 再次唤醒。
  - 关键点：新 SW 安装后会进入 `waiting` 状态（旧 SW 仍在服务页面），直到旧 SW 控制的页面全部关闭。`skipWaiting()` 可以强制立即激活，但可能导致版本不兼容。

- **缓存策略矩阵**：
  | 策略 | 行为 | 适用场景 |
  |------|------|----------|
  | Cache First | 缓存命中则返回，否则请求网络并缓存 | 静态资源（JS/CSS/Font） |
  | Network First | 网络请求，失败则返回缓存 | API 数据 |
  | Stale-While-Revalidate | 返回缓存同时更新缓存 | 图片、用户头像 |
  | Network Only | 永远请求网络 | 支付、敏感操作 |
  | Cache Only | 永远返回缓存 | 预缓存的离线内容 |

- **缓存版本与淘汰**：
  - 使用版本化的 Cache Name（如 `v2-assets`），升级时删除旧版本缓存。
  - `Cache.addAll()` 批量缓存，任一失败则全部回滚。
  - 缓存空间有限（各浏览器在 20-200MB 之间），LRU 算法手动管理淘汰。

- **静默更新**：
  - SW 后台检查更新（浏览器每 24 小时自动检查，也可手动触发 `registration.update()`）。
  - 新版本下载后不立即激活，等用户刷新或弹窗提示"新版本可用"。

### 2. 离线复杂业务闭环

离线不仅仅是"缓存页面"，而是完整的离线业务逻辑。

- **离线表单提交**：
  - 用户离线填写表单 → 数据存入 IndexedDB 队列 → 联网后自动提交。
  - 复杂场景：订单、报销单等需要附件（图片/文件）的表单。
  - 关键设计：**乐观 UI**——用户看到"已提交"界面，实际后台排队等待发送。

- **数据队列**：
  - 所有操作（增删改）序列化为操作日志存入 IndexedDB。
  - 联网后按顺序回放操作到服务端。
  - 幂等设计：每条操作带唯一 ID，服务端保证重复执行不产生副作用。

- **冲突校验**：
  - 离线修改的数据在服务器也可能被其他人修改。
  - 解决：使用版本号/ETag。提交时检查服务端版本 → 冲突时提示用户手动合并或使用服务端版本。

- **在线状态检测**：
  - `navigator.onLine` 不可靠（只检测网络连接，不检测实际可达性）。
  - 正确做法：定时 ping 服务端 health check 端点 + `online`/`offline` 事件。

### 3. 原生级能力补齐

PWA 的目标是将 Web 体验拉到原生水平。

- **桌面快捷方式（Add to Home Screen）**：
  - `manifest.json` 配置图标（至少 192x192 + 512x512）、名称（short_name + name）、主题色、启动模式（standalone/fullscreen）。
  - Chrome 有 `beforeinstallprompt` 事件可以自定义安装提示。

- **状态栏通知（Push Notification）**：
  - Push API：服务端 → 浏览器推送服务（FCM/Mozillla APNs/Apple APNs）→ Service Worker → Notification API。
  - 关键：**必须在 SW 中订阅和接收**（即使在页面关闭后）。
  - 通知点击 → SW 处理 → 打开指定页面 → 聚焦到相关内容。

- **后台同步（Background Sync）**：
  - `navigator.serviceWorker.ready.then(reg => reg.sync.register('sync-tag'))`。
  - 浏览器在网络恢复时自动触发 SW 的 `sync` 事件 → 执行数据同步。
  - 注意：Background Sync API 在各浏览器实现差异巨大，Chrome 相对完整，Safari 不支持。

- **全屏沉浸式**：
  - `display: standalone` + `display-mode: fullscreen`。
  - CSS `env(safe-area-inset-*)` 适配刘海屏和安全区域。
  - Web App Banner：`meta name="apple-mobile-web-app-capable"` 支持 iOS。

### 4. 存储极限方案

PWA 的本地存储能力是离线体验的物理基础。

- **IndexedDB 深度使用**：
  - 实际可用容量：桌面端 ~60% 磁盘剩余空间，移动端 ~20% 剩余空间。Chrome 对单个源限制约 2GB（动态）。
  - 事务 ACID：IndexedDB 的事务是自动提交的（不可手动 commit/rollback），但支持读/写锁。
  - 索引设计：组合索引提高查询效率。类似数据库的"慢查询分析"一样，需要关注索引命中率。

- **分片存储**：
  - 海量数据拆分到多个 Object Store 或按时间/ID 范围分片。
  - 类似数据库分表——避免单个 Store 过大导致写入阻塞。

- **数据备份与恢复**：
  - 定期导出 IndexedDB 数据到文件（JSON 导出）。
  - 服务端备份：增量同步到后端数据库。

### 5. 多端状态离线一致性

多设备登录同一账号时，各设备的离线修改如何合并？

- **同步策略**：
  - Last-Write-Wins（LWW）：简单但丢数据（一个设备的修改覆盖另一个）。
  - 版本向量（Version Vector）：追踪每个设备的操作版本，服务端做冲突检测。
  - CRDT：将协同编辑的最终一致性思想用于数据同步。

- **数据版本控制**：
  - 每条数据记录带 `updatedAt` + `deviceId`。
  - 服务端维护全量的变更日志（Change Log），设备 reconnect 时重放变更。

---

## 深入学习路线

```
入门（1-2月）
├── Service Worker 生命周期：install/activate/fetch 事件
├── manifest.json 配置：图标、名称、启动模式
├── Cache API：缓存静态资源、离线可用
└── 最小 PWA：可安装、可离线

进阶（3-4月）
├── 离线数据队列：IndexedDB + 操作日志 + 自动回放
├── IndexedDB 深度：事务、组合索引、容量管理
├── Push Notification：VAPID 密钥、FCM 推送
├── Background Sync API 实践
└── 实战：完整离线待办/笔记应用

深入（6月+）
├── 复杂业务离线闭环：订单+附件离线提交
├── 数据加密：IndexedDB 中存储加密数据
├── 多设备同步架构：增量同步 + 冲突解决
├── Desktop PWA：文件处理、快捷方式、协议注册
└── 与原生应用体验对齐：启动速度、动画流畅度
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Workbox](https://developer.chrome.com/docs/workbox/) | Google SW 缓存策略库，大幅简化 SW 开发 |
| [idb](https://github.com/jakearchibald/idb) | IndexedDB Promise 封装，API 优雅 |
| [Dexie.js](https://dexie.org/) | IndexedDB 封装库，支持复杂查询 |
| [localForage](https://localforage.github.io/localForage/) | 多存储后端（IndexedDB/WebSQL/localStorage）的统一 API |
| [web-push](https://github.com/web-push-libs/web-push) | 服务端 Web Push 库（Node.js），VAPID 密钥管理 |
| [PWA Builder](https://www.pwabuilder.com/) | PWA 打包与检测工具 |
