# 块级编辑器架构：类 Notion 的 Block Editor

**状态**：架构设计完成
**日期**：2026-06-20
**核心哲学**：块是原子，CRDT 是灵魂，Yjs 是肌肉

> 一个类 Notion 的块级编辑器，核心挑战不在"渲染"，而在"多人实时协同编辑同一棵树"。本文档给出从数据结构到前后端的全套方案。

---

## 1. 核心问题拆解

| 问题 | 解法 |
|------|------|
| 块怎么存？ | 树状结构，每块有唯一 ID，父子关系通过 `children: Vec<BlockId>` 维护 |
| 多人同时编辑怎么不冲突？ | CRDT（Yjs），自动合并，无需 OT 服务器 |
| 前端怎么渲染？ | React/Vue 组件映射，每种块类型对应一个组件 |
| 后端怎么存？ | Fjall 存 Yjs 二进制文档 + 增量更新日志 |
| 块怎么排序？ | Yjs Y.Array 保证顺序一致性 |

---

## 2. 数据结构设计

### 2.1 Block 枚举（Rust）

```rust
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

/// 块的唯一标识
pub type BlockId = String;

/// 块类型枚举 — 使用默认 Externally Tagged 模式
/// 兼容 Postcard/Bincode（不触发 deserialize_any）
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub enum BlockType {
    Paragraph,
    Heading { level: u8 },           // 1-6
    BulletList,
    NumberedList,
    Todo { checked: bool },
    Code { language: Option<String> },
    Image { url: String, caption: Option<String> },
    Table { rows: Vec<Vec<String>> },
    Callout { icon: Option<String> },
    Divider,
    Toggle { title: String },
    Embed { url: String },
    Quote,
    Column { ratio: f32 },           // 布局列
}

/// 单个块
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct Block {
    pub id: BlockId,
    pub block_type: BlockType,
    pub content: Option<String>,      // 富文本（Markdown 或 JSON Rich Text）
    pub children: Vec<BlockId>,       // 子块 ID 列表（有序）
    pub metadata: HashMap<String, serde_json::Value>,  // 扩展字段
}

/// 文档 = 块的集合 + 根块顺序
#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct Document {
    pub id: String,
    pub title: String,
    pub root_block_ids: Vec<BlockId>,  // 顶层块的顺序
    pub blocks: HashMap<BlockId, Block>,
    pub version: u64,
}
```

### 2.2 为什么用 HashMap + Vec 而不是嵌套树？

```
❌ 嵌套树（Notion 的物理存储）：
Block {
    children: Vec<Block>,  // 嵌套深了就是性能地狱
}

✅ 扁平化 HashMap（推荐）：
Document {
    blocks: HashMap<BlockId, Block>,  // O(1) 查找
    root_block_ids: Vec<BlockId>,     // 顶层顺序
}
```

**优势**：
- **O(1) 随机访问**：通过 ID 直接定位任何块，不需要遍历树
- **移动块零拷贝**：从父 A 的 children 移到父 B 的 children，只需改 ID 引用
- **CRDT 友好**：Yjs 的 Y.Map 天然对应 HashMap，Y.Array 天然对应 Vec
- **序列化简单**：Postcard/CBOR 直接序列化整个 HashMap，不需要递归

### 2.3 富文本模型（Inline Content）

块的 `content` 字段存储行内内容。Notion 用 JSON 表示富文本，我们用更紧凑的方案：

```rust
/// 行内内容片段
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct InlineSpan {
    pub text: String,
    pub marks: Vec<Mark>,  // 加粗、斜体、链接等
}

#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub enum Mark {
    Bold,
    Italic,
    Code,
    Link { url: String },
    Highlight,
    Strikethrough,
}

/// 富文本 = 多个片段
pub type RichText = Vec<InlineSpan>;
```

**为什么不用 Yjs Y.Text 直接存富文本？**
- Yjs Y.Text 是 CRDT 文本，适合协同编辑场景
- 但存储时需要序列化，用 `RichText` 更紧凑
- 协同编辑时，前端用 Y.Text，后端存储时转为 `RichText`

