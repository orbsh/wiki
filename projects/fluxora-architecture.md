# Fluxora 架构

**路径**：`~/world/fluxora/`  
**类型**：事件驱动、AI 原生 UI 框架（Rust/Dioxus）  
**状态**：仅维护（定期依赖更新）。战略重心已转移到图记忆。

## 核心概念

Fluxora 不是传统的 Web 框架。它是一个**事件驱动的 UI 总线**，通过 Kafka/Iggy 消息队列将业务逻辑与展示解耦。UI 通过声明式 JSON DSL（`Brick`）描述，而非代码。

## 架构拓扑

```
┌──────────┐   WS    ┌──────────┐  outgo   ┌─────────────────┐
│   UI     │◄───────►│ Gateway  │─────────►│ 业务服务         │
│ (Dioxus) │         │ (Axum)   │          │ (Chat, CRM, AI…)│
└──────────┘         └────┬─────┘          └────────┬────────┘
                          │   income                │
                          └─────────────────────────┘
```

- **UI（Dioxus/WASM）**：渲染 `Brick` JSON 树。组件绑定到事件名称——无 API URL，无 HTTP 动词。
- **Gateway（Axum）**：WebSocket 路由器。将 UI 事件分发到 Kafka `outgo`；将后端 `income` 投递到正确的 WS 会话。处理 greet 钩子、webhook 拦截、模板渲染。
- **业务服务**：独立的 Kafka 消费者。每个服务（chat、crm、echo、analysis…）处理自己的逻辑。服务彼此**透明**——analysis 可以拦截 chat 流而 chat 不知情。

## 项目结构

```
fluxora/
├── crates/brick/         # 核心 JSON DSL——类型化 Brick 枚举、属性、绑定
├── crates/brick_macro/   # BrickOps、ClassifyBrick、渲染提示的派生宏
├── crates/message/       # 统一消息协议、Event trait、Kafka/Iggy、Codec
├── crates/content/       # 内容动作：Create、Set、Join、Tmpl、Empty
├── crates/gateway/       # WS 路由器 + minijinja + webhooks + 会话管理
├── crates/ui/            # Dioxus/WASM 前端——Frame 渲染器、WS store、merge
├── crates/ui_macro/      # UI 组件派生宏
├── crates/chat/          # 演示业务服务（基于 channel 的聊天）
└── crates/agent/         # Agent 服务骨架
```

## 核心原则

