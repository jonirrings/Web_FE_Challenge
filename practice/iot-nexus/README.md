# IoT Nexus — 物联网监控大屏

> 智能工厂/智慧城市场景，设备数据实时采集 + 3D 数字孪生 + 告警

---

## 核心功能

- 设备管理：蓝牙/串口/NFC 连接传感器设备
- 实时数据流：WebSocket 推送设备采集数据
- 可视化大屏：时序图、GIS 热力图、3D 数字孪生
- 智能告警：异常检测 + 多渠道通知
- 安全通信：端到端加密、设备双向认证
- 边缘计算：数据预处理下沉到边缘节点

---

## 挑战落地

### 08 硬件交互 — 设备直连

- **Web Bluetooth**：扫描并连接 BLE 传感器（温湿度/气压），读取 GATT Characteristic 数据
- **Web Serial**：通过串口读取工控设备数据（Modbus RTU 协议解析）
- **WebNFC**：NFC 标签识别设备 ID，碰一碰配对
- **WebXR**：AR 模式下叠加设备数据到真实设备画面上

**练习要点**：
- Web Bluetooth API 流程（扫描 → 连接 → 发现服务 → 读特征值）
- 串口数据帧解析（Modbus CRC 校验）
- NFC NDEF 消息解析
- WebXR AR Session 管理

### 09 网络架构 — 实时数据管道

- **WebSocket 集群**：前端连接 WS 网关，网关后端多节点，支持水平扩展
- **MQTT-over-WS**：前端 MQTT.js 客户端订阅设备 Topic，QoS 1 保证至少一次送达
- **断线重连**：指数退避重连 + 消息补发（服务端缓存离线期间消息）
- **流量削峰**：前端采样聚合（1s 内多个数据点取均值再渲染），避免高频渲染卡顿

**练习要点**：
- WebSocket 心跳与重连策略
- MQTT QoS 级别选择与消息去重
- 背压处理（Backpressure）：数据消费速度 < 生产速度时如何降级
- 消息序列化（Protocol Buffers / MessagePack 压缩传输）

### 12 大数据可视化 — 海量数据渲染

- **时序图**：万级数据点流式渲染，WebGL 绘制（uplot / 自研），降采样算法（LTTB）
- **GIS 热力图**：Mapbox GL + Deck.gl 渲染设备分布热力图，支持百万级 POI
- **拓扑图**：设备关系拓扑，力导向布局（d3-force），Canvas 批量渲染
- **3D 数字孪生**：WebGPU 渲染工厂 3D 模型，设备状态实时映射（颜色/动画）

**练习要点**：
- 降采样算法（LTTB / Min-Max）减少渲染点数
- WebGL Instanced Rendering 批量绘制
- Deck.gl Layer 性能优化（数据分块、GPU 加速）
- glTF 模型加载与 PBR 材质渲染

### 17 可观测性 — 全链路监控

- **RUM 指标**：采集 FCP/LCP/CLS/INP，上报到监控平台
- **异常捕获**：全局 JS Error 捕获 + 未处理 Promise Rejection + 资源加载失败
- **性能告警**：大屏帧率 < 30fps 自动告警，WebSocket 断连率 > 5% 告警
- **OpenTelemetry**：前端 Trace（页面加载 → API 调用 → WS 消息），与后端 Trace 串联

**练习要点**：
- PerformanceObserver 采集 Web Vitals
- SourceMap 反解线上错误堆栈
- OpenTelemetry SDK 集成（@opentelemetry/web）
- 告警规则配置与降噪

### 24 Serverless — 边缘计算

- **边缘函数**：设备数据预处理下沉到边缘节点（单位转换、阈值判断），减少中心端压力
- **边缘渲染**：大屏首屏在边缘节点 SSR，就近返回 HTML，FCP < 500ms
- **前端直连数据库**：边缘函数代理查询时序数据库，前端无需自建后端

**练习要点**：
- Cloudflare Workers / Vercel Edge Functions 开发
- 边缘 SSR 框架（Next.js Edge Runtime / Remix）
- 边缘缓存策略（Cache API / KV Store）

### 10 安全 — 端到端防护

- **设备双向 TLS**：前端 + 设备双向证书认证，防止伪造设备接入
- **Web Crypto**：敏感数据端到端加密（AES-GCM），服务端无法解密
- **CSP**：严格 Content-Security-Policy，防止 XSS 注入第三方图表脚本
- **反调试**：检测 DevTools 打开行为，生产环境混淆关键逻辑

**练习要点**：
- Web Crypto API（SubtleCrypto）加解密流程
- CSP 策略配置与 violation 上报
- 证书固定（Certificate Pinning）在浏览器的实现
- 安全头部（HSTS/X-Content-Type-Options）

---

## 技术栈建议

```
前端框架：React 18+
图表引擎：ECharts（时序）+ Deck.gl（GIS）+ 自研 WebGPU（3D 孪生）
设备通信：Web Bluetooth API / Web Serial API / MQTT.js
实时推送：WebSocket（native）+ MQTT-over-WS
监控：OpenTelemetry + 自建 RUM
边缘：Cloudflare Workers
构建工具：Vite
```