---

## 3. Yjs CRDT 协同架构

### 3.1 为什么选 Yjs？

| 维度 | Yjs | Automerge | ShareDB (OT) |
|------|-----|-----------|--------------|
| 性能 | **极快**（C 实现 WASM） | 较慢（纯 JS） | 需要中心服务器 |
| 文档大小 | **极小**（增量编码） | 较大 | 中等 |
| 离线支持 | **完美** | 支持 | 不支持 |
| 生态 | **最成熟**（ProseMirror, TipTap） | 成长中 | 老旧 |
| 内存占用 | **低** | 高 | 中等 |

### 3.2 Yjs 文档结构映射

```
Y.Doc
├── Y.Map "blocks"           ←→ Document.blocks (HashMap)
│   ├── Y.Map "block-uuid-1" ←→ Block { id, type, content, children }
│   ├── Y.Map "block-uuid-2" ←→ ...
│   └── ...
├── Y.Array "root_block_ids" ←→ Document.root_block_ids (Vec)
└── Y.Text "title"           ←→ Document.title
```

### 3.3 Rust 后端：Yrs（Yjs 的 Rust 实现）

```rust
use yrs::{Doc, Map, Array, Text, Transaction, ReadTxn};

/// 创建新文档
pub fn create_document(doc_id: &str) -> Doc {
    let doc = Doc::new();
    let mut txn = doc.transact_mut();

    // 初始化根块列表
    let root_ids = txn.get_array("root_block_ids");
    root_ids.insert(&mut txn, 0, "block-root-001");

    // 初始化块集合
    let blocks = txn.get_map("blocks");
    let mut block = Map::new();
    block.insert(&mut txn, "id", "block-root-001");
    block.insert(&mut txn, "type", "paragraph");
    block.insert(&mut txn, "content", "");
    block.insert(&mut txn, "children", Array::new());
    blocks.insert(&mut txn, "block-root-001", block);

    doc
}

/// 应用增量更新（从 WebSocket 收到的）
pub fn apply_update(doc: &Doc, update: &[u8]) {
    let mut txn = doc.transact_mut();
    doc.apply_update(&mut txn, update);
}

/// 获取二进制快照（存盘用）
pub fn get_snapshot(doc: &Doc) -> Vec<u8> {
    doc.encode_state_as_update_v1(&Default::default())
}
```

### 3.4 协同协议（WebSocket）

```
客户端 A ──┐                    ┌── 客户端 B
           │                    │
     ┌─────▼─────┐        ┌────▼─────┐
     │  Yjs Client │        │ Yjs Client │
     └─────┬─────┘        └────┬─────┘
           │                    │
     ┌─────▼────────────────────▼─────┐
     │        WebSocket Server        │
     │   (Rust: yrs + tokio-tungstenite) │
     └─────────────┬─────────────────┘
                   │
     ┌─────────────▼─────────────────┐
     │         Fjall Storage         │
     │   (Yjs 文档 + 增量更新日志)      │
     └───────────────────────────────┘
```

**消息格式**（使用 Postcard 序列化）：

```rust
#[derive(Serialize, Deserialize)]
pub enum WsMessage {
    /// 客户端发送增量更新
    SyncStep1(Vec<u8>),           // Yjs 编码的增量
    /// 服务端回复状态向量
    SyncStep2(Vec<u8>),           // Yjs 状态向量
    /// 双向增量同步
    Update(Vec<u8>),              // Yjs 增量更新
    /// 光标/选区感知（Awareness）
    Awareness { client_id: u64, state: Vec<u8> },
}
```

---

## 4. 存储架构

### 4.1 Fjall 存储设计

