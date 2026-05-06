# 十四、网页文档处理引擎

文档处理引擎的目标是：在浏览器中完整呈现 Office 文档、PDF、图片和音视频，无需安装任何桌面软件。核心挑战在于——这些格式本身就是几十年的工业标准，复杂度和兼容性远超普通网页。

---

## 核心挑战

### 1. 前端在线 Office 解析

Word/Excel 文件的格式复杂度远超"一段 HTML"。

- **Docx 解析**：
  - DOCX 本质是一个 ZIP 包，内含 XML 文件（`document.xml` 主文档、`styles.xml` 样式、`numbering.xml` 列表编号等）。
  - 需要解析 OOXML（Office Open XML）标准：段落、run（文字段）、样式、表格、图片、页眉页脚、脚注、批注。
  - 关键库：mammoth.js（专注于语义化 HTML 输出）、docx.js（低层级操作）。
  - 复杂排版还原的难点：分页、分栏、环绕排版、文本框、公式（MathML/OMML）、修订标记。

- **Excel 解析**：
  - XLSX 也是 ZIP + XML 结构，但包含：单元格值（数字/字符串/日期/公式）、样式（颜色/边框/字体）、合并单元格、数据透视表、图表、条件格式。
  - SheetJS（xlsx）是当前最成熟的前端 Excel 库，支持读/写 XLSX。
  - 公式计算引擎：前端需要实现 Excel 的函数库（SUM/IF/VLOOKUP/INDEX-MATCH），`formula.js` 提供了大多数函数的 JS 实现。
  - 表格透视（Pivot Table）：动态分组、聚合、多维分析——需要 OLAP 计算引擎。

- **渲染方案**：
  - Canvas 渲染：将文档内容逐页绘制到 Canvas。优势是还原度最高、可控性最强；劣势是文本不可选中、无障碍差。
  - DOM 渲染：将文档结构映射为 HTML DOM。优势是原生文本选择和搜索；劣势是复杂排版还原难（尤其是 Word 的分页和环绕）。

### 2. PDF 前端渲染编辑

- **PDF 渲染**：
  - PDF.js（Mozilla）：将 PDF 的页面描述转换为 Canvas 逐页渲染。
  - 分页解析：PDF 不是流式文档，每页独立。可以只加载当前页 → 分片加载。
  - 文字层：PDF.js 不仅渲染图像，还可以提取文字层（Text Layer）覆盖在 Canvas 上 → 支持文本选择和搜索。

- **批注与签名**：
  - PDF 批注：高亮、下划线、文本注释、图形批注（矩形/椭圆/箭头）→ 这些需要额外的 SVG/Canvas 层叠加。
  - 电子签名：Canvas 手写签名 → 转为图像 → 嵌入 PDF 页面特定位置 → 导出新的 PDF（服务端或前端完成）。
  - PDF-lib：前端库，支持修改和生成 PDF（添加文字、图片、表单）。

- **大 PDF 分片加载**：
  - 几百页的 PDF 不能一次性加载。需要 Range Requests（HTTP Range Header）下载指定字节范围。
  - PDF 尾部有交叉引用表（XRef），先加载尾部 → 获取页面偏移 → 按需下载页面内容。

### 3. 图片音视频前端编解码

浏览器不再只是"播放媒体"，而是能"处理媒体"。

- **图片处理**：
  - Canvas 的 `drawImage` + `toBlob` / `toDataURL` 提供基础图片处理能力。
  - 格式转换：PNG ↔ JPEG ↔ WebP ↔ AVIF，通过 Canvas 渲染后重新编码。
  - 水印合成：Canvas 上绘制原图 + 叠加文字/Logo → 导出。
  - 大图处理：超大图片（如卫星照片 >10000px）需要分片加载和处理。

- **视频剪辑**：
  - WebCodecs API（Chrome/Edge/Safari）：提供音视频编解码器的底层访问，帧级操作。
  - 帧提取：`VideoDecoder` 解码视频 → 获取 `VideoFrame` → 绘制到 Canvas → 导出为图片。
  - 格式转码：`VideoEncoder` + `VideoDecoder` 组合实现格式转换。
  - ffmpeg.wasm：FFmpeg 编译到 Wasm，功能完整的视频处理。缺点是体积大（~30MB），首次加载慢。

- **音频处理**：
  - Web Audio API：AudioContext → AudioBuffer 获取 PCM 数据 → AnalyzeNode 频谱分析 → 自定义处理。
  - AudioWorklet：在独立线程中处理音频流，延迟低（< 2ms），适合实时音频效果。

---

## 深入学习路线

```
入门（1-2月）
├── PDF.js 渲染流程：理解 PDF 页面 → Canvas
├── Canvas 2D API：图片绘制、转换、导出
├── 常见格式认识：PNG/JPEG/WebP/AVIF 特点与适用场景
└── 最小文档预览 Demo

进阶（3-4月）
├── Docx 解析：ZIP 解压 + XML 解析 + 样式映射
├── Excel 解析：SheetJS + 公式计算引擎
├── WebCodecs API：视频编解码帧级操作
├── Web Audio API：AudioContext + AudioWorklet
└── 实战：文档在线预览平台

深入（6月+）
├── 复杂排版还原：分页、分栏、公式、批注
├── 大文件分片加载：HTTP Range + 流式处理
├── ffmpeg.wasm：视频转码/剪辑/合并
└── 文档协同编辑集成：CRDT + 文档渲染
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [PDF.js](https://mozilla.github.io/pdf.js/) | Mozilla PDF 渲染引擎 |
| [pdf-lib](https://www.pdf-lib.com/) | PDF 创建与修改 |
| [SheetJS](https://sheetjs.com/) | Excel (XLSX) 读写库 |
| [mammoth.js](https://github.com/mwilliamson/mammoth.js) | Docx → HTML 转换 |
| [docx.js](https://docx.js.org/) | 前端生成 Docx |
| [formula.js](https://github.com/handsontable/formula.js) | Excel 公式引擎 (JS) |
| [WebCodecs API](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API) | 浏览器原生编解码 |
| [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm) | FFmpeg 的浏览器版本 |
