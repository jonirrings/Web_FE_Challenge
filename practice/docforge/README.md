# DocForge — 智能文档工作台

> 类 Notion + Google Docs，富文本协同编辑 + AI 辅助写作 + 文档格式互转

---

## 核心功能

- 富文本编辑：标题/列表/表格/代码块/嵌入块
- 多人协作：实时光标、冲突合并
- AI 辅助：续写、摘要、语法纠错
- 格式互转：Word/Excel/PDF 导入导出
- 离线编辑：断网可写，联网自动同步
- 安全登录：Passkeys 免密认证

---

## 挑战落地

### 02 CRDT — 富文本协同编辑

- **块级 CRDT**：每个段落/标题/表格行为一个 CRDT Node，块间排序用 RGA 算法
- **行内 CRDT**：块内文本用 YATA 算法（Yjs）实现字符级冲突合并
- **光标同步**：Yjs Awareness 协议广播每个协作者的选区位置
- **离线持久化**：Yjs Update 增量存入 IndexedDB，联网后 push 到服务端

**练习要点**：
- Yjs + y-websocket 集成
- 自定义 Block Type（表格、代码块）的 CRDT 建模
- Undo/Redo 在 CRDT 下的实现（Yjs UndoManager）

### 04 浏览器 AI — 端侧智能

- **文本续写**：WebLLM 加载小型 LLM（如 Phi-3-mini），端侧推理生成续写内容
- **摘要生成**：选中段落 → 端侧推理生成摘要，插入文档
- **语法纠错**：TensorFlow.js 加载语法纠错模型，实时标注错误并建议修正
- **性能功耗**：推理任务放 Web Worker，避免阻塞主线程；检测设备算力决定端侧/云端

**练习要点**：
- WebLLM 集成与模型加载优化
- Web Worker 内运行推理
- WebGPU 后端加速推理
- 降级策略：低端设备 fallback 到云端 API

### 14 文档处理 — 格式互转

- **Word 解析**：前端解析 .docx（mammoth.js / 自研 OOXML Parser），转为内部 Block 模型
- **Excel 解析**：前端解析 .xlsx（SheetJS），表格块直接渲染
- **PDF 渲染**：pdf.js 渲染 PDF 页面，支持文字选择与复制
- **PDF 导出**：内部 Block 模型 → HTML → Puppeteer/端侧生成 PDF
- **图片编解码**：WebAssembly 版 libwebp/turbojpeg 处理图片压缩

**练习要点**：
- OOXML 格式结构与解析
- PDF 渲染管线（pdf.js + Canvas）
- 大文件流式解析（避免一次性加载到内存）

### 15 分布式数据库 — 本地优先

- **本地存储**：IndexedDB 存储文档数据 + CRDT Metadata
- **多端同步**：Yjs Doc 通过 WebSocket 与服务端同步，断网时本地继续写入
- **冲突解决**：CRDT 自动合并，无需手动解决冲突
- **最终一致性**：所有端最终收敛到同一状态，保证交换律/结合律/幂等性

**练习要点**：
- 本地优先（Local-first）架构设计
- IndexedDB 事务与版本管理
- 同步协议设计（状态向量 / 快照 + 增量）

### 22 WebAuthn — 现代认证

- **Passkeys 注册**：浏览器原生支持，用户指纹/Face ID 创建 Passkey
- **Passkeys 登录**：条件 UI（Conditional UI）自动提示可用 Passkey
- **多因子认证**：Passkey + TOTP 组合，高安全场景双因子
- **安全模型**：FIDO2/CTAP2 协议，私钥永远不出设备

**练习要点**：
- WebAuthn API（navigator.credentials.create / get）
- 依赖方（Relying Party）服务端实现
- Conditional UI 自动填充体验
- 跨设备 Passkey 同步（iCloud Keychain / Google Password Manager）

### 20 国际化 — 多语言文档

- **文档翻译**：选中段落 → AI 翻译 → 插入翻译结果
- **CJK 排版**：中文竖排、日文混排、韩文断词规则
- **RTL 混排**：阿拉伯语/希伯来语从右到左排版，LTR 代码块内嵌
- **动态加载**：语言包按需加载，首屏不加载全部翻译

**练习要点**：
- Intl API（Segmenter、ListFormat、DisplayNames）
- CSS `writing-mode: vertical-rl` 竖排
- `dir="auto"` 双向文本自动检测
- i18n 框架集成（react-intl / i18next）

---

## 技术栈建议

```
前端框架：React 18+
编辑器：ProseMirror + Yjs（y-prosemirror）
AI 推理：WebLLM + TensorFlow.js
文档解析：mammoth.js / SheetJS / pdf.js
本地存储：IndexedDB（Dexie.js）
认证：WebAuthn API + @simplewebauthn/server
构建工具：Vite
```
