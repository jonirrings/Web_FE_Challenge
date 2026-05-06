# 二、CRDT 分布式协同编辑

CRDT（Conflict-free Replicated Data Type，无冲突复制数据类型）是多端协同编辑的核心算法基础。与传统通过加锁（悲观锁/乐观锁）实现的协作不同，CRDT 基于**最终一致性**理论，天然支持离线编辑、自动合并冲突、无需中心化协调。

---

## 核心挑战

### 1. 冲突无锁自动合并

协同编辑最核心的问题：两个人同时改同一个地方，如何保证不乱？

- **悲观锁**：修改前先锁住区域。问题：锁粒度的选择（字符级太细、段落级太粗），且锁住后其他人无法编辑该区域，体验差。
- **乐观锁/OT（Operational Transformation）**：Google Docs 早期方案。操作先执行再检查冲突，通过变换操作来合并。需要中心服务器做冲突裁决，不支持离线编辑。
- **CRDT**：每个字符/节点有全局唯一 ID（逻辑时钟 + 客户端 ID），插入和删除操作可以**任意顺序到达**各端，通过算法保证最终状态一致。天然支持离线编辑和多播。

CRDT 的核心数学保证：
- **交换律**：操作顺序不影响最终结果（强最终一致性）
- **幂等律**：重复操作不会导致重复结果

### 2. CRDT 算法选型与数据结构

CRDT 不是单一算法，而是一族数据结构。不同结构适配不同场景。

- **文本序列 CRDT**：
  - **YATA（Yjs 使用）**：为每个字符分配逻辑时钟，通过"左右锚点"确定插入位置。YATA 在字符级协同上表现优秀，是当前社区最活跃的方案。
  - **RGA（Automerge 早期使用）**：基于树形结构的复制增长数组，插入位置由前后节点的全局 ID 决定。合并逻辑比 YATA 复杂。
  - **Fugue（Atom Teletype 使用）**：类似 RGA 但使用 tombstone 压缩优化。

- **结构化数据 CRDT**：
  - **JSON-CRDT（Yjs Y.Map/Y.Array）**：将 CRDT 思想应用于 JSON 树——Map 用 LWW-Register（Last-Write-Wins 寄存器），Array 用类似 YATA 的序列 CRDT。
  - **Automerge（Peritext）**：将文本当作结构树，每个字符是带属性的节点。优点是可以把格式标记（粗体/斜体）直接内嵌在 CRDT 中，缺点是比 Yjs 更重。

- **不是按"字符级/节点级/样式级"分级**，而是按底层数据结构设计来区分：Counter（PN-Counter）、Register（LWW/MV）、Set（G-Set/2P-Set/OR-Set）、Sequence（YATA/RGA）。

### 3. 富文本结构复杂兼容

富文本不是简单字符串，而是带有丰富嵌套结构的树。

- **图文混排**：图片节点在文档中占一个位置但有不同的渲染尺寸和 aspect ratio。图片的 CRDT 操作 = 插入占位符 + 附件数据同步（通常不通过 CRDT，而是对象存储 URL）。
- **表格**：表格有二维结构——行、列、单元格。插入一行不仅仅是插入一个字符，而是对整个表格结构的 CRDT 操作。
- **嵌套块级结构**：如 Notion 的 Block 嵌套（页面 → 列表 → 列表项 → 段落），需要树形 CRDT。Yjs 通过 Y.Map 嵌套 Y.Array/Y.Text 来表达。
- **方案**：ProseMirror（富文本编辑框架）+ Yjs 绑定（y-prosemirror）是当前社区最成熟的富文本 CRDT 方案。ProseMirror 负责编辑体验，Yjs 负责 CRDT 同步。

### 4. 光标位置精准同步

光标同步的难点在于：本地的光标/选区在其他用户的编辑操作下**如何不失位**。

- **相对定位**：光标存储为相对于 CRDT 文档中某个节点的位置偏移。当其他用户在你光标前插入内容时，你的光标会自动向后推移。
- **多端光标渲染**：非选中态的光标（remote cursor）需要在 DOM 中渲染——通常通过绝对定位的 `<div>` 叠加在文本上。高并发下多人光标频繁更新，需要用 RAF 限频。
- **选区同步**：选区比单光标复杂，包含起始锚点 + 结束焦点。需要同步选区的方向（正向/反向选择）。
- **不抖动**：关键在于光标更新频率控制在 30-60fps，在 RAF 中批量更新多个 remote cursor 的 DOM 位置。

### 5. 离线持久化 + 增量同步

