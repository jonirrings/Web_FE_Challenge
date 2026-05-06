# 五、WebAssembly Wasm 底层高性能重构

WebAssembly 将 C/C++/Rust 等底层语言的性能带到浏览器。它不是要替代 JavaScript，而是在计算密集型场景中提供**接近原生的执行速度**（通常是 JS 的 10-50 倍）。Wasm 的真正价值在于将已有的成熟 C/C++ 生态（FFmpeg、SQLite、OpenCV）直接搬到前端。

---

## 核心挑战

### 1. C/C++/Rust 编译到浏览器

不同源语言的编译链：

- **Rust → Wasm**（推荐）：
  - `wasm-pack` 工具链一站式：Rust 编写 → `wasm-bindgen` 生成 JS 绑定 → 发布 npm 包。
  - 优势：Rust 的内存安全保证延续到 Wasm，零额外成本抽象。
  - 典型项目：SWC（Rust→JS/TS 编译器，比 Babel 快 20-70x）、Prisma（数据库 ORM 查询引擎）。

- **C/C++ → Wasm**（Emscripten）：
  - `emcc` 编译器将 C/C++ 代码编译为 Wasm + JS 胶水代码。
  - 可以移植成熟的 C 库：FFmpeg（音视频编解码）、SQLite（嵌入式数据库）、OpenCV（计算机视觉）。
  - 缺点：生成的 JS 胶水代码较大，需要手动优化。

- **Go → Wasm**：
  - `GOOS=js GOARCH=wasm go build`。Go runtime 会被编译进 Wasm（增加 ~1-2MB 体积）。
  - 适合 Go 后端的计算逻辑前移，但体积和性能不如 Rust。

- **AssemblyScript**：
  - TypeScript 语法 → Wasm 的直译器，学习成本低。
  - 缺点：功能受限（无闭包、无 async），生态不成熟。

### 2. JS 与 Wasm 双向通信

JS 和 Wasm 之间的数据交换是性能的关键路径。

- **基本通信方式**：
  - 简单数值（i32/i64/f32/f64）：通过函数参数和返回值，几乎零开销。
  - 字符串/复杂数据：通过 `WebAssembly.Memory`（线性内存）共享。

- **零拷贝通信**（SharedArrayBuffer）：
  - JS 和 Wasm 共享同一块 `WebAssembly.Memory`，两边都可以直接读写。
  - JS 写入数据 → Wasm 直接读取处理 → JS 读取结果，全程无拷贝。
  - **部署约束**：SharedArrayBuffer 要求页面启用**跨域隔离**（`Cross-Origin-Opener-Policy: same-origin` + `Cross-Origin-Embedder-Policy: require-corp`）。这与第三方嵌入（埋点、广告、CDN 视频）冲突，需要权衡。

- **批量传输**：
  - 避免频繁的单次调用（跨 JS-Wasm 边界有固定开销），将多次调用的数据打包为批量请求一次性传递。

### 3. Wasm 多线程

- **Web Workers + Wasm**：
  - Wasm 本身不直接创建线程（没有 OS 线程概念），通过共享 `WebAssembly.Memory` + Web Worker 实现并行。
  - 多个 Worker 实例化同一个 Wasm 模块，通过 `SharedArrayBuffer` 共享线性内存。

- **原子操作**：
  - Wasm 的 `i32.atomic` 系列指令：`atomic.load` / `atomic.store` / `atomic.rmw.cmpxchg`。
  - 对应 C 的 `std::atomic` 和 Rust 的 `AtomicI32`。

- **线程数限制**：
  - `navigator.hardwareConcurrency` 返回可用逻辑核心数。
  - 移动端通常 4-8 核，桌面端 8-16 核。创建超过物理核心数的线程是反优化。

- **应用场景**：
  - 图像/视频处理：每个 Worker 处理一帧或一个图像区域。
  - 加密/哈希：并行计算不同的数据块。
  - 编译/构建：SWC 利用多 Worker 并行编译 TS 文件。

### 4. 超大二进制体积优化

Wasm 模块的体积直接影响页面加载体验。

