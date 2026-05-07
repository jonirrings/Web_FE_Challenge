# 十五、前端分布式数据库

前端分布式数据库将数据库能力从服务端搬到浏览器端——端侧存储、离线查询、多设备自动同步。它不是 IndexedDB 的简单封装，而是将 **CouchDB 风格的分布式同步协议**带到浏览器中。

---

## 核心挑战

### 1. 端侧数据库引擎

浏览器端的"数据库"不是真正的 SQL 引擎，而是结构化文档存储。

- **PouchDB**：
  - 兼容 CouchDB 协议的浏览器端数据库，默认用 IndexedDB 存储。
  - 核心能力：文档存储（JSON）、增量同步（Replication）、冲突检测（基于 rev 版本号）。
  - 支持 MapReduce 查询（通过 Map 函数定义索引，Reduce 做聚合）。

- **RxDB**：
  - 基于 RxJS 的响应式数据库，底层可用 PouchDB / LokiJS / Dexie 作为存储引擎。
  - 特色：Observable 查询（数据变化自动推送）、Schema 定义（基于 jsonschema）、加密插件、GraphQL 复制。
  - 更偏"实时应用"场景——状态变化自动反映到 UI。

- **Dexie.js**：
  - IndexedDB 的 Promise 封装，API 简洁优雅。
  - 适合：数据量大但同步需求相对简单的应用。
  - `dexie-cloud` 提供云同步功能（商业服务）。

- **SQLite Wasm**：
  - SQLite 编译到 Wasm，在浏览器中运行完整的关系型数据库。
  - 支持完整 SQL（SELECT/JOIN/GROUP BY/子查询），真正的 ACID 事务。
  - 存储方案：文件系统持久化到 Origin Private File System（OPFS），性能接近本地 SQLite。

### 2. 多端数据同步

多设备同步是前端数据库最难的部分。

- **增量同步（Replication）**：
  - 不是"每次同步全量数据"，而是同步变更集（Change Set）。
  - CouchDB 协议：每个文档有 `_rev` 版本号。同步时比较本地版本号 vs 远程版本号 → 只传输变化文档。
  - 检查点（Checkpoint）：上次同步到哪里了（CouchDB 用序列号 `since`），下次从检查点之后开始同步。

- **冲突检测与解决策略**：
  - 多端同时修改同一文档 → 同步时产生冲突。
  - **自动策略**：
    - Last-Write-Wins（LWW）：基于时间戳，新覆盖旧。简单但丢数据。
    - 版本树（DAG）：保留所有冲突版本，等待手动或自动合并。
  - **自动合并**：
    - 对文本字段可以用 CRDT（如 `Y.Text`）做自动合并。
    - 对结构化字段用三路合并（3-way merge）：原值 v0、本地修改 v1、远程修改 v2 → 合并。
  - **手动解决**：
    - 当自动合并不可能时，标记文档为"冲突状态" → 用户选保留哪版或手动编辑。

- **事务与一致性**：
  - IndexedDB 的事务是自动提交的（不能手动 commit/rollback）。
  - 需要手动实现"写入前检查 + 写入后验证"的逻辑。
  - CAP 定理权衡：网络分区时，选择可用性（继续本地写，后面合并）还是一致性（拒绝写）。

### 3. 离线优先架构

离线不是"没网时凑合用"，而是"没网是常态，有网是优化"。

- **Offline-First 原则**：
  - 所有操作默认在本地执行（本地数据库优先），后台异步同步到服务端。
  - 用户感知到的延迟 = IndexedDB 读写延迟（< 10ms），而非网络延迟。

- **CRUD 离线化**：
  - Create：线下创建文档，生成临时 ID → 同步时服务端分配永久 ID → 本地更新 ID 映射。
  - Update：线下修改文档 → 记录修改内容 → 同步时按顺序回放。
  - Delete：线下软删除（标记 deleted 而非真删除）→ 同步时服务端决定是硬删除还是透传。

- **同步状态 UI**：
  - 显示当前同步状态：已同步 ✓ / 等待同步 ↻ / 同步失败 ⚠。
  - 未同步的数据标记提示（如笔记列表中的"未同步"徽标）。

---

## 深入学习路线

```
入门（1-2月）
├── IndexedDB 深度使用：事务、索引、容量管理
├── PouchDB / RxDB 入门：文档存储 + 基础查询
├── 理解 CouchDB 同步协议：rev 版本号 + 变更集
└── 最小离线 Todo 应用

进阶（3-4月）
├── 增量同步：Replication 原理 + 检查点机制
├── 冲突检测与解决：版本树 + 自动合并 + 手动解决
├── 事务设计：本地 ACID 操作 + 同步时冲突处理
└── 实战：离线笔记应用（多 Tab 同步 + 跨设备同步）

深入（6月+）
├── CRDT 在前端数据库的应用：RxDB + Yjs 集成
├── 大规模数据存储优化：分片、压缩、增量备份
├── 跨设备数据同步架构：WebSocket + P2P（WebRTC Data Channel）
└── 参与开源社区：RxDB 插件开发、PouchDB 贡献
```

---

## 关键技术与开源项目

| 项目 | 用途 |
|------|------|
| [PouchDB](https://pouchdb.com/) | CouchDB 兼容的浏览器数据库 |
| [RxDB](https://rxdb.info/) | 响应式实时数据库，支持多存储后端 |
| [Dexie.js](https://dexie.org/) | IndexedDB 包装库，API 优雅 |
| [SQLite Wasm](https://sqlite.org/wasm) | 浏览器中的完整 SQLite |
| [LokiJS](https://github.com/techfort/LokiJS) | 内存优先的 JS 数据库 |
| [CouchDB](https://couchdb.apache.org/) | 分布式文档数据库（服务端），PouchDB 的天然搭档 |
| [Replicache](https://replicache.dev/) | 端侧可缓存的应用数据库（商业） |
