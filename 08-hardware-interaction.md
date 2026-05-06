# 八、浏览器硬件交互能力

浏览器正逐渐突破"纯软件"边界，通过一系列硬件 API 直接与物理世界交互。这类 API 极大扩展了 Web 应用的使用场景——从工业物联网到 AR/VR 体验，前端不再只是网页，而是**物理设备的控制面板**。

---

## 核心挑战

### 1. 蓝牙 Web Bluetooth API

Web Bluetooth 让浏览器可以直接与身边的蓝牙设备通信。

- **蓝牙扫描与连接**：
  - `navigator.bluetooth.requestDevice({ filters: [{ services: ['battery_service'] }] })` 触发浏览器原生蓝牙选择器。
  - 用户必须主动点击触发（需要用户手势），防止恶意静默扫描。
  - 扫描结果只返回匹配 filter 的设备，且设备名称可能被截断或为空。

- **GATT 通信**：
  - 连接后通过 GATT（Generic Attribute Profile）与设备交换数据。
  - Service → Characteristic → Value：层级结构。需要阅读设备文档理解其 GATT Profile。
  - 写（Write）/ 读（Read）/ 通知（Notify）：`characteristic.writeValue(buffer)` / `characteristic.readValue()` / `characteristic.addEventListener('characteristicvaluechanged', ...)`。

- **串口透传（Serial over Bluetooth）**：
  - 很多工业蓝牙设备实质是 BLE 虚拟串口（UART Service）。
  - RX 和 TX 是两个 Characteristic，一端写 TX → 另一端从 RX 读。

- **兼容性痛点**：
  - Chrome（桌面 + Android）支持最完整，Safari/iOS 不支持 Web Bluetooth。
  - BLE 连接距离限制（通常 < 10 米），断开后需要自动重连。
  - 同一设备同时只能被一个浏览器 Tab 连接。

### 2. 串口 Web Serial API

Web Serial 将串口通信带入浏览器，工业自动化和硬件调试的前端化成为可能。

- **基本流程**：
  - `navigator.serial.requestPort()` → 用户选择串口设备。
  - `port.open({ baudRate: 9600 })` → 打开串口并设置波特率。
  - `port.readable.getReader()` / `port.writable.getWriter()` → 双向通信。

- **串口参数配置**：
  - 波特率（baudRate）、数据位（dataBits）、停止位（stopBits）、校验位（parity）、流控制（flowControl）。
  - 不同设备需要不同的参数组合，需要查阅设备说明。

- **常见设备集成**：
  - 扫码枪：扫描条码 → 串口输入 → 浏览器接收 → 自动填充表单。
  - 单片机（Arduino / ESP32）：浏览器直接与单片机双向通信，下发指令、读取传感器数据。
  - 工业仪器：温湿度计、电子秤、RFID 读卡器通过串口接入。

- **安全限制**：
  - 仅 HTTPS/localhost 可用。
  - 必须有用户手势触发 `requestPort()`。
  - 切换 Tab 后串口可能被浏览器自动关闭。

### 3. USB WebUSB API

WebUSB 允许浏览器直接与自定义 USB 设备通信，跳过了串口转换的中间层。

- **设备发现与授权**：
  - `navigator.usb.requestDevice({ filters: [{ vendorId: 0x1234, productId: 0x5678 }] })`。
  - 需要知道设备的 USB VID/PID（Vendor ID / Product ID）。

- **端点通信**：
  - USB 设备通过端点（Endpoint）收发数据：IN 端点（设备→主机）、OUT 端点（主机→设备）。
  - `device.transferIn(endpointNumber, length)` / `device.transferOut(endpointNumber, data)`。

- **设备固件协议设计**：
  - WebUSB 设备需要自定义固件（如 Arduino 代码中声明 WebUSB 兼容）。
  - 协议设计：自定义命令格式、数据包结构、错误处理。

- **局限**：
  - 仅 Chrome 支持（2024 年状态），Safari/Firefox 尚未支持。
  - USB 设备需要专为 WebUSB 设计固件（普通 USB 设备如鼠标/键盘无法直接使用 WebUSB）。

### 4. NFC / 传感器 / 定位