1. **AI 原生，非 AI 依赖**：AI 生成由 Schema 验证的结构化 JSON（Brick DSL）。固定业务使用模板以获得性能；AI 用于动态/探索性 UI。
2. **事件溯源**：通过 Kafka/Iggy（`outgo`/`income` 队列）解耦服务。服务透明地拦截流而彼此不知情。
3. **声明式绑定**：组件绑定到事件名称，而非 API URL。Input=POST、Display=GET 语义。Gateway 将事件名称映射到 Kafka topic。
4. **协议无关**：三层编解码策略（见下文）。CBOR 是默认二进制编解码器；bincode 已被移除。Postcard 可作为轻量替代，但需配合 Externally Tagged 枚举模式（见 [ADR-001 补充](fluxora-decisions.md#adr-001cbor-优于-bincode)）。

---

## 为什么 AI 生成 JSON 而不是 HTML

传统 Web 开发中，AI 直接输出 HTML/CSS/JS 标记。这有三个致命问题：

1. **体积膨胀** — 页面的 HTML/JS/CSS 远比正文内容长，AI 要生成大量样板标记（标签闭合、class 命名、嵌套层级），token 消耗高
2. **输出不稳定** — 同样的 prompt，每次生成的样式细节都不同（class 名、间距、嵌套结构），表面相似但细节差异显著。HTML 是视觉产物，数据正确性只能靠无头测试或肉眼识别
3. **验证困难** — HTML 没有 schema，AI 输出对不对无法在编译期检查

JSON-render 的思路是反过来：**AI 只生成结构化 JSON（数据 + 组件类型声明），不做任何样式决策**。框架负责把 JSON 渲染成固定样式的 UI。

| 维度 | AI 生成 HTML | AI 生成 JSON（Fluxora Brick） |
|------|-------------|-------------------------------|
| Token 消耗 | 高（大量标记样板） | 低（JSON 比 HTML 小一个数量级） |
| 输出稳定性 | 每次不同（样式漂移） | 确定性（样式由框架锁死） |
| 数据验证 | 无 schema，靠视觉检查 | JsonSchema 编译期验证 |
| AI 职责 | 内容 + 结构 + 样式（全包） | 只管内容和结构（样式交给框架） |

**本质上是把"创意决策"（样式）从 AI 手里拿走，只让 AI 做它擅长的事（内容和结构）。** Anthropic 近期鼓吹用 HTML 替代 Markdown 作为 AI 输出格式，方向反了——HTML 越来越重，AI 的输出越不稳定。JSON-render 是让 AI 的输出尽量轻、尽量结构化，渲染的活交给确定性代码。

---

## `brick` Crate（UI DSL）

**位置**：`crates/brick/src/lib.rs`

### Brick 枚举

- `#[serde(tag = "type")]` — 零歧义 JSON 序列化。AI 输出 `{"type": "button", ...}` → 自动反序列化为 `Brick::Button`。
- 通过 `sub: Option<Vec<Brick>>` 实现递归树。
- 通过 `item: Option<Vec<Brick>>` 实现列表项（用于 `Rack`、`Fold`）。
- 通过 `Render { name, data }` 实现模板实例化。
- 所有结构体派生：`Serialize, Deserialize, JsonSchema, BrickOps`。

### Bind 系统

```json
{
  "bind": {
    "input": { "event": "chat.send", "type": "text" },
    "output": { "source": "chat.msg" }
  }
}
```

- **`source`**：监听事件流（GET 等价）。
- **`event`**：发出事件（POST 等价）。
- **`target` / `field` / `submit`**：高级绑定模式。

无 API URL。Gateway 自动将事件名称映射到 Kafka topic。

### 流式合并（`crates/brick/src/merge.rs`）

三种合并策略用于局部化路径更新：

- **`Replace`**：覆盖值。
- **`Concat`**：追加（文本拼接、数字相加、数组 push、对象合并）。
- **`Delete`**：移除（字符串替换、数字相减、对象键移除）。

**关键**：合并是**相对于组件绑定的数据源**，而非全局。组件通过 `id` 匹配合并。多个服务流式传输到同一页面而无冲突。

### 宏（`brick_macro`）

- `BrickOps`：自动生成 `get_type()`、`borrow_sub()`、`get_bind()` 等。
- `ClassifyBrick` / `ClassifyAttrs`：分类支持。
- `render_brick`：渲染提示。

---

## 组件系统设计权衡

### 封闭组件集：为什么不允许扩展

组件集是**有限封闭**的，刻意不允许自由扩展。三个约束共同决定：

1. **AI 可靠性** — 组件集是 AI 生成 JSON 的"词汇表"。词汇表无限膨胀，AI 无法可靠输出正确组件。封闭集 = AI 可以穷举学习
2. **WASM 体积** — 每个组件增加 bundle size。组件数量直接影响用户加载速度
3. **枚举即门卫** — `Brick` enum 是封闭集的显式边界。加组件必须手动过 enum 这关，这个摩擦力是故意的

边界可能调整（组件替换），但总数不会有大的变化。

### 为什么用枚举而非运行时注册

`inventory`/`linkme` 运行时注册方案（每个组件自注册，中央 registry 按名字查找）不适用：

1. **WASM 兼容性** — `inventory` 依赖链接器行为收集注册项，WASM 目标下可能静默丢失注册
2. **类型擦除** — 运行时注册需要统一成函数指针签名，Dioxus `rsx!` 的编译期类型安全被破坏
3. **编译期安全丢失** — 组件函数签名错了、漏注册了，运行时才报错

### 为什么不用宏 DSL 替代枚举

声明式宏统一定义（一个宏调用同时生成枚举和分发）不适用：

1. **集中审计** — 枚举是所有组件的一览表，review 时一眼看清全貌。宏 DSL 把定义分散到调用语法里，失去集中管控
2. **模块排除** — 禁用组件 = 从 enum 删一行。宏方案需要额外机制控制 inclusion

### `gen_dispatch!` 的取舍

过程宏从 Brick 枚举 AST 自动生成 match 分发。每个选择都有明确理由：

| 选择 | 替代方案 | 不采用的理由 |
|------|---------|-----------|
| `include_bytes!` 追踪文件 | `build.rs` + `cargo:rerun-if-changed` | 避免独立进程，破坏增量编译 |
| 自写 `syn` AST walk | `darling` / `strum` | 属性就几个 key=value，零额外依赖 |
| `Span::call_site()` 错误 | `Span::mixed_site()` | 无外部用户，不存在调试场景 |
| 保留 Brick enum | `inventory` 运行时注册 / trait object | 一行注册换零成本 serde + 编译期穷举 + WASM 兼容 |

### 开放组件系统在 Rust/WASM 下不可行

如果需求是开放组件（加组件不改中心文件），可选方案：

- **`inventory` 运行时注册** — WASM 下链接器行为不可靠
- **`typetag` trait object** — 依赖 `inventory`，同样的 WASM 问题 + vtable 开销
- **声明式宏 DSL** — 本质仍是集中定义，不算真正开放

Rust 的类型系统和 WASM 编译模型天然推着你走向封闭集。JavaScript 能做到开放是因为运行时动态性。Rust 编译期确定一切，开放注册的基础设施在 WASM 下不可靠。封闭 enum 不是妥协，是约束下的自然解。

---

## 后端：事件溯源 + 消息队列

### 消息协议（`crates/message/`）

- `Envelope<Created>`：`{ receiver: Vec<Session>, message: ChatMessage }`
- `ChatMessage<Created>`：`{ sender, created, content: Value }`
- `Event<C>` trait：`fn event(&self) -> Option<&str>`
- 同时支持 **Kafka**（`rdkafka`）和 **Iggy**（`iggy`）后端。

### 内容动作（`crates/content/`）

- `Create`：设置根布局 Brick。
- `Set`：设置命名数据 Brick（按事件名称键控）。
- `Join`：合并到列表（支持 Replace/Concat/Delete 方法）。
- `Tmpl`：注册模板（名称 + minijinja 字符串）。
- `Empty`：无操作。

### Gateway（`crates/gateway/`）

- **WebSocket 处理器**：会话管理、greet 钩子、事件路由到 `outgo_tx`。
- **模板引擎**：`minijinja` 用于 Gateway 层模板渲染。
- **Webhooks**：拦截特定事件（如登录）进行特殊处理。可调用外部端点或应用本地模板。
- **会话分发**：`send_to_ws()` 监听 `income_rx` 并按 `receiver` 字段路由。

### UI Store（`crates/ui/src/libs/store.rs`）

- `Status`：持有 `layout: Signal<Brick>`、`data: Signal<HashMap<String, Brick>>`、`list: Signal<HashMap<String, Vec<Brick>>>`。
- `dispatch()`：处理传入的 `Message<Brick>` 动作（Create、Set、Join、Tmpl）。
- 模板渲染通过全局 `TMPL: LazyLock<RwLock<Environment>>`。
- 流式合并：`Join` 动作使用 `BrickOp` trait（Replace/Concat/Delete）与基于 `id` 的匹配。

---

## 编解码器架构

### 为何不用 `dyn Codec`？

带泛型方法的 Rust trait **不是对象安全的**：

```rust
// 这会失败：
pub trait Codec {
    fn encode<T: Serialize>(&self, value: &T) -> Result<Vec<u8>>;  // 泛型 T！
}
pub fn build_codec() -> Box<dyn Codec> { ... }  // 错误：trait 不 dyn 兼容
```

### 解决方案：枚举分发（`ActiveCodec`）

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq, serde::Deserialize)]
pub enum CodecType {
    Json,
    #[default]
    Cbor,
}