- **编译优化影响体积**：
  - `-O3` / `-Oz`：Rust/C 编译优化级别对 Wasm 体积影响显著。`-Oz` 以速度为代价换最小体积，`-O3` 反之。
  - `wasm-opt`（Binaryen 工具）：对 Wasm 二进制做后优化，可再缩减 10-30%。

- **代码裁剪**：
  - `wasm-snip`：删除 Wasm 中未使用的函数。
  - LTO（Link Time Optimization）：编译时全模块分析，自动消除死代码。Rust 的 `lto = true` 通常缩减 20-40% 体积。

- **压缩**：
  - Brotli 压缩后 Wasm 通常缩减 50-70%（比 gzip 好 10-20%）。
  - 服务端必须配置 Wasm MIME type + Brotli 支持。

- **分包加载**：
  - 将 Wasm 拆分为主模块 + 按需加载的功能模块。
  - 例如 ffmpeg.wasm：基础解码器立即加载，特殊编码器按需加载。

- **懒初始化**：
  - Wasm 模块的实例化（`WebAssembly.instantiate`）可能需要数百 ms → 首屏不急需的 Wasm 延迟到 `requestIdleCallback` 或用户交互后初始化。

### 5. WASI 与沙箱安全

WASI（WebAssembly System Interface）将 Wasm 从浏览器扩展到操作系统层面。

- **WASI 标准**：
  - `wasi_snapshot_preview1`：当前主流版本，提供文件系统、环境变量、时钟、随机数等系统接口。
  - `wasi:http` / `wasi:cli` / `wasi:sockets`：新版 WASI 0.2（组件模型），提供网络、命令行等能力。

- **权限模型**：
  - WASI 默认无权限，需要显式授予（如 `--dir=./data` 只允许访问特定目录）。
  - 浏览器中的 Wasm 默认不能访问文件系统和网络（通过 JS 代理），WASI 在浏览器中目前受限。

- **浏览器中的 WASI**：
  - WasmEdge / Wasmer 等运行时可以在浏览器中嵌入一个"伪 WASI"环境。
  - 真正完整的 WASI 目前在 Node.js 服务端有更好的支持。

- **Wasm 组件模型（Component Model）**：
  - 将不同语言的 Wasm 模块组合成一个应用——如 Rust 写的计算引擎 + JS 写的 UI 逻辑，通过规范化的接口描述（WIT）连接。

---

## 深入学习路线

```
入门（1-2月）
├── Rust 基础：Rustlings 练习
├── wasm-pack 工具链：wasm-pack build → npm link
├── 第一个 Rust → Wasm 函数：斐波那契
└── wasm-bindgen：JS 与 Rust 结构体互调

进阶（3-4月）
├── Wasm 内存模型：Linear Memory、页分配、数据视图
├── SharedArrayBuffer + Web Worker 并行计算
├── WASI 标准理解：在 Node.js 中运行 Wasm 系统调用
├── 性能优化：内联函数标记、SIMD 指令使用
└── 实战：用 Wasm 加速图像/视频处理（Canvas + Wasm）

深入（6月+）
├── 自研 Wasm 运行时：理解指令解码、栈机执行
├── WASI 沙箱安全模型深度：Capability-based Security
├── Component Model：多语言组件组合、WIT 接口定义
└── 开源项目阅读：WasmEdge、Wasmer、wasmtime 源码
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [wasm-pack](https://rustwasm.github.io/wasm-pack/) | Rust → Wasm 构建工具链 |
| [wasm-bindgen](https://rustwasm.github.io/wasm-bindgen/) | Rust/JS 互操作绑定 |
| [Emscripten](https://emscripten.org/) | C/C++ → Wasm 编译器 |
| [Binaryen](https://github.com/WebAssembly/binaryen) | Wasm 二进制优化工具链 |
| [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm) | FFmpeg 的浏览器 Wasm 版本 |
| [SWC](https://swc.rs/) | Rust 编写的 JS/TS 编译器，基于 Wasm |
| [WasmEdge](https://wasmedge.org/) | CNCF 运行时，支持 WASI + 浏览器嵌入 |
| [Wasmer](https://wasmer.io/) | 通用 Wasm 运行时，支持 WASI |
