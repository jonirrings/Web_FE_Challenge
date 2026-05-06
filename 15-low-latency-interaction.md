# 十五、低时延交互控制系统

低时延交互系统的目标是：**用户在浏览器中的操作在 < 100ms 内得到视觉反馈**，无论背后是多复杂的网络或计算。这涵盖了在线游戏和远程桌面两大方向，它们的核心问题相似——网络延迟与本地响应之间的矛盾。

---

## 核心挑战

### 1. 游戏前端引擎

浏览器游戏不是"画个 Canvas 小人"——而是完整的帧循环、物理、渲染、网络同步体系。

- **游戏循环（Game Loop）**：
  - `requestAnimationFrame(gameLoop)` 驱动每帧的更新 + 渲染。
  - Delta Time（帧间隔）用于帧率无关的物理更新：`position += velocity * deltaTime`。
  - 目标帧率：60fps（16.6ms/帧），移动端可降为 30fps。

- **帧同步 vs 状态同步**：
  - **帧同步（Lockstep）**：所有玩家输入广播到所有客户端 → 每帧执行同样的逻辑 → 确定性保证状态一致。
    - 优势：带宽极小（只传输入指令）。
    - 劣势：任一玩家网络延迟阻塞所有人（需要输入缓冲延迟补偿）。
  - **状态同步（State Sync）**：服务端是权威状态源，客户端是近似渲染。
    - 优势：单个玩家的延迟不影响其他人。
    - 劣势：带宽大（服务端需下发完整状态），客户端预测易出错。

- **客户端预测与服务器校准**：
  - **预测（Prediction）**：用户按下"前进"键 → 客户端立即移动角色（不等服务器确认）→ 消除感知延迟。
  - **校准（Reconciliation）**：服务器返回真实位置 → 客户端对比预测位置 → 误差大时拉回修正。
  - **插值（Interpolation）**：其他玩家的状态在本地做插值平滑（如等 100ms 的历史状态做视觉插值），掩盖网络抖动。

- **物理引擎**：
  - 常见的 JS 物理引擎：Matter.js（2D 刚体）、Planck.js（Box2D 的 JS 移植）、Cannon-es（3D）。
  - 物理引擎集成到游戏循环：`physicsWorld.step(deltaTime)` → 读取物体位置 → 渲染。

### 2. 远程桌面 Web 远程控制

在浏览器中实现类似 VNC/RDP 的远程桌面体验。

- **屏幕差分传输**：
  - 不是全屏截图传输（一帧 1080p 图片 > 200KB → 30fps = 6MB/s，带宽不可接受）。
  - 差分编码：只传输当前帧与上一帧不同的区域（Dirty Rect → 变化矩形）。
  - 变化矩形可以用 JPEG 压缩（照片内容）或 PNG 压缩（UI 界面，大块纯色区域）。

- **键鼠指令透传**：
  - 本地键盘/鼠标事件（mousedown/mousemove/keydown）→ 封装为指令 → 发送到远程服务器。
  - 特殊键处理：快捷键组合（Ctrl+C/Ctrl+V）、系统键（Win/Command）、多键同时按。
  - 光标渲染：是客户端渲染光标（零延迟感知）还是服务端回传光标图像（1帧延迟）？推荐客户端渲染。

- **画面压缩策略**：
  - H.264/H.265 视频编码：对变化画面用视频编码器压缩（利用帧间预测），比逐帧 JPEG 高效数倍。
  - WebCodecs API 可以调用硬件编码器（GPU 加速），延迟极低。
  - WebRTC 数据通道传输编码后的视频帧。

- **延迟指标**：
  - 端到端延迟（点击 → 画面反馈）应 < 100ms 才能有"操作本地机器"的感觉。
  - 延迟分解：输入传输（~10ms）+ 远程处理（~20ms）+ 视频编码（~20ms）+ 传输（~10ms）+ 解码渲染（~20ms）= ~80ms 理论值。

---

## 深入学习路线

```
入门（1-2月）
├── Canvas 2D / WebGL 游戏渲染基础
├── 游戏循环实现：requestAnimationFrame + Delta Time
├── 物理引擎使用：Matter.js / Planck.js
└── 最小游戏 Demo（Flappy Bird / 贪吃蛇）

进阶（3-4月）
├── 网络同步策略：帧同步 vs 状态同步，选型与实践
├── 客户端预测 + 服务器校准 + 插值渲染
├── 物理引擎集成：碰撞检测、刚体、约束
└── 实战：WebSocket 多人简易游戏

深入（6月+）
├── WebRTC Data Channel：极低延迟 P2P 数据传输
├── 远程桌面协议：VNC/RDP 前端实现（如 noVNC）
├── 服务器权威架构：确定性逻辑 + 回滚（Rollback Netcode）
└── 游戏引擎原理：Unity WebGL 导出、ECS 架构
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Matter.js](https://brm.io/matter-js/) | 2D 刚体物理引擎 |
| [Planck.js](https://piqnt.com/planck.js/) | Box2D 的 JS 移植 |
| [Cannon-es](https://github.com/pmndrs/cannon-es) | 3D 物理引擎 |
| [noVNC](https://novnc.com/) | VNC 协议的浏览器客户端 |
| [Phaser](https://phaser.io/) | 2D 游戏框架 |
| [Babylon.js](https://www.babylonjs.com/) | 3D 游戏/渲染引擎 |
| [Colyseus](https://colyseus.io/) | Node.js 多人游戏服务器框架 |