#[derive(Debug, Clone)]
pub enum ActiveCodec {
    Json,
    Cbor,
}

impl ActiveCodec {
    pub fn new(t: CodecType) -> Self {
        match t {
            CodecType::Json => Self::Json,
            CodecType::Cbor => Self::Cbor,
        }
    }

    pub fn as_type(&self) -> CodecType {
        match self {
            Self::Json => CodecType::Json,
            Self::Cbor => CodecType::Cbor,
        }
    }

    pub fn encode<T: Serialize>(&self, value: &T) -> Result<Vec<u8>, CodecError> {
        match self {
            Self::Json => serde_json::to_vec(value).map_err(|e| CodecError::Encode(e.to_string())),
            Self::Cbor => {
                let mut out = Vec::new();
                ciborium::ser::into_writer(value, &mut out)
                    .map_err(|e| CodecError::Encode(e.to_string()))?;
                Ok(out)
            }
        }
    }

    pub fn decode<T: DeserializeOwned>(&self, bytes: &[u8]) -> Result<T, CodecError> {
        match self {
            Self::Json => serde_json::from_slice(bytes).map_err(|e| CodecError::Decode(e.to_string())),
            Self::Cbor => {
                ciborium::de::from_reader(&mut std::io::Cursor::new(bytes))
                    .map_err(|e| CodecError::Decode(e.to_string()))
            }
        }
    }
}
```

零 vtable 开销，完全可 Clone 用于异步任务。

### 为何拒绝 Bincode

- **根因**：Bincode **无法反序列化 `#[serde(tag = "...")]` 枚举**——它调用 `deserialize_any`，而 bincode 不支持。这影响 `Brick`/`Content` 类型的所有反序列化，不仅是线路协议。
- serde 2.x 不兼容（v1.x 损坏，v2.x API 不稳定）
- 无类型自描述（Gateway 无法部分解析路由元数据）
- 无跨语言支持（bincode 无 JS 库）

