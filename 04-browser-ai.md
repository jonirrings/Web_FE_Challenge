# 四、浏览器端 AI 前端人工智能

浏览器端 AI 将模型推理从云端下放到用户设备，实现了**零延迟、离线可用、隐私保护**。核心矛盾在于：浏览器是一个受限的沙箱环境，而神经网络模型动辄数百 MB。如何在受限环境中高效运行 AI 模型是最大的挑战。

---

## 核心挑战

### 1. 推理框架选型与模型轻量化

浏览器端 AI 推理的主流框架：

- **TensorFlow.js**：
  - 支持 WebGL 和 WebGPU 后端，自动选择最佳 GPU 加速后端。
  - 可以直接加载 Keras / SavedModel / TF Hub 模型。
  - 缺点：运行时体积大（>1MB gzipped），对于简单推理任务过于臃肿。

- **ONNX Runtime Web**：
  - 支持 WebGL / WebGPU / Wasm 后端。
  - ONNX 是业界通用的模型交换格式（PyTorch → ONNX → Web）。
  - 轻量、高效，Wasm 后端在缺少 GPU 时仍可用。

- **模型量化压缩**：
  - **INT8 量化**：将 FP32 权重压缩为 8 位整数。模型体积缩为 1/4，推理速度提升 2-4 倍（取决于硬件 SIMD 支持）。精度损失通常 <1%。
  - **FP16 量化**：体积减半，精度几乎无损。适合 WebGPU 后端（原生支持 fp16）。
  - **权重剪枝（Pruning）**：移除接近零的权重 → 稀疏模型 → 体积缩小 30-50%。
  - **知识蒸馏（Knowledge Distillation）**：大模型教小模型，用小模型部署到浏览器。

- **模型分片加载**：大模型（如 Llama-2-7B）通过分片编码（sharded encoding）和按需加载（权重分片 mmap 到 IndexedDB），避免一次性下载全部。

### 2. 端侧 AI 实时处理

浏览器摄像头/麦克风 + 轻量化模型 = 前端实时 AI。

- **人脸检测与识别**：
  - BlazeFace（MediaPipe）：专为移动端设计的轻量人脸检测模型，单次推理 < 5ms（WebGPU）。
  - FaceMesh：468 个 3D 面部关键点，用于 AR 特效、虚拟试妆。

- **姿态识别**：
  - PoseNet / MoveNet（TensorFlow.js）：实时人体 17 个关键点检测。
  - 应用场景：健身动作检测、手势控制、虚拟交互。

- **图像分割**：
  - BodyPix / SelfieSegmentation（MediaPipe）：人像与背景分离。
  - 应用场景：视频会议虚拟背景（比 Zoom 的纯色幕布更精准）、图像编辑。

- **OCR 离线识别**：
  - Tesseract.js（Wasm 编译）：支持 100+ 种语言的离线文字识别。
  - 挑战：大模型体积（语言包 > 10MB）、移动端推理速度。

- **语音识别**：
  - Whisper.cpp → Wasm：OpenAI Whisper 模型压缩后可在浏览器运行。
  - 应用场景：离线语音输入、会议实时字幕。

### 3. 大模型前端部署

将 LLM 运行在浏览器中，是最近一年最热门的方向之一。

- **WebLLM**：
  - 基于 Apache TVM 的 MLCEngine，将 LLM 编译到 WebGPU / WebGL。
  - 典型部署：Llama-2-7B（INT4 量化 → ~4GB），在 RTX 4090 上可达 30-50 token/s。
  - 架构：模型权重存 IndexedDB（首次加载后缓存），推理时 GPU 加速。

- **Wasm LLM**：
  - llama.cpp 编译到 Wasm（利用 Wasm SIMD）。
  - 纯 CPU 推理，速度远低于 WebGPU 但兼容性更好。
  - 典型性能：7B INT4 模型，M2 Max 上约 15-20 token/s。