```
Fjall Database
├── "documents"          Keyspace
│   ├── Key: doc_id      Value: Document 元数据（Postcard）
│   └── ...
├── "blocks"             Keyspace
│   ├── Key: (doc_id, block_id)  Value: Block（Postcard）
│   └── ...
└── "yjs_snapshots"      Keyspace
    ├── Key: doc_id      Value: Yjs 完整快照（二进制）
    └── ...
└── "yjs_updates"        Keyspace
    ├── Key: (doc_id, seq)  Value: Yjs 增量更新（二进制）
    └── ...
```

### 4.2 写入策略

```
实时协同流程：
1. 客户端 A 编辑 → 产生 Yjs 增量 update
2. 增量通过 WebSocket 广播给所有客户端
3. 同时写入 Fjall "yjs_updates"（追加日志）
4. 定期（每 5 分钟或 1000 条更新）合并为完整快照，写入 "yjs_snapshots"
5. 合并后清理旧的增量日志
```

### 4.3 查询场景

| 场景 | 查询方式 |
|------|----------|
| 打开文档 | 读取 `yjs_snapshots[doc_id]` → 应用未合并的增量 |
| 渲染所有块 | 读取 `blocks[(doc_id, *)]` → Postcard 反序列化 |
| 搜索块内容 | Fjall 全文索引（或独立的 Tantivy 索引） |
| 导出为 Markdown | 遍历 `root_block_ids` → 递归渲染块树 |
| 版本历史 | 保留关键时间点的 Yjs 快照 |

---

## 5. 前端架构

### 5.1 React 组件映射

```typescript
// Block 类型定义（TypeScript）
interface Block {
  id: string;
  blockType: BlockType;
  content: string | null;
  children: string[];
  metadata: Record<string, any>;
}

// 块组件映射
const BlockRenderer: Record<BlockType['type'], React.FC<BlockProps>> = {
  paragraph: ParagraphBlock,
  heading: HeadingBlock,
  bulletList: BulletListBlock,
  numberedList: NumberedListBlock,
  todo: TodoBlock,
  code: CodeBlock,
  image: ImageBlock,
  table: TableBlock,
  callout: CalloutBlock,
  divider: DividerBlock,
  toggle: ToggleBlock,
  embed: EmbedBlock,
  quote: QuoteBlock,
};

// 主编辑器组件
function BlockEditor({ docId }: { docId: string }) {
  const [doc, setDoc] = useState<Document | null>(null);
  const ydoc = useRef(new Y.Doc());

  useEffect(() => {
    // 连接 WebSocket，加载文档
    const ws = new WebSocket(`ws://localhost:8080/doc/${docId}`);
    ws.onmessage = (e) => {
      const update = new Uint8Array(e.data);
      Y.applyUpdate(ydoc.current, update);
      // 重新渲染
      setDoc(encodeDocument(ydoc.current));
    };
    return () => ws.close();
  }, [docId]);

  if (!doc) return <Loading />;

  return (
    <div className="block-editor">
      {doc.rootBlockIds.map(blockId => (
        <BlockRenderer
          key={blockId}
          block={doc.blocks[blockId]}
          ydoc={ydoc.current}
        />
      ))}
    </div>
  );
}
```

### 5.2 单个块组件示例

```typescript
function ParagraphBlock({ block, ydoc }: BlockProps) {
  const textRef = useRef<Y.Text>();

  useEffect(() => {
    // 绑定 Yjs Y.Text 到 DOM
    const ytext = ydoc.getText(`block:${block.id}:content`);
    textRef.current = ytext;

    // 用 Yjs Binding 自动同步到 contenteditable
    const binding = new QuillBinding(ytext, editor);
    return () => binding.destroy();
  }, [block.id]);

  return (
    <div className="block-paragraph" data-block-id={block.id}>
      <div
        className="block-content"
        contentEditable
        suppressContentEditableWarning
      />
    </div>
  );
}
```

### 5.3 块操作 API

```typescript
// 块操作（通过 Yjs Map 操作，自动同步）
class BlockOperations {
  constructor(private ydoc: Y.Doc) {}

