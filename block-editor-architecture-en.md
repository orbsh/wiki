# Block Editor Architecture: A Notion-Style Block Editor

> **Languages:** [English](block-editor-architecture-en.md) (primary) · [中文](block-editor-architecture.md)

**Status**: Architecture design complete
**Date**: 2026-06-20
**Core philosophy**: Blocks are atoms, CRDT is the soul, Yjs is the muscle

> A Notion-style block editor, whose core challenge lies not in "rendering" but in "multiple people collaboratively editing the same tree in real time". This document presents a complete solution, from data structures to frontend and backend.

---

## 1. Core Problem Breakdown

| Problem | Solution |
|------|------|
| How are blocks stored? | Tree structure; each block has a unique ID; parent-child relationships are maintained via `children: Vec<BlockId>` |
| How do concurrent editors avoid conflicts? | CRDT (Yjs), automatic merging, no OT server required |
| How does the frontend render? | React/Vue component mapping; each block type corresponds to one component |
| How does the backend store? | Fjall stores Yjs binary documents + incremental update logs |
| How are blocks ordered? | Yjs Y.Array guarantees order consistency |

---

## 2. Data Structure Design

### 2.1 The Block Enum (Rust)

```rust
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

/// Unique identifier for a block
pub type BlockId = String;

/// Block type enum — uses the default Externally Tagged mode
/// Compatible with Postcard/Bincode (does not trigger deserialize_any)
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
    Column { ratio: f32 },           // Layout column
}

/// A single block
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct Block {
    pub id: BlockId,
    pub block_type: BlockType,
    pub content: Option<String>,      // Rich text (Markdown or JSON Rich Text)
    pub children: Vec<BlockId>,       // List of child block IDs (ordered)
    pub metadata: HashMap<String, serde_json::Value>,  // Extension fields
}

/// Document = collection of blocks + root block order
#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct Document {
    pub id: String,
    pub title: String,
    pub root_block_ids: Vec<BlockId>,  // Order of top-level blocks
    pub blocks: HashMap<BlockId, Block>,
    pub version: u64,
}
```

### 2.2 Why Use HashMap + Vec Instead of a Nested Tree?

```
❌ Nested tree (Notion's physical storage):
Block {
    children: Vec<Block>,  // deep nesting becomes performance hell
}

✅ Flattened HashMap (recommended):
Document {
    blocks: HashMap<BlockId, Block>,  // O(1) lookup
    root_block_ids: Vec<BlockId>,     // top-level order
}
```

**Advantages**:
- **O(1) random access**: Locate any block directly by its ID, without traversing the tree
- **Zero-copy block moves**: Moving from parent A's children to parent B's children only requires changing the ID reference
- **CRDT-friendly**: Yjs's Y.Map naturally corresponds to a HashMap, and Y.Array naturally corresponds to a Vec
- **Simple serialization**: Postcard/CBOR serializes the entire HashMap directly, with no recursion needed

### 2.3 Rich Text Model (Inline Content)

A block's `content` field stores inline content. Notion represents rich text as JSON; we use a more compact scheme:

```rust
/// Inline content span
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct InlineSpan {
    pub text: String,
    pub marks: Vec<Mark>,  // bold, italic, link, etc.
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

/// Rich text = multiple spans
pub type RichText = Vec<InlineSpan>;
```

**Why Not Store Rich Text Directly with Yjs Y.Text?**
- Yjs Y.Text is CRDT text, well suited to collaborative editing scenarios
- But it must be serialized for storage; `RichText` is more compact
- During collaborative editing the frontend uses Y.Text, and the backend converts it to `RichText` when storing

---

## 3. Yjs CRDT Collaboration Architecture

### 3.1 Why Choose Yjs?

| Dimension | Yjs | Automerge | ShareDB (OT) |
|------|-----|-----------|--------------|
| Performance | **Extremely fast** (C implementation, WASM) | Slower (pure JS) | Requires a central server |
| Document size | **Very small** (incremental encoding) | Larger | Medium |
| Offline support | **Perfect** | Supported | Not supported |
| Ecosystem | **Most mature** (ProseMirror, TipTap) | Growing | Dated |
| Memory footprint | **Low** | High | Medium |

### 3.2 Yjs Document Structure Mapping

```
Y.Doc
├── Y.Map "blocks"           ←→ Document.blocks (HashMap)
│   ├── Y.Map "block-uuid-1" ←→ Block { id, type, content, children }
│   ├── Y.Map "block-uuid-2" ←→ ...
│   └── ...
├── Y.Array "root_block_ids" ←→ Document.root_block_ids (Vec)
└── Y.Text "title"           ←→ Document.title
```

