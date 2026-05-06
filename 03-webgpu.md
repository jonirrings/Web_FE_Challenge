# 三、WebGPU 高性能图形/可视化引擎

WebGPU 是继 WebGL 2.0 之后的下一代浏览器图形 API，对标现代底层 API（Vulkan/Metal/DirectX 12）。它给前端打开了**GPU 原生级编程能力**的大门——不仅是画三角形的游戏，更是 GPU 通用并行计算（GPGPU）进入浏览器的里程碑。

---

## 核心挑战

### 1. GPU 底层管线编程

WebGPU 完全抛弃了 WebGL 的固定管线（gl.useProgram / gl.uniform / gl.bindTexture），要求开发者手动管理渲染管线的每一个阶段。

- **渲染管线（Render Pipeline）**：
  - Vertex Shader → Rasterization → Fragment Shader → Output Merger
  - 管线是**不可变对象**（创建后不能修改），这与 WebGL 的状态机模型截然不同。好处是 GPU 可以大幅优化（无需运行时状态验证），代价是设计时需要更仔细的管线规划。

- **绑定组（Bind Group）**：
  - WebGL 的 uniform/texture 绑定顺序是隐式的（gl.activeTexture + gl.bindTexture），WebGPU 用显式 Bind Group 一次性声明所有资源绑定。
  - Bind Group Layout 定义了"管线期望什么样的资源"，Bind Group 提供实际数据。布局不对 → 直接报错。

- **WGSL 着色器语言**：
  - WGSL（WebGPU Shading Language）是 W3C 专为 WebGPU 设计的着色器语言，语法接近 Rust。
  - 业界常用 **GLSL → naga → WGSL** 转换链路（naga 是 WebGPU 生态的标准着色器编译库）。
  - 着色器编译在浏览器运行时完成（不同于 WebGL 的即时编译），可以提前检查语法错误。

- **缓冲区（Buffer）与纹理（Texture）**：
  - Buffer 用于顶点数据、索引、uniform、storage（计算着色器读写）。
  - Texture 用于图像采样、渲染目标。
  - 两者数据不能混用，需要通过 Copy Command 转换。

### 2. GPGPU 通用并行计算

WebGPU 的革命性特性不是画图，而是**计算着色器（Compute Shader）**。

- **计算管线（Compute Pipeline）**：
  - 完全脱离渲染流程，纯 GPU 数据并行处理。
  - 工作网格（Workgroup Grid）：将数据切分为 workgroup（工作组），每个 workgroup 内有 N 个线程并行执行同一个 Compute Shader。
  - 典型应用：物理碰撞检测、粒子系统更新、数据聚类、图像处理（卷积/滤波）、AI 推理加速。

- **Storage Buffer**：
  - 计算着色器可以直接读写 storage buffer（GPU 端可读写内存）。
  - 数据流转：JS → `device.queue.writeBuffer()` → GPU Compute → `device.queue.readBuffer()` → JS。这是 WebGPU 的 GPGPU 核心路径。

- **GPGPU 典型工作流**：
  ```
  JS: 准备数据 → GPU Buffer
  GPU: Compute Shader 并行处理
  JS: 读回结果 或 直接渲染到 Canvas
  ```

### 3. 海量可视化渲染

千万级数据的可视化必须将渲染逻辑下放 GPU，JS 只能做调度。

- **实例化渲染（Instanced Rendering）**：
  - 一个 draw call 渲染成千上万个相同几何体的变体（不同位置/颜色/大小）。
  - 典型场景：散点图（百万点）、粒子系统、瓦片地图中的标记点。

- **视口裁剪（View Frustum Culling）**：
  - GPU 端裁剪：在 Compute Shader 中计算哪些物体在视口内，将可见列表写入 Indirect Buffer，然后 Indirect Draw。
  - CPU 端裁剪（降级方案）：JS 做空间索引（四叉树/R-Tree），仅提交可见部分给 GPU。

- **LOD（Level of Detail）**：
  - 远处的物体用简化几何体、近处用精细几何体。
  - GPU LOD：根据物体到摄像机的距离动态选择渲染的 LOD 级别。

- **绘制批次瓶颈**：
  - 每次 `draw()` 调用都有固定开销。瓶颈不在 GPU 算力，而在 CPU→GPU 的 draw call 数量。
  - 优化：合批（Batching）、实例化渲染、Indirect Draw（GPU 端生成绘制指令）。

### 4. 3D 物理引擎自研

将物理计算搬到 GPU 是大规模物理模拟的关键。

- **刚体物理**：
  - 碰撞检测（Broad Phase / Narrow Phase）：Broad Phase 用空间哈希在 Compute Shader 中并行计算，Narrow Phase 逐对精确检测。
  - 约束求解：迭代求解器（Gauss-Seidel）可以映射到 GPU 并行。