- **Web NFC**：
  - `navigator.nfc` 或通过 `NDEFReader` 读写 NFC 标签（NDEF 格式）。
  - 应用：签到打卡、资产盘点、NFC 名片的浏览器读取。
  - 限制：仅 Android Chrome 支持，且需用户主动触发。

- **地理位置（Geolocation API）**：
  - `navigator.geolocation.watchPosition()` 持续追踪位置。
  - 高精度模式（`enableHighAccuracy: true`）使用 GPS 而非 IP 定位。
  - 纠偏：WGS-84（GPS 原始坐标）→ GCJ-02（国测局坐标）→ BD-09（百度坐标），不同地图服务使用不同坐标系。

- **陀螺仪/加速度计（DeviceOrientation / DeviceMotion）**：
  - `deviceorientation` 事件：alpha/yaw（Z 轴旋转）、beta/pitch（X 轴旋转）、gamma/roll（Y 轴旋转）。
  - `devicemotion` 事件：加速度（含重力 × 不含重力）、旋转速率。
  - **iOS 13+ 必须用户主动授权**（`DeviceOrientationEvent.requestPermission()`），Android 无需。
  - 数据噪声处理：使用卡尔曼滤波器或互补滤波器平滑传感器数据。

### 5. WebXR AR/VR

WebXR 将虚拟现实和增强现实带入浏览器，无需安装 App。

- **会话类型**：
  - `immersive-vr`：完全沉浸式 VR（需要 VR 头显如 Quest、Vision Pro）。
  - `immersive-ar`：增强现实（手机 AR，如 iOS ARKit / Android ARCore 底层支持）。
  - `inline`：在页面内渲染 3D 内容（非沉浸式）。

- **空间追踪**：
  - 6DoF（六自由度）追踪：头部位置 + 旋转。
  - 空间锚点（Anchor）：将虚拟物体固定在现实世界的某个位置，移动手机后物体留在原位。

- **手势识别**：
  - VR 手柄：追踪按钮、摇杆、触控板输入。
  - 裸手追踪：Leap Motion / Quest 裸手 → 手指关节位置。

- **双目渲染**：
  - VR 需要分别为左眼和右眼渲染不同的画面（立体视差）。
  - `XRWebGLLayer` 的 `framebuffer` 是双眼共用的宽画布，需要分别渲染到左右半区。

- **性能要求**：
  - VR：至少 72fps（Quest 2 最高 120fps），每帧渲染时间 < 13.8ms。
  - AR：60fps，需要将虚拟内容稳定锚定在现实场景上（位置漂移 < 1mm）。

---

## 深入学习路线

```
入门（1-2月）
├── Web Bluetooth API：心率带/温度计 Demo
├── Web Serial API：连接 Arduino 读取传感器
├── 设备选型与采购：Arduino 开发板 + BLE 模块
└── 最小硬件交互 Demo

进阶（3-4月）
├── WebUSB 协议设计：自定义固件 + 浏览器驱动
├── WebXR 入门：immersive-vr 会话 + Three.js 3D 内容
├── 设备固件开发：Arduino/ESP32 固件编写
└── 实战：扫码枪集成、打印机票据打印

深入（6月+）
├── Web NFC NDEF 解析与安全
├── WebXR 手势识别：裸手交互 + 空间锚点
├── 工业协议移植：Modbus RTU/TCP、OPC-UA 浏览器解析
├── 自定义 HID 设备：WebHID API（Chrome 实验）
└── 硬件驱动级调试：USB 协议分析仪、逻辑分析仪
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API) | MDN 蓝牙 API 文档 |
| [Web Serial API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Serial_API) | MDN 串口 API 文档 |
| [WebUSB API](https://developer.mozilla.org/en-US/docs/Web/API/WebUSB_API) | MDN USB API 文档 |
| [Arduino WebSerial](https://github.com/UncaughtTypeError/Arduino-WebSerial) | Arduino 的 Web Serial 适配库 |
| [WebXR Device API](https://immersiveweb.dev/) | 沉浸式 Web 标准与教程 |
| [Three.js WebXR](https://threejs.org/docs/#manual/en/introduction/How-to-create-VR-content) | Three.js 的 WebXR 支持 |
| [WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API) | 短信验证码自动填充 API |