### 3.3 Rust Backend: Yrs (the Rust Implementation of Yjs)

```rust
use yrs::{Doc, Map, Array, Text, Transaction, ReadTxn};

/// Create a new document
pub fn create_document(doc_id: &str) -> Doc {
    let doc = Doc::new();
    let mut txn = doc.transact_mut();

    // Initialize the root block list
    let root_ids = txn.get_array("root_block_ids");
    root_ids.insert(&mut txn, 0, "block-root-001");

    // Initialize the block collection
    let blocks = txn.get_map("blocks");
    let mut block = Map::new();
    block.insert(&mut txn, "id", "block-root-001");
    block.insert(&mut txn, "type", "paragraph");
    block.insert(&mut txn, "content", "");
    block.insert(&mut txn, "children", Array::new());
    blocks.insert(&mut txn, "block-root-001", block);

    doc
}

/// Apply an incremental update (received from WebSocket)
pub fn apply_update(doc: &Doc, update: &[u8]) {
    let mut txn = doc.transact_mut();
    doc.apply_update(&mut txn, update);
}

/// Get a binary snapshot (for persistence)
pub fn get_snapshot(doc: &Doc) -> Vec<u8> {
    doc.encode_state_as_update_v1(&Default::default())
}
```

### 3.4 Collaboration Protocol (WebSocket)

```
Client A ──┐                    ┌── Client B
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
     │   (Yjs documents +            │
     │   incremental update log)     │
     └───────────────────────────────┘
```

**Message format** (serialized with Postcard):

```rust
#[derive(Serialize, Deserialize)]
pub enum WsMessage {
    /// Client sends an incremental update
    SyncStep1(Vec<u8>),           // Yjs-encoded delta
    /// Server replies with a state vector
    SyncStep2(Vec<u8>),           // Yjs state vector
    /// Bidirectional incremental sync
    Update(Vec<u8>),              // Yjs incremental update
    /// Cursor/selection awareness (Awareness)
    Awareness { client_id: u64, state: Vec<u8> },
}
```

---

## 4. Storage Architecture

### 4.1 Fjall Storage Design

```
Fjall Database
├── "documents"          Keyspace
│   ├── Key: doc_id      Value: Document metadata (Postcard)
│   └── ...
├── "blocks"             Keyspace
│   ├── Key: (doc_id, block_id)  Value: Block (Postcard)
│   └── ...
└── "yjs_snapshots"      Keyspace
    ├── Key: doc_id      Value: Yjs full snapshot (binary)
    └── ...
└── "yjs_updates"        Keyspace
    ├── Key: (doc_id, seq)  Value: Yjs incremental update (binary)
    └── ...
```

### 4.2 Write Strategy

```
Real-time collaboration flow:
1. Client A edits → produces a Yjs incremental update
2. The incremental update is broadcast to all clients via WebSocket
3. At the same time, it is written to Fjall "yjs_updates" (append log)
4. Periodically (every 5 minutes or 1000 updates) merge into a full snapshot, written to "yjs_snapshots"
5. After merging, clean up the old incremental update log
```

### 4.3 Query Scenarios

| Scenario | Query method |
|------|----------|
| Open a document | Read `yjs_snapshots[doc_id]` → apply unmerged updates |
| Render all blocks | Read `blocks[(doc_id, *)]` → Postcard deserialization |
| Search block content | Fjall full-text index (or a separate Tantivy index) |
| Export to Markdown | Traverse `root_block_ids` → recursively render the block tree |
| Version history | Keep Yjs snapshots at key points in time |

---

## 5. Frontend Architecture

### 5.1 React Component Mapping