- **软体/布料**：
  - 质点-弹簧模型（Mass-Spring），每个质点的力计算独立 → GPU 并行。
  - 位置动力学（Position Based Dynamics）：天然适合 GPU 并行。

- **流体模拟**：
  - SPH（Smoothed Particle Hydrodynamics）：每个粒子与邻域粒子交互。
  - GPU 加速邻域搜索（Uniform Grid），百万粒子实时模拟成为可能。

### 5. 跨平台渲染兼容

WebGPU 的浏览器支持正在快速推进，但兼容仍是现实问题。

- **当前支持**：
  - Chrome/Edge：桌面端和 Android 已全面支持。
  - Firefox：Nightly 版本支持。
  - Safari：macOS/iOS 近期版本已支持。

- **降级方案**：
  - 使用 `navigator.gpu` 特性检测，不支持时 fallback 到 WebGL 2.0。
  - Dawn/Vulkan：Chrome 的 WebGPU 实现基于 Dawn（C++），支持的 API 子集不断变化。
  - 移动端差异：移动 GPU（Adreno/Mali）的 WGSL 支持、存储限制、工作组大小限制都有差异。

### 6. GPU 内存管控

浏览器中的 GPU 内存管理是一个灰色地带——你能分配但不能直接控制回收。

- **GPU 内存泄漏**：
  - 创建 Buffer/Texture 后未 `destroy()` → GPU 内存不释放。
  - JS GC 不会自动触发 GPU 内存回收（JS 对象被 GC 了，但 GPU 资源可能未回收）。
  - 浏览器通常有 GPU 内存上限（取决于设备），超限 → WebGPU 上下文丢失。

- **缓冲区复用**：
  - 频繁创建/销毁 Buffer 是性能大忌，应该预分配大型 Buffer 池，通过 offset 复用。
  - Ring Buffer 模式：循环使用 staging buffer 的区间。

- **纹理压缩**：
  - Basis Universal（ETC2/ASTC/BC7 统一压缩格式）：前端一次加载，GPU 端解码为平台原生格式。
  - ASTC（移动端）/ BC7（桌面端）：需要检测 `navigator.gpu.wgslLanguageFeatures` 判断硬件支持的压缩格式。

### 7. Shader 性能极致优化

- **分支化简**：GPU 的线程束（Warp/Wavefront，通常 32/64 线程）内，如果线程走向不同分支，两个分支都会执行（SIMT 串行化）。关键：避免基于线程 ID 的分支。
- **向量并行**：WGSL 内置 `vec4f` 等向量类型，一次指令操作 4 个 float。尽量让算法向量化。
- **片元着色器裁剪**：早期片元测试（Early Fragment Test）避免对深度测试不通过的片元执行着色器。
- **带宽优化**：减少采样次数（Texture Cache）、减少非连续内存访问（Coalesced Access）。

---

## 深入学习路线

```
入门（1-2月）
├── WebGPU 官方规范通读（MDN / w3c/GPUWeb）
├── W3C WebGPU 教程（webgpufundamentals.org）
├── WGSL 着色器语言基础：类型系统、内置函数
└── 实现最小三角形 → 纹理立方体

进阶（3-4月）
├── 渲染管线全流程：Vertex → Fragment → Output
├── 纹理采样器（Sampler）配置：寻址/过滤/Mipmap
├── 计算着色器实践：GPU 矩阵乘法
└── 实战：粒子系统（GPU 更新 + 渲染）

深入（6月+）
├── 性能优化：Instanced Drawing、Indirect Draw、批处理
├── 视口裁剪 + LOD 完整实现
├── WebGPU + AI 推理：运行 ONNX 模型（ONNX Runtime Web GPU 后端）
├── 图形学基础：线性代数、渲染方程、PBR 物理着色
└── WebGPU 引擎原理：Three.js WebGPU 后端源码阅读
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [webgpufundamentals.org](https://webgpufundamentals.org/) | WebGPU 最佳入门教程 |
| [naga](https://github.com/gfx-rs/naga) | GLSL/WGSL 着色器编译器，WebGPU 生态基础设施 |
| [Three.js WebGPU](https://threejs.org/examples/#webgpu) | Three.js 的 WebGPU 渲染后端 |
| [wgpunn](https://github.com/praeclarum/wgpunn) | WebGPU 实现的神经网络推理 |
| [Babylon.js WebGPU](https://doc.babylonjs.com/features/featuresDeepDive/webGPU) | Babylon.js 的 WebGPU 支持 |
| [WebGPU Samples](https://github.com/webgpu/webgpu-samples) | 官方示例集合 |