### 为何用 CBOR 而非 MessagePack？

- CBOR 是 IETF 标准（RFC 8949），有更好的类型标记和确定性长度前缀，支持部分解析/查找。
- 成熟的 JS 生态（`cbor-x`），对未来前端集成至关重要。
- 自描述——对 WS 和 Kafka 工作负载来说，轻微的大小/速度权衡可接受。

### 三层编解码策略

| 层 | 策略 | 原因 |
|----|------|------|
| **Gateway WS** | 握手时从 URL 参数 `?codec=json\|cbor` 固定 | URL 参数在 WS 升级时读取。Greet 立即以正确编解码器发送。零 RTT。|
| **Kafka 队列** | 硬编码 CBOR | 所有队列流量用 CBOR。生产者：`into_writer`；消费者：`from_reader`。如出现多语言消费者，Avro/Schema Registry 作为未来可能性记录。|
| **UI（WASM）** | 默认 CBOR，查询参数覆盖 | `Status` 存储 `ActiveCodec`，`send()` 选择 Text vs Bytes 帧。|

### 为何不是全局单一可配置编解码器？

统一编解码器设置在纸面上更简单但在实践中失败：

- **Gateway** 服务异构客户端。URL 参数提供每连接控制而无配置负担。
- **Kafka** 存储持久状态。始终 CBOR——无配置。CBOR 的自描述意味着调试仍然可能。如出现多语言消费者，Avro 作为未来选项记录。
- **UI** 是用户面向层。默认应是最高效选项（CBOR），带调试逃生舱。

### Gateway：握手时固定编解码器（URL 参数）

编解码器在 WS 升级时解析——**零额外 RTT**。编解码器**在握手时固定**——无动态检测，无首帧适配。

**握手流程迭代**（早期方法被迭代和完善）：

1. ~~Greet 立即以固定 JSON 发送~~ — 如果客户端期望 CBOR 则编解码器不匹配
2. ~~客户端先发送 `client_info` 首帧，然后 greet~~ — 在 UI 渲染前增加 1 RTT
3. **当前：URL `?codec=json|cbor` 在 WS 升级时解析** — 零额外 RTT，greet 立即以正确编解码器发送

