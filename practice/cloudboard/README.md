# CloudBoard — 实时协作白板

> 类 Miro/Figma，多人同时在一个无限画布上绘图、贴便签、连线

---

## 核心功能

- 无限画布：缩放、平移、无限延伸
- 多人实时协作：光标同步、语音通话
- 图形绘制：矩形、圆形、线条、自由画笔
- 便签与文本：创建/编辑/拖拽便签
- 连线与箭头：元素间智能连线
- 离线支持：断网继续绘图，联网自动同步
- 导出：画布导出为 SVG/PNG

---

## 挑战落地

### 01 WebRTC — 实时通信

- **光标同步**：每个协作者的光标位置通过 WebRTC DataChannel 实时广播，延迟 < 50ms
- **语音通话**：画布内 1 对 1 / 多人语音，WebRTC GetUserMedia 采集 + RTCPeerConnection 传输
- **信令协商**：WebSocket 做信令服务器，ICE 候选交换

**练习要点**：
- 实现 Signaling Server（WebSocket）
- ICE/STUN/TURN 配置与 NAT 穿透
- DataChannel 可靠/不可靠模式选择
- 音频 3A（AEC/ANS/AGC）处理

### 02 CRDT — 冲突合并

- **便签位置冲突**：多人同时拖拽同一便签，使用 LWW-Register（Last Writer Wins）合并位置
- **文本内容冲突**：便签内文本编辑，使用 YATA 算法（Yjs）实现字符级冲突合并
- **离线改写**：断网期间本地操作记录到 Yjs Doc，联网后自动 merge

**练习要点**：
- 集成 Yjs，理解 YATA 算法原理
- 本地持久化（IndexedDB 存储 Yjs Update）
- 感知协作（Awareness）：谁在线、光标在哪

### 03 WebGPU — 高性能渲染

- **万级图形批量渲染**：矩形/圆形/线条通过 WebGPU Render Pipeline 批量绘制，单帧 10K+ 元素
- **画布缩放平移**：Uniform Buffer 传入 Transform Matrix，GPU 侧完成坐标变换
- **自由画笔**：Path 数据上传 GPU，Vertex Shader 平滑曲线

**练习要点**：
- WebGPU Render Pipeline 搭建
- Uniform/Storage Buffer 管理
- 多 Pass 渲染（背景 → 图形 → 选中高亮）
- GPU Picking（点击检测）

### 05 WebAssembly — 高性能计算

- **SVG 编码**：C++ 移植的 SVG 路径解析器，编译为 Wasm，处理复杂路径运算
- **PNG 编码**：Wasm 版 libpng，导出大画布为 PNG（比 Canvas toBlob 快 3-5x）
- **碰撞检测**：Wasm 实现图形间碰撞检测算法

**练习要点**：
- C++ → Emscripten 编译流程
- JS-Wasm 内存通信（SharedArrayBuffer）
- Wasm 多线程（SharedArrayBuffer + Web Workers）

### 06 PWA — 离线可用

- **Service Worker 缓存**：App Shell + 静态资源预缓存，离线可打开应用
- **画布快照**：定时将画布状态序列化存入 IndexedDB，断网后恢复
- **Background Sync**：离线期间的编辑操作排队，联网后自动同步到服务端

**练习要点**：
- Service Worker 生命周期与缓存策略（StaleWhileRevalidate）
- IndexedDB 封装（idb / Dexie.js）
- Background Sync API

### 13 性能优化 — 大画布流畅

- **虚拟滚动**：只渲染视口内的图形元素，视口外元素从 DOM/渲染管线中移除
- **合成层隔离**：每个图形元素独立合成层，避免重绘扩散
- **内存泄漏排查**：监听画布元素创建/销毁，确保 EventListener 和 GPU Buffer 正确释放
- **主线程调度**：requestIdleCallback 处理低优先级任务（如自动保存）

**练习要点**：
- Performance面板分析帧率瓶颈
- Chrome DevTools Memory 面板排查泄漏
- requestAnimationFrame vs requestIdleCallback 调度

### 21 CSS 新能力 — 交互增强

- **View Transitions**：画布主题切换（亮色/暗色）时的平滑过渡动画
- **Container Queries**：侧边栏组件库面板根据容器宽度自适应布局
- **Scroll Animations**：缩放时间线动画

**练习要点**：
- `@view-transition` + `view-transition-name` 配置
- `@container` 规则与 `container-type` 设置
- `animation-timeline: scroll()` 使用

---

## 技术栈建议

```
前端框架：React 18+
状态管理：Zustand（轻量、支持订阅选择器）
CRDT：Yjs + y-websocket
渲染引擎：WebGPU（@webgpu/types）
Wasm：Emscripten / wasm-pack
离线存储：IndexedDB（Dexie.js）
构建工具：Vite
```