```typescript
// Block type definition (TypeScript)
interface Block {
  id: string;
  blockType: BlockType;
  content: string | null;
  children: string[];
  metadata: Record<string, any>;
}

// Block component mapping
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

// Main editor component
function BlockEditor({ docId }: { docId: string }) {
  const [doc, setDoc] = useState<Document | null>(null);
  const ydoc = useRef(new Y.Doc());

  useEffect(() => {
    // Connect WebSocket, load the document
    const ws = new WebSocket(`ws://localhost:8080/doc/${docId}`);
    ws.onmessage = (e) => {
      const update = new Uint8Array(e.data);
      Y.applyUpdate(ydoc.current, update);
      // Re-render
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

### 5.2 A Single Block Component Example

```typescript
function ParagraphBlock({ block, ydoc }: BlockProps) {
  const textRef = useRef<Y.Text>();

  useEffect(() => {
    // Bind Yjs Y.Text to the DOM
    const ytext = ydoc.getText(`block:${block.id}:content`);
    textRef.current = ytext;

    // Auto-sync to contenteditable with a Yjs Binding
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

### 5.3 Block Operations API

```typescript
// Block operations (via Yjs Map operations, auto-synced)
class BlockOperations {
  constructor(private ydoc: Y.Doc) {}

  // Add a block
  addBlock(parentId: string, index: number, type: BlockType) {
    const blocks = this.ydoc.getMap('blocks');
    const newBlock = new Y.Map();
    const id = crypto.randomUUID();

    newBlock.set('id', id);
    newBlock.set('type', type);
    newBlock.set('content', '');
    newBlock.set('children', new Y.Array());

    blocks.set(id, newBlock);

    // Add to the parent block's children
    const parent = blocks.get(parentId) as Y.Map;
    const children = parent.get('children') as Y.Array;
    children.insert(index, [id]);
  }

  // Delete a block
  deleteBlock(blockId: string) {
    const blocks = this.ydoc.getMap('blocks');
    const block = blocks.get(blockId) as Y.Map;

    // Recursively delete child blocks
    const children = block.get('children') as Y.Array;
    for (let i = children.length - 1; i >= 0; i--) {
      this.deleteBlock(children.get(i) as string);
    }

    // Remove from the parent block's children
    // (Need to find the parent block first; can be done by traversal or maintaining a parentMap)
    blocks.delete(blockId);
  }

  // Move a block
  moveBlock(blockId: string, newParentId: string, index: number) {
    // Remove from the old parent block
    // Insert into the new parent block
  }
}
```

---

## 6. Complete Data Flow

```
User types in Block A
        │
        ▼
[React contenteditable] ──input──► [Y.Text for block:A:content]
        │
        ▼
[Y.Doc produces an incremental update]
        │
        ├──► [WebSocket broadcasts to other clients]
        │         │
        │         ▼
        │    [Other clients' Y.Doc merges the update]
        │         │
        │         ▼
        │    [React re-renders the affected blocks]
        │
        └──► [WebSocket server receives the update]
                  │
                  ▼
             [Fjall appends to yjs_updates]
                  │
                  ▼
             [Periodically merged into yjs_snapshots]
```

---

## 7. Integration with the Aura Ecosystem

| Component | Correspondence in Aura | Integration method |
|------|--------------|----------|
| Block storage | Fjall Keyspace | The `blocks` keyspace uses Fjall directly |
| Collaborative sync | A separate consensus service (etcd-like) | Yjs handles CRDT merging; metadata consistency goes through external consensus or single-writer |
| Text search | Tantivy | Block content is indexed into Tantivy, enabling full-text search |
| Vector search | LanceDB | Block content embeddings are stored in LanceDB, enabling semantic search |
| AI generation | Steel Lisp | Lisp scripts generate Block JSON → validate → insert into the document |
| Export | Arrow IPC | The document is converted to Arrow columnar format, enabling Polars analysis |

---

## 8. Technology Selection Decisions

| Decision point | Choice | Rationale |
|--------|------|------|
| CRDT engine | Yjs (yrs) | Best performance, most mature ecosystem, validated by ProseMirror/TipTap |
| Serialization format | Postcard (RPC) + CBOR (storage) | Same full stack as Aura; Externally Tagged enum compatible |
| Storage engine | Fjall | Same full stack as Aura; LSM-Tree suits append writes |
| Frontend framework | React + TipTap | TipTap is Yjs's best partner, the de facto standard for block editors |
| WebSocket | tokio-tungstenite | Native Rust, seamlessly integrated with the Tokio ecosystem |
| Rich text | TipTap (ProseMirror) | De facto standard for block editors, official Yjs Binding |

---

## 9. Cross-References

- **[Aura Architecture](aura-architecture-en.md)**: A modern distributed booth engine with storage-compute integration
- **[Serialization Protocol Comparison](serialization-protocol-comparison-en.md)**: Postcard + Externally Tagged enum
- **[Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md)**: Fjall + Arrow + Polars full-stack storage-compute integration
- **[Fluxora Architecture](projects/fluxora-architecture-en.md)**: An AI-native UI framework; the block editor can serve as Fluxora's document editing component

**First principles**: The core of a block editor is not "editing" but "multiple people collaboratively editing the same tree in real time". Yjs solves 90% of the problem; the remaining 10% is storage and querying.