- **流式分词（Streaming Tokenization）**：
  - LLM 推理是逐 token 生成，通过流式 API（ReadableStream）将每个新 token 实时推送到 UI。
  - 用户体验好的关键：首 token 延迟 < 500ms（Time to First Token），后续 token 间隔 < 50ms。

- **内存分片加载**：
  - 模型权重不可能全放内存，需要类似 mmap 的方式从 IndexedDB 按需加载。
  - 每次推理只需当前层的权重 → 加载当前层 → 释放 → 加载下一层。

### 4. AI 渲染增强

将 AI 模型与渲染管线结合，实现画质增强。

- **超分辨率（Super Resolution）**：
  - 低分辨率图像/视频 → AI 放大 → 高分辨率。
  - ESRGAN 等模型可压缩部署到浏览器（ONNX Runtime Web）。
  - 应用：浏览器端图片无损放大、视频画质增强。

- **实时美颜与背景虚化**：
  - 人脸检测 + 皮肤分割 + 磨皮/美白/瘦脸算法。
  - 虚化（Bokeh）：语义分割出主体 + 高斯模糊背景 = 单反效果。
  - 关键：美颜需要实时处理视频流（30fps），每帧推理时间 < 10ms。

- **AI 抠图（Matting）**：
  - 不是简单的二值分割（前/背景），而是计算 alpha 通道（透明度）。
  - 可以精细处理头发丝、半透明物体边缘。

### 5. 性能与功耗平衡

移动端浏览器 AI 推理的最大限制是发热和续航。

- **GPU 功耗**：
  - WebGPU 计算着色器满载运行 → 设备发热 → 性能降频（thermal throttling）→ 帧率断崖下跌。
  - 解决方案：帧率降级（30fps → 15fps → 5fps）、模型切换（大模型 → 小型化模型）、间歇推理。

- **后台限制**：
  - 浏览器 Tab 后台时，`requestAnimationFrame` 被暂停、timer 被降频。
  - Service Worker 中可做后台推理但 API 受限（无法访问 WebGPU）。

- **内存限制**：
  - 移动端浏览器 JS 堆通常限制在几百 MB。
  - 大模型加载 + 推理中间张量很容易 OOM。
  - 策略：模型权重常驻 IndexedDB（持久化），推理时加载到 GPU Buffer 用完即释放中间结果。

---

## 深入学习路线

```
入门（1-2月）
├── TensorFlow.js 官方文档通读
├── ONNX Runtime Web 使用
├── 加载预训练模型：MobileNet、COCO-SSD
└── 实战：图像分类 Demo

进阶（3-4月）
├── 模型量化实践：INT8/FP16 压缩与精度评估
├── WebGPU 后端性能调优：shader 选择、内存管理
├── 端侧模型组合：MediaPipe（人脸） + 自定义模型
└── 实战：人脸检测 + 姿态识别应用

深入（6月+）
├── WebLLM 大模型部署：Llama 量化、流式输出
├── AI 渲染管线：超分、美颜、背景虚化的完整实现
├── 自定义模型训练：TensorFlow Lite Model Maker → 浏览器推理
└── 性能极致优化：内存管理、功耗控制、渐进式模型加载
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [TensorFlow.js](https://www.tensorflow.org/js) | Google 浏览器 AI 框架，生态最全 |
| [ONNX Runtime Web](https://onnxruntime.ai/docs/tutorials/web/) | 跨框架模型推理，Wasm/WebGPU 后端 |
| [MediaPipe](https://developers.google.com/mediapipe) | Google 端侧 ML 解决方案，人脸/手势/姿态 |
| [WebLLM](https://webllm.mlc.ai/) | 浏览器内大语言模型推理 |
| [whisper.cpp](https://github.com/ggerganov/whisper.cpp) | OpenAI Whisper 的 C++ 实现，可编译到 Wasm |
| [Tesseract.js](https://tesseract.projectnaptha.com/) | 浏览器 OCR，Wasm 编译 |
| [Transformers.js](https://github.com/xenova/transformers.js) | HuggingFace Transformers 的浏览器版 |