```rust
// main.rs — 路由处理器从 URL 读取编解码器
let mut codec = codec_for_router.clone();
if let Some(codec_str) = q.get("codec").and_then(|v| v.as_str()) {
    if let Ok(ct) = codec_str.parse::<CodecType>() {
        codec = ActiveCodec::new(ct);
    }
}
ws.on_upgrade(async move |socket| {
    handle_ws(socket, tx, state, config, tmpls.clone(), codec, &a).await;
});
```

**握手流程（零 RTT）**：
1. 客户端打开 WS 连接（`/channel?codec=json&token=xxx`）
2. Gateway 从 URL 读取编解码器 → 立即发送 greet（正确帧类型）
3. 客户端接收 greet → 渲染基础 UI
4. 所有后续消息使用相同编解码器（连接生命周期内固定）

**`encode_ws()` 辅助函数**：

```rust
// shared.rs — CodecType 直接在 Client 上（CodecHint 作为冗余被移除）
pub struct Client<T> {
    pub sender: T,
    pub codec: CodecType,  // 握手时从 URL 参数固定
}

pub fn encode_ws<T: Serialize>(codec: CodecType, value: &T) -> Option<axum::extract::ws::Message> {
    let bytes = ActiveCodec::new(codec).encode(value).ok()?;
    Some(match codec {
        CodecType::Json => axum::extract::ws::Message::Text(String::from_utf8(bytes).unwrap().into()),
        CodecType::Cbor => axum::extract::ws::Message::Binary(bytes.into()),
    })
}
```

**路由器从 URL 解析编解码器**：
```rust
let mut codec = config_codec.clone();
if let Some(cs) = q.get("codec").and_then(|v| v.as_str()) {
    if let Ok(ct) = cs.parse::<CodecType>() { codec = ActiveCodec::new(ct); }
}
ws.on_upgrade(async move |socket| {
    handle_ws(socket, tx, state, config, tmpls.clone(), codec, &a).await;
});
```

### 为何移除 `CodecHint`

`CodecHint { Json, Cbor }` 在结构上与 `CodecType { Json, Cbor }` 相同，带 `From` 转换。它复制了 `ActiveCodec::encode()` 逻辑——`CodecHint::encode()` 只是"编码字节 → 包装在 Text/Binary 帧中"。**模式：不要创建 1:1 映射的并行枚举类型。** 直接使用 `CodecType` + `encode_ws()` 辅助函数。

### UI（WASM）

UI 尊重 `?codec=json` 或 `?codec=cbor` URL 参数进行发送和接收。`ActiveCodec` 存储在 `Status` 中，由 `send()` 使用以决定帧类型：

```rust
// store.rs — Status::send 尊重编解码器
if let Ok(buf) = self.codec.encode(&outflow) {
    let msg = match &self.codec {
        ActiveCodec::Json => gloo_net::websocket::Message::Text(String::from_utf8(buf).unwrap()),
        ActiveCodec::Cbor => gloo_net::websocket::Message::Bytes(buf),
    };
    self.ws.send(msg).await;
}
```

**注意**：`gloo_net` 使用 `Message::Bytes`，不是 `Message::Binary`。

### WASM 依赖隔离

当 `ui` 依赖 `message` 时，禁用默认特性以避免 WASM 不兼容的原生依赖：

```toml
# crates/ui/Cargo.toml
message = { path = "../message", default-features = false }
```

### Kafka 队列

Kafka 使用**硬编码 CBOR**——无需配置。

生产者编码 CBOR 二进制，消费者通过 `ciborium::de::from_reader` 解码。

**迁移**（2026-05-08，Bincode → CBOR）：

| 文件 | 变更内容 |
|------|---------|
| `crates/message/src/codec.rs` | 移除 `CodecType::Bincode` 和 `ActiveCodec::Bincode` 变体 |
| `Cargo.toml`（workspace）| 移除 `bincode = "1.3.3"` |
| `crates/message/Cargo.toml` | 移除 `bincode.workspace = true` |
| `crates/gateway/Cargo.toml` | 移除 `bincode.workspace = true` |
| `crates/ui/Cargo.toml` | 添加 `ciborium.workspace = true` |
| `crates/ui/src/libs/store.rs` | `Status::send()` → CBOR 编码 + `gloo_net::websocket::Message::Bytes` |
| `crates/message/src/kafka/mod.rs` | 生产者：`into_writer`；消费者：`from_reader` 替代 `serde_json::from_str` |