  // 添加块
  addBlock(parentId: string, index: number, type: BlockType) {
    const blocks = this.ydoc.getMap('blocks');
    const newBlock = new Y.Map();
    const id = crypto.randomUUID();

    newBlock.set('id', id);
    newBlock.set('type', type);
    newBlock.set('content', '');
    newBlock.set('children', new Y.Array());

    blocks.set(id, newBlock);

    // 添加到父块的 children
    const parent = blocks.get(parentId) as Y.Map;
    const children = parent.get('children') as Y.Array;
    children.insert(index, [id]);
  }

  // 删除块
  deleteBlock(blockId: string) {
    const blocks = this.ydoc.getMap('blocks');
    const block = blocks.get(blockId) as Y.Map;

    // 递归删除子块
    const children = block.get('children') as Y.Array;
    for (let i = children.length - 1; i >= 0; i--) {
      this.deleteBlock(children.get(i) as string);
    }

    // 从父块的 children 中移除
    // (需要先找到父块，可以通过遍历或维护 parentMap)
    blocks.delete(blockId);
  }

  // 移动块
  moveBlock(blockId: string, newParentId: string, index: number) {
    // 从旧父块移除
    // 插入到新父块
  }
}
```

---

## 6. 完整数据流

```
用户在 Block A 中打字
        │
        ▼
[React contenteditable] ──input──► [Y.Text 对 block:A:content]
        │
        ▼
[Y.Doc 产生增量 update]
        │
        ├──► [WebSocket 广播给其他客户端]
        │         │
        │         ▼
        │    [其他客户端 Y.Doc 合并更新]
        │         │
        │         ▼
        │    [React 重新渲染受影响的块]
        │
        └──► [WebSocket 服务端收到 update]
                  │
                  ▼
             [Fjall 追加写入 yjs_updates]
                  │
                  ▼
             [定期合并为 yjs_snapshots]
```

---

## 7. 与 Aura 生态的集成

| 组件 | Aura 中的对应 | 集成方式 |
|------|--------------|----------|
| 块存储 | Fjall Keyspace | `blocks` keyspace 直接用 Fjall |
| 协同同步 | Openraft | Yjs 处理 CRDT 合并，Raft 处理元数据一致性 |
| 文本搜索 | Tantivy | 块内容索引到 Tantivy，支持全文搜索 |
| 向量搜索 | LanceDB | 块内容 Embedding 存入 LanceDB，支持语义搜索 |
| AI 生成 | Steel Lisp | Lisp 脚本生成 Block JSON → 验证 → 插入文档 |
| 导出 | Arrow IPC | 文档转为 Arrow 列式，支持 Polars 分析 |

---

## 8. 技术选型决策

| 决策点 | 选择 | 理由 |
|--------|------|------|
| CRDT 引擎 | Yjs (yrs) | 性能最优，生态最成熟，ProseMirror/TipTap 验证 |
| 序列化格式 | Postcard (RPC) + CBOR (存储) | 同 Aura 全链路，Externally Tagged 枚举兼容 |
| 存储引擎 | Fjall | 同 Aura 全链路，LSM-Tree 适合追加写入 |
| 前端框架 | React + TipTap | TipTap 是 Yjs 最佳搭档，块编辑器事实标准 |
| WebSocket | tokio-tungstenite | Rust 原生，与 Tokio 生态无缝集成 |
| 富文本 | TipTap (ProseMirror) | 块编辑器事实标准，Yjs 官方 Binding |

---

## 9. 交叉引用

- **[Aura 架构](aura-architecture.md)**：存算一体的现代分布式 Actor 引擎
- **[序列化协议抉择](serialization-protocol-decision.md)**：Postcard + Externally Tagged 枚举
- **[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md)**：Fjall + Arrow + Polars 全链路存算一体
- **[Fluxora 架构](projects/fluxora-architecture.md)**：AI-native UI 框架，块编辑器可作为 Fluxora 的文档编辑组件

**第一性原理**：块编辑器的核心不是"编辑"，而是"多人实时协同编辑同一棵树"。Yjs 解决了 90% 的问题，剩下的 10% 是存储和查询。