- **离线缓存**：使用 IndexedDB（y-indexeddb 适配器）持久化 CRDT 文档。用户离线时所有编辑操作在本地 CRDT 文档上正常执行（CRDT 不依赖网络），操作历史被追加到本地存储。
- **增量同步**：重连时只同步**离线期间的增量操作**（Yjs 的 `encodeStateAsUpdate` 可以获取从某时钟之后的增量），不是全量文档。这避免了全网同步刷屏。
- **断点续连**：重连时的状态对比——本地版本 vs 服务端版本，获取双方各自的增量并双向合并。
- **关键指标**：百万字符文档，增量更新包应在 KB 级别（不是重新发送整个文档），同步延迟 < 500ms 完成合并。

### 6. 版本回溯与时光机

- **操作日志**：CRDT 中每次操作都带有逻辑时钟。Yjs 的 `Y.UndoManager` 通过逆向操作（插入→删除，删除→恢复）来实现撤销/重做。
- **全局撤销**：多人协作下，"撤销"到底撤销谁的？是本地上次操作？还是全局最近一次操作？Y.UndoManager 默认跟踪**当前本地用户**的操作序列。
- **历史快照**：定期保存文档快照（如每 500 次操作），配合增量日志实现任意版本回溯。快照间隔 = 回放计算量 vs 存储成本的权衡。
- **操作回放**：回放不是重新执行 CRDT 操作（那会导致重复），而是从快照 + 增量日志重建目标时刻的状态。

### 7. 大文档性能瓶颈

CRDT 在大文档下性能衰减的核心原因：

- **Yjs 百万字符级别会明显变慢**：不是因为 CRDT 算法本身 O(n)，而是因为：
  - **内存占用**：每个字符的 CRDT 元数据（ID + 位置信息）可达数十字节，百万字符 = 几十 MB 元数据。
  - **编码开销**：`encodeStateAsUpdate` 在百万字符级可能需要数百 ms 来序列化。
  - **Json 序列化**：`JSON.stringify(yDoc.toJSON())` 在大量嵌套结构下的开销。
- **不仅与字数相关，还与操作类型和并发度相关**：
  - 连续插入（一个人快速打字）性能优于随机位置的多人并发插入。
  - 嵌套结构（Map 深层嵌套）的操作比纯文本操作昂贵得多。
- **优化手段**：
  - **二进制编码**：Yjs 的 `encodeStateAsUpdate` 使用二进制编码而非 JSON，体积和速度都优于 JSON。
  - **分片加载**：大文档拆成多个 Y.Doc 子文档，按需加载（类似 Notion 的分页策略）。
  - **GC 回收墓碑**：删除操作产生的 tombstone 可通过 GC 清理（但有风险：离线用户回来后可能无法正确合并）。

---

## 深入学习路线

```
入门（1-2月）
├── CRDT 基础理论：理解强最终一致性、操作交换律
├── Yjs 官方文档与 Demo：理解 Y.Text、Y.Map、Y.Array
├── 区分 CRDT vs OT（Operational Transformation）
└── 实现简单文本协同：两个浏览器 Tab 实时同步

进阶（3-4月）
├── 深入 YATA 算法原理：逻辑时钟、锚定插入、删除墓碑
├── 富文本 CRDT 绑定：ProseMirror + y-prosemirror
├── 光标同步原理：相对定位、remote cursor 渲染
├── 离线存储集成：y-indexeddb 适配器
└── 实战：多人协作文档编辑器

深入（6月+）
├── 大文档优化：二进制编码、分片加载、GC 调优
├── 版本历史与 Undo Manager 源码理解
├── 自研 CRDT：为特定数据结构（表格/画布）设计 CRDT
├── 阅读 Yjs、Automerge 源码核心机制
└── 参与社区：yshape、y-py 等周边生态
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [Yjs](https://github.com/yjs/yjs) | 最成熟的 CRDT 实现，生态丰富，性能优秀 |
| [Automerge](https://automerge.org/) | Rust/JS CRDT，强调快照和历史，适合文档版本管理 |
| [ProseMirror](https://prosemirror.net/) | 富文本编辑框架，与 Yjs 深度绑定 |
| [y-prosemirror](https://github.com/yjs/y-prosemirror) | ProseMirror + Yjs 绑定 |
| [Liveblocks](https://liveblocks.io/) | 协作托管服务，内置 CRDT + 光标同步 + 权限 |
| [PartyKit](https://www.partykit.io/) | 实时协作基础设施，Cloudflare Workers 生态 |
| [y-indexeddb](https://github.com/yjs/y-indexeddb) | Yjs 的 IndexedDB 持久化适配器 |
