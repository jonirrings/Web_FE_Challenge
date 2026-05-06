# 一、实时音视频体系 WebRTC

WebRTC 是浏览器端实时音视频通信的基石技术。与传统的直播推流（RTMP/HLS）不同，WebRTC 面向的是**超低延迟的点对点实时通信**，端到端延迟需控制在 200ms 以内。其复杂性不在于"能跑通"，而在于**大规模异构网络环境下稳定可靠地跑好**。

---

## 核心挑战

### 1. 网络自适应抗弱网

实时音视频最大的敌人不是代码，是网络。真实用户的网络环境千差万别：Wi-Fi 信号衰减、4G/5G 切换、电梯地铁盲区、跨国高延迟链路。

- **丢包（Packet Loss）**：UDP 不保证送达，视频帧的某几个包丢了会导致花屏甚至完全无法解码。应对手段包括：
  - **FEC 前向纠错（Forward Error Correction）**：发送端冗余编码，用额外 5%-20% 的带宽换回少量丢包下的完整恢复。低延迟场景（<100ms RTT）FEC 是首选。
  - **ARQ 自动重传（Automatic Repeat reQuest）**：接收端主动请求重传丢失包。缺点是增加一个 RTT 的延迟，高 RTT 网络下不可用。实践中通常组合：关键帧（I 帧）用 ARQ 重传，非关键帧用 FEC 兜底。
  - **NACK（Negative ACK）**：RTCP 协议中的丢包反馈机制，接收端告诉发送端哪些包没收到。与 ARQ 配合使用。

- **抖动（Jitter）**：网络包到达间隔不均匀，导致声音忽快忽慢、视频卡顿。
  - **JitterBuffer**：接收端维护一个缓冲队列，故意增加几十毫秒的延迟来平滑到达间隔。Buffer 过小 → 丢包增多；Buffer 过大 → 延迟增加。关键在于**自适应 JitterBuffer**：实时监测网络抖动水平，动态调整缓冲大小。

- **动态码率控制（GCC / BBR）**
  - Google Congestion Control (GCC)：WebRTC 默认的拥塞控制算法，基于延迟梯度（delay-based）和丢包率（loss-based）双重信号动态调整编码码率。
  - 网络变差时：先降分辨率 → 再降帧率 → 最后降码流质量。平滑降级，不直接断流。

- **新兴方案：WebTransport**
  - WebTransport 基于 HTTP/3 (QUIC)，相比 WebRTC 数据通道有**更低的队头阻塞**（QUIC 的多流特性）。对于自定义数据通道场景（非媒体流），WebTransport 在弱网下表现更优，但目前浏览器支持仍在普及中。

### 2. 多端媒体设备兼容

理论上 `getUserMedia` 几行代码拿到媒体流，现实中设备兼容性是噩梦。

- **设备枚举**：同一台笔记本可能有内置麦克风、外接摄像头、蓝牙耳机、USB 声卡、虚拟摄像头（OBS/VCam）。`navigator.mediaDevices.enumerateDevices()` 返回的设备列表在不同浏览器、不同操作系统下行为不一致——设备 label 可能为空（需要先获取权限）、deviceId 在页面刷新后可能变化。

- **蓝牙耳机路由**：用户接听/挂断电话时，音频输入输出设备动态切换。浏览器需要监听 `devicechange` 事件并平滑迁移音频轨道。iOS Safari 和 Android Chrome 在这里行为差异巨大。

- **虚拟摄像头**：OBS 虚拟摄像头、虚拟背景插件、Snap Camera 这类虚拟设备经常报告异常的分辨率/帧率能力，实际性能远低于宣称值。需要通过实际推流测试来验证设备能力。

- **权限模型差异**：
  - Chrome：需要 HTTPS 或 localhost，每次页面加载都需要重新授权（可设置持久化权限）
  - Safari：对权限更严格，iframe 跨域无法获取音视频权限
  - Firefox：支持多个版本的 `getUserMedia` 约束语法
  - iOS WKWebView：权限模型与 Safari 不同，有额外限制

### 3. 回声消除/降噪/自动增益（音频 3A）

- **AEC 回声消除（Acoustic Echo Cancellation）**：消除扬声器播放的声音被麦克风重新采集导致的回声。场景差异巨大：
  - **普通耳机场景**：浏览器内置 AEC3（Chrome）/ AEC（Firefox）已足够。
  - **外放场景**：手机外放/会议室音箱 → 回声路径长、非线性失真大，内置算法力不从心。
  - **高端需求**：需要自研音频 DSP 算法，用 ML 模型（如 RNNoise）做残差回声抑制。

- **ANS 降噪（Ambient Noise Suppression）**：去除环境噪声——键盘敲击、风扇、空调、街道噪音。浏览器内置降噪在稳态噪声（风扇）场景表现好，对非稳态噪声（键盘、狗叫）处理差。

- **AGC 自动增益（Automatic Gain Control）**：自动调节输入音量，防止远端听到的声音忽大忽小。多人场景下靠麦克风近的人声音大、远的人声音小，AGC 能部分缓解但不能根本解决。

- **自研 3A 路径**：通过 AudioContext / AudioWorklet 获取原始 PCM 音频帧 → Wasm 运行 C/C++ DSP 算法 → 处理后写入输出轨道。延迟控制在 10ms 以内才算合格。

### 4. ICE 信令协商与 NAT 穿透

WebRTC 的点对点直连在现实网络中面临大量 NAT 和防火墙阻挡。

- **STUN（Session Traversal Utilities for NAT）**：告诉客户端"你的公网 IP 和端口是什么"，用于大部分锥形 NAT 穿透。成本低（公共 STUN 服务器免费），但对称 NAT 打洞失败。