**Kafka 生产者发送原始字节**：
```rust
let mut buf = Vec::new();
into_writer(&value, &mut buf)?;
producer.send(FutureRecord::to(&topic).payload(&buf), ...);
```

**Kafka 消费者有效载荷是二进制，非字符串**：
```rust
// 旧：payload_view::<str>() + serde_json::from_str
let payload = msg.payload().unwrap();
let mut cursor = std::io::Cursor::new(payload);
let value: T = ciborium::de::from_reader::<T, _>(&mut cursor)?;
```

### 性能说明

- 枚举分发 = 编译时 match，零 vtable 开销。
- 编解码枚举上的 `Clone` 廉价（单元变体）。
- CBOR 是默认——接近 bincode 性能，带 IETF 标准类型自描述和跨语言支持。
- JSON 用于调试和 AI 生成内容。

---

## 流式合并策略：`Vec<String>` 片段缓冲

### 为何用 `Vec<String>`，而非 `&[&str]` 或 `String`

三种方法被评估用于累积流式文本片段：

| 方法 | 问题 | 状态 |
|------|------|------|
| `&[&str]` | 生命周期地狱 + **内存放大** — 引用将完整原始片段（信封 + 有效载荷）锁定在内存中；渲染完成前无法丢弃信封。放大因子 = 1 / payload_ratio（如 20% 有效载荷 → 5× 内存）| 拒绝 |
| `String` + `push_str` | 随缓冲区增长，每 token 可能 O(N) 重新分配 + memcpy | 拒绝 |
| `Vec<String>` + `push` | O(1) 摊销追加；仅 N × 8 字节指针开销；有效载荷可立即提取，信封丢弃 | **接受** |

`Vec<String>` 和 `String` 存储基本相同的内容——`Vec` 只是每片段添加一个指针（8 字节）。分配模式严格更优：每 token 摊销 O(1) vs `push_str` 可能每 token O(N) 重新分配。

### 渲染策略

渲染组件最终将被重新设计为直接消费 `Vec<String>`，迭代片段而不预连接。这甚至避免单次最终分配。暂时推迟。

### 协议开销

每个片段携带完整 JSON 信封（元数据、路由），其中文本有效载荷是小部分。每片段存储完整 `String` 意味着内存随信封大小缩放，而不仅是有效载荷。这是解耦架构的可接受成本。如果这成为瓶颈，在路由后仅提取和存储有效载荷。

### 关键原则

- **强制反序列化**：每条消息（即使单个 token）必须反序列化以定位正确的 UI 树节点。
- **频繁合并**：每 token 合并以实现流畅"打字效果"。比"批量复制"更高 CPU 成本但防止浏览器卡顿。

---

## 组件原子性：Table 和 SVG

- **直接映射**：这些直接映射到标准 HTML/SVG 元素。
- **AI 对齐**：LLM 在标准 HTML/SVG 上大量训练。保持原子确保高生成准确性。
- **无过度抽象**：语义包装器不必要且阻碍灵活性。

## 并发：同步模板引擎

- **CPU 绑定**：`minijinja` 渲染是纯计算，无 IO。阻塞时间可忽略（微秒）。
- **无需 spawn_blocking**：异步包装器为轻量级文本处理增加开销。
- **为规模保留**：异步卸载仅在高并发/重模板场景需要。

## 设计时 vs 运行时

### 设计时（AI 辅助）

- AI 生成 Brick JSON + 模拟数据 → 开发者在 UI 中预览 → 保存为模板。
- AI 充当"超级前端工程师"——降低编写 JSON 组件的门槛。

### 运行时（模板驱动）

- 业务服务返回纯数据 → Gateway 应用模板 → 毫秒级速度渲染。
- 零 LLM 延迟。确定性输出。无敏感数据馈送给 LLM。

