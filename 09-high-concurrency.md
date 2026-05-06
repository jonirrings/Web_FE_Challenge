# 九、前端高并发高可用架构

当在线用户从千级跨越到百万级时，前端架构面临根本性挑战。不再是"能连上就行"，而是**连接可靠、消息不丢、状态一致、性能可控**。这涉及 WebSocket 集群设计、消息推送全链路、前端限流削峰、以及跨 Tab/窗口的分布式状态管理。

---

## 核心挑战

### 1. 百万级长连接 WebSocket 集群

单一 WebSocket 服务器（如单个 Node.js 进程）能承载的连接数上限约为 10,000-50,000（受限于单机端口数和文件描述符）。百万连接必须分布式。

- **连接负载均衡**：
  - 多台 WS 服务器 → 前端连接时通过负载均衡分配（通常由 Nginx/HAProxy 或网关层路由）。
  - 一致性哈希：根据用户 ID 做哈希 → 同用户始终路由到同一台服务器（会话亲和）。

- **心跳保活**：
  - 浏览器 WebSocket 默认无心跳，需要应用层实现 PING/PONG。
  - 心跳间隔：一般 15-30 秒。过短浪费带宽，过长连接被中间代理掐断。
  - 重连策略：指数退避（Exponential Backoff）—— 1s → 2s → 4s → 8s → 最大 30s。

- **消息幂等**：
  - 网络断连重试 → 同一条消息可能被发送多次 → 服务端必须幂等处理。
  - 前端每条消息带唯一 `messageId`（UUID），服务端基于 ID 去重。

- **广播分组**：
  - 万人在线直播间：一条消息需要广播给同房间的一万人。
  - Publish/Subscribe 模型：用户订阅 Topic（房间 ID）→ 消息发布到 Topic → 网关广播给订阅者。

- **连接迁移**：
  - WS 服务器重启/宕机 → 上面的连接需要迁移到其他服务器。
  - 前端重连到新服务器 → 新服务器从消息队列/DB 恢复连接状态。

### 2. 消息推送全链路

浏览器消息推送不是单一通道，而是多通道混合降级。

- **推送通道对比**：
  | 通道 | 延迟 | 离线可达 | 兼容性 | 适用场景 |
  |------|------|----------|--------|----------|
  | WebSocket | < 50ms | 否 | 几乎所有浏览器 | 实时双向通信 |
  | SSE（Server-Sent Events） | < 100ms | 否 | Chrome/Firefox/Safari | 单向事件推送 |
  | Web Push | 秒级 | 是 | Chrome/Firefox/Edge | 离线通知 |
  | 厂商推送（APNs/FCM） | 不确定 | 是 | 需原生 APP 或 PWA 安装 | 移动端离线通知 |

- **混合降级策略**：
  - 优选用 WebSocket 做双向实时通信。
  - WS 断连 → 降级到 SSE + HTTP Polling（长轮询）。
  - 用户离线 → 回退到 Web Push / 厂商推送。
  - 前端需实现通道优先级管理，自动检测并切换到最佳可用通道。

- **消息可达性**：
  - 在线消息（WS/SSE）：实时推送，无需持久化。
  - 离线消息：存储在服务端，用户上线后批量推送（Push History）。
  - 已读/未读状态同步：需要前后端协同设计 ACK 机制。

### 3. 前端流量削峰限流

高并发场景下，用户操作会产生流量洪峰（如秒杀、弹幕高峰期）。

- **接口防抖与节流**：
  - 防抖（Debounce）：用户停止输入 N ms 后才发请求（搜索框）。
  - 节流（Throttle）：在时间窗口内最多发一次请求（滚动加载）。

- **请求排队机制**：
  - 高并发请求不直接打到服务器 → 前端请求队列缓冲 → 异步批量/顺序发出。
  - 优先级调度：关键请求（支付、下单）插队处理，次要请求（埋点、日志）延迟发送。

- **请求合并（Batching）**：
  - 将多个独立 API 调用合并为一个批量请求。
  - 如 GraphQL 的多字段一次查询，或自定义 BFF 层的请求聚合。
  - 实现：在 Event Loop 的一个 Tick 内收集所有 API 调用，合并为一个请求发送。

- **服务端限流错误处理**：
  - 收到 429（Too Many Requests）/ 503（Service Unavailable）→ 前端自动退避重试。
  - 显示限流状态给用户（如"请求过于频繁，请稍后"）。

### 4. 分布式前端状态

同一用户可能打开多个 Tab/窗口，这些"前端实例"之间的状态需要保持同步。

- **跨 Tab 通信方式**：
  - **BroadcastChannel API**：同源页面间的广播消息（推荐）。
  - **SharedWorker**：多个 Tab 共享同一个 Worker 实例，Worker 中维护共享状态。
  - **localStorage + storage 事件**：A 页面修改 localStorage → B 页面收到 'storage' 事件。仅在同源非同 Tab 时触发。
  - **Service Worker 消息中继**：SW 作为页面间的消息中心。

- **状态同步场景**：
  - 用户在一个 Tab 中登录/登出 → 其他 Tab 自动更新登录状态。
  - 购物车：A Tab 加商品 → B Tab 购物车图标数字实时更新。
  - 暗色模式/主题切换 → 所有 Tab 同步切换。

- **锁机制**：
  - 某些操作不能同时在多 Tab 执行（如文件编辑、金融交易）。
  - 使用 `navigator.locks.request()`（Web Locks API）获取跨 Tab 互斥锁。

---

## 深入学习路线

```
入门（1-2月）
├── WebSocket 帧格式：理解 Opcode、Masking、Close Frame
├── 心跳机制：PING/PONG 与断线检测
├── 断线重连：指数退避、状态恢复
└── 实战：WebSocket 聊天 Demo

进阶（3-4月）
├── 百万级 WS 集群设计：负载均衡、会话亲和、消息路由
├── 消息队列模型：Pub/Sub、Topic 设计
├── SSE 与 WebSocket 混合推送方案
├── 跨 Tab 通信：BroadcastChannel + Web Locks API
└── 性能监控：连接数、消息吞吐量、延迟分布

深入（6月+）
├── 分布式 WebSocket 网关：连接迁移、灰度发布
├── MQTT over WebSocket：物联网消息协议浏览器端
├── 消息持久化：离线消息存储、已读状态追踪
└── 开源项目阅读：Socket.io、ws、uWebSockets.js 源码
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Socket.io](https://socket.io/) | 实时通信框架，自动降级（WS → HTTP 长轮询） |
| [ws](https://github.com/websockets/ws) | Node.js 高性能 WebSocket 库 |
| [uWebSockets.js](https://github.com/uNetworking/uWebSockets.js) | C++ 实现的超高性能 WebSocket |
| [Nchan](https://nchan.io/) | Nginx 模块实现的 Pub/Sub 消息服务 |
| [Centrifugo](https://centrifugal.dev/) | 开源实时消息服务器，支持多种协议 |
| [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) | MDN SSE 文档 |
| [BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API) | MDN BroadcastChannel API |