- **TURN（Traversal Using Relays around NAT）**：中继服务器转发所有流量。100% 穿透成功，但成本高——每个 TURN 连接消耗服务器带宽，1Mbps 视频流 = 1 小时 450MB 流量。需要自己部署或购买 TURN 服务。

- **ICE（Interactive Connectivity Establishment）**：STUN 和 TURN 的统一框架，收集所有可能的连接候选地址（host/srflx/relay），按优先级尝试连通。ICE 的坑：
  - **Trickle ICE**：逐个发送候选项减少连接建立时间，但与部分信令服务器/SFU 兼容差。
  - **ICE 超时**：默认约 30 秒，移动网络切换时可能需要更长。
  - **IPv6/IPv4 双栈**：候选地址可能混有 IPv6 和 IPv4，对端可能只支持其一。

- **内网多层路由**：企业内网通常有多层 NAT（路由器 + 防火墙 + VPN），穿透失败率极高。此时 TURN 是唯一解。

### 5. 屏幕共享 + 窗口区分

屏幕共享的复杂性不在于能共享，而在于**精细化控制共享内容**。

- **权限限制**：
  - Chrome：支持整个屏幕、应用窗口、浏览器标签页三种模式。标签页共享时浏览器自动给视频加红框标识。
  - Safari：仅支持整个屏幕或应用窗口，不支持标签页。
  - Firefox：粒度类似 Chrome 但有独立权限模型。
  - macOS 需要用户到系统偏好设置中额外授权"屏幕录制"权限。

- **帧率与画质权衡**：
  - 共享静态内容（PPT/文档）：低帧率（5-10fps）+ 高分辨率。
  - 共享视频/动画：高帧率（30fps）+ 动态码率。
  - 需要根据 `MediaStreamTrack` 的 `contentHint` 属性提示浏览器优化方向（`detail` vs `motion` vs `text`）。

- **光标渲染**：是否显示光标、光标样式（箭头/手型/文本选择）、光标位置，不同平台行为不一致。Chrome 标签页共享默认不渲染光标，窗口共享默认渲染。

- **区域裁剪**：`getDisplayMedia` 后通过 CropTarget 或 Canvas 二次处理可实现区域裁剪，但有性能损耗。

### 6. 直播连麦混合架构

这是 WebRTC 和传统直播的交叉地带。

- **架构模式**：
  - **SFU（Selective Forwarding Unit）**：服务器接收所有上行流，为每个参与者选择性转发（按需选择层/分辨率）。不对媒体做转码，CPU 开销低。
  - **MCU（Multipoint Control Unit）**：服务器将所有流混成一路合成流再下发。客户端只收一路流，CPU 低但服务器开销大。
  - **旁路转推**：WebRTC 互动流 → 服务端转换为 RTMP/HLS 流推向 CDN，让数万普通观众也能观看。

- **合流布局同步**：多画面布局（九宫格、演讲者模式），各画面必须音频口型对齐。通过 NTP 时间戳同步。

- **录制回放**：录制时需要保存每个参与者的独立流（不能只保存合流，否则无法回放时切换布局），存储和回放逻辑复杂。

### 7. 安全加密

- **DTLS-SRTP**：WebRTC 强制加密，不可关闭。
  - DTLS（Datagram Transport Layer Security）：基于 UDP 的 TLS，用于密钥协商。交换 SRTP 的加密密钥（master key + salt）。
  - SRTP（Secure Real-time Transport Protocol）：用协商好的密钥对媒体数据做 AES 加密和 HMAC-SHA1 完整性校验。
  - 整个过程即使在信令服务器被攻破的情况下，媒体流仍然是加密的（端到端），除非 TURN 服务器做中间人。

---

## 深入学习路线

```
入门（1-2月）
├── 《WebRTC 权威指南》
├── MDN WebRTC 文档通读
├── 理解 getUserMedia + RTCPeerConnection + RTCDataChannel 三件套
└── 最小 Demo：两个浏览器页面视频通话

进阶（3-4月）
├── ICE/STUN/TURN 原理：部署 Coturn 服务器
├── SDP 协议深度解析：offer/answer 模型、Plan B vs Unified Plan
├── 媒体流处理：MediaStream Track API、MediaRecorder 录制
├── 理解 JitterBuffer、NACK、FEC、GCC 拥塞控制
└── 实战：一对一视频通话（含信令服务器、TURN 部署）

深入（6月+）
├── 阅读 mediasoup/Janus SFU 源码
├── 音频 3A 算法原理：AEC 自适应滤波、RNNoise 降噪
├── 大并发 SFU 架构：负载均衡、级联
├── WebRTC 性能调优：码率控制策略、Simulcast/SVC 分层编码
└── 低延迟直播方案：WHIP/WHEP 协议
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [mediasoup](https://mediasoup.org/) | Node.js SFU，性能优秀，适合自建 RTC 基础设施 |
| [Janus](https://janus.conf.meetecho.com/) | 通用 WebRTC 网关，支持 SFU/MCU 插件架构 |
| [Coturn](https://github.com/coturn/coturn) | 开源 TURN/STUN 服务器 |
| [Pion](https://github.com/pion/webrtc) | Go 语言 WebRTC 实现，适合服务端开发 |
| [LiveKit](https://livekit.io/) | 新兴开源 WebRTC 平台，API 设计现代 |
| [RNNoise](https://github.com/xiph/rnnoise) | Mozilla 开源的 RNNoise 降噪算法，可编译到 Wasm |