### 探索性（AI 生成）

- 当结构不可预测时（仪表板、演示文稿），AI 生成一次性 UI。
- 仅保留给极端场景。

## CSS 策略

- 现代 CSS：变量、嵌套、`@layer`。
- 基于 trait 的组合（如 `.accent.txt`、`.accent.border`）。
- 无需预处理器。

## 战略意义

- **厚壳**：通过 Schema 和类型控制 AI 输出，确保可靠性。
- **确定性 UI**：JSON 生成比代码生成更低 token、更高准确性。
- **多 Agent 就绪**：服务彼此透明；任意数量 Agent 可拦截和注入。
- **当前状态**：完成。维护模式。重心转移到图记忆作为核心差异化因素。

## 未来迁移路径：Aura 底层

当 Fluxora 成熟时，可以选择将底层从 Kafka 迁移到 **Aura**（存算一体的现代分布式 Actor 引擎）。

### 迁移收益

| 维度 | Kafka（当前） | Aura（未来） |
|------|--------------|-------------|
| **部署复杂度** | 需要 Kafka 集群 | 单二进制，14ms 启动 |
| **一致性** | 最终一致 | 强一致（Raft） |
| **状态存储** | 无（无状态服务） | Fjall（有状态 Actor） |
| **分析能力** | 无 | Polars 内存联邦查询 |
| **运维成本** | 高（Kafka 集群管理） | 低（单二进制） |

### 迁移策略

1. **渐进式迁移**：Fluxora 的 Gateway 层可以选择性接入 Aura 后端
2. **双轨运行**：Kafka 和 Aura 可以并行运行，逐步切换
3. **零停机**：通过事件溯源日志（RaftLog）实现平滑迁移

### 架构关系

```
[Fluxora UI 层]（独立项目）
    ↓ 可选接入
[Aura 底层]（独立项目）
    ├── Openraft（Raft 共识）
    ├── Fjall（Arrow IPC 存储）
    └── Polars（内存分析）
```

**关键点**：两个项目保持独立。Fluxora 是一个完整的事件驱动 UI 框架，可以独立运行；Aura 是一个底层分布式基础设施，可以被其他项目使用。

## 为何事件溯源？

- **解耦**：业务服务是独立 Kafka 消费者。添加/移除无需触碰其他。
- **可审计**：完整事件历史支持重放、调试、AI 分析。
- **多 Agent**：服务透明地拦截流并注入响应。

## 为何分离队列？

- **流控**：AI 流式 = 高频/碎片化。专用队列防止阻塞低频控制消息（通知、加入等）。

## 架构：厚壳，主权控制

- Fluxora 体现 Orbit 的"厚壳"哲学：系统通过 Schema 验证控制 AI 输出，而非相反。
- AI 在约束内生成；框架确保正确性。
- 业务逻辑完全独立于 UI 渲染——服务生产数据，Gateway 应用模板，UI 渲染。

## 快速命令

```bash
cargo build --workspace   # 构建所有 crate
cargo build -p gateway    # 仅构建 Gateway
cargo build -p ui         # 构建 UI（WASM）
```

## Gateway 配置

```toml
# gateway.toml — codec 设置默认值
# UI 可通过 URL 覆盖：?codec=json 或 ?codec=cbor（零 RTT）
# 编解码器在握手时固定——无运行时检测或更新。
# Kafka 始终 CBOR。
```

---

## 陷阱

- **`dyn Codec` 不是对象安全的**：trait 上的泛型方法不能与 `dyn` 一起使用。始终使用枚举分发（`ActiveCodec`）。
- **`gloo_net::websocket::Message::Bytes` 非 `Binary`**：在 WASM 中，`gloo_net` 使用 `Message::Bytes(Vec<u8>)` 表示二进制 WebSocket 帧，不是 `Message::Binary`（变体未找到，编译错误）。
- **axum 0.8 `Message::Text` 接受 `Utf8Bytes`**：`String` 不直接转换——使用 `.into()`。`Message::Text(s.into())` 有效；`Message::Text(s)` 无效。
- **`SessionManager` 包装在 `Arc` 中**：无法通过 Arc 获取 `&mut`。在包装器上添加 `get_mut(&self)` 以获取 `dashmap::mapref::one::RefMut` 进行可变访问。
- **WASM 构建因 Kafka 失败**：`rdkafka` 需要原生 C 库。在 `ui/Cargo.toml` 中对 `message` 使用 `default-features = false`。
- **Iggy 宏作用域**：`iggy_conn!` 宏引用 `#[cfg(feature = "iggy")]` 后的类型。宏调用也必须用相同 cfg 守卫。
- **Lint 误报**：项目使用 Rust 2024 特性（let chains、async fn）但某些 crate 可能报告"Rust 2015"lint。这些是预先存在的版本配置问题，非逻辑错误。
- **文档更新**：更新 README 时，**合并**新决策与现有决策。永远不要删除原始理由。
- **`ActiveCodec` 是 `Debug + Clone` 非 `Copy`**：不要尝试在 match 守卫中使用而不克隆。
- **Kafka 编解码器硬编码 CBOR**：无 `queue.codec` 配置字段。Kafka 生产者/消费者直接使用 `ciborium::ser::into_writer` 和 `ciborium::de::from_reader`。如未来需要 Avro 支持，这将是单独的设计工作。
- **Bincode 根因是 `deserialize_any` + 内部标记枚举**：bincode 失败的最深层原因是 `#[serde(tag = "...")]` 需要 `deserialize_any`，而 bincode 不支持。这影响 `Brick`/`Content` 类型的所有反序列化，不仅是线路协议。
- **URL 编解码器解析必须独立于 token**：在 `ui/src/main.rs` 中，`?codec=` 查询参数解析器错误嵌套在 `?token=` 分支内。如果 token 来自 `data-token` 属性而非 URL，编解码器解析完全被跳过。将编解码器解析为兄弟分支，不在 token 分支内。
- **`CodecHint` 冗余**：是 `CodecType` 的结构重复，带 `From` 转换和重复的编码逻辑。用 `Client` 上的 `CodecType` + 单个 `encode_ws()` 函数替代。不要创建 1:1 映射的并行枚举类型。
- **`Status::send` 必须尊重编解码器**：UI 的 `send()` 硬编码为 `ciborium::ser::into_writer` + `Message::Bytes`，始终发送 CBOR 而不管配置的编解码器。使用 `self.codec.encode()` 并匹配编解码器以选择 `Message::Text` vs `Message::Bytes`。
- **Greet 必须立即发送，不可延迟**：Greet 包含基础 UI 布局。延迟它以等待客户端首帧（用于编解码器检测）增加 1 RTT 并导致连接时白屏。使用 URL `?codec=` 参数替代——在 WS 升级时解析，零额外 RTT。
- **`handle_ws` 参数顺序**：`handle_ws(socket, outgo_tx, state, config, tmpls, codec, session)` — `codec` 是握手时确定的**最终**值（来自 URL 参数或配置默认），不是被覆盖的回退。不要添加动态检测逻辑。
- **文档和注释必须纯英文**：所有 ADR、README 章节、设计决策和源代码注释必须用英文。项目文档或代码注释中无中文。

---

## 交叉引用

本文档是 Fluxora 项目的完整设计，与以下详细分析形成完整的决策闭环：

- **[Aura 架构](aura-architecture.md)**：存算一体的现代分布式 Actor 引擎，Fluxora 未来可选接入的底层基础设施。
- **[反应式架构](reactive-architecture.md)**：Fluxora 是反应式架构的激进实践——UI 和协议层完全事件驱动，无 HTTP 请求-响应模型。
- **[Aura + Fluxora DevOps](aura-fluxora-devops.md)**：脚本即代码（Script-as-Code）的工程实践，含 GitOps 工作流、CI/CD 配置、脚本测试与调试。
- **[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md)**：Fjall + Arrow + Polars 全链路存算一体。

**统一的第一性原理**：不搞技术崇拜，不吃开源画的大饼，只看真实的硬件物理限制与团队生产力。
