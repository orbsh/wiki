# Fluxora 架构决策记录（ADR）

## ADR-001：CBOR 优于 Bincode

- **决策**：所有 Kafka 队列流量使用 CBOR。拒绝 Bincode。
- **根因**：Bincode 在 `serde(tag = "...")`（内部标记枚举）上失败，因为它需要 `deserialize_any`，而 bincode 不支持。
- **影响**：影响 `Brick`/`Content` 类型的所有反序列化，不仅是线路协议。
- **替代路径**（2026-06-20 补充）：如果追求极致轻量，可以不切换格式，而是改用 Externally Tagged 默认枚举模式（变体用数字索引，不触发 `deserialize_any`）。Postcard 同样不支持 `#[serde(tag)]`，但配合 Externally Tagged 模式可安全使用。详见 [序列化协议抉择](serialization-protocol-decision.md#postcard-替代-bincode-的核心理由)。

## ADR-002：通过 URL 参数实现零 RTT 握手

- **决策**：编解码器在握手时基于 URL `?codec=json|cbor` 固定。
- **考虑的替代方案**：首帧检测（客户端发送信息，然后 Greet）。
- **拒绝原因**：Greet 包含基础 UI 布局；延迟它增加 1 RTT 并导致连接时白屏。URL 参数更干净且零 RTT。
- **陷阱**：不要使用 `CodecHint`（冗余）。不要延迟 greet。

## ADR-003：流式合并策略（`Vec<String>`）

- **决策**：将流式文本累积为 `Vec<String>` 片段，惰性展平。
- **考虑的替代方案**：`String`（push_str）或 `&[&str]`。
- **理由**：
  - `String`：每个 token 重新分配+复制整个缓冲区（O(N²)）。
  - `&[&str]`：生命周期地狱 + 内存放大（引用锁定完整原始片段）。
  - `Vec<String>`：摊销 O(1)，允许立即提取有效载荷，信封路由后丢弃。

## ADR-004：通过枚举进行编解码器分发（`ActiveCodec`）

- **决策**：使用 `enum ActiveCodec { Json, Cbor }` 而非 `Box<dyn Codec>`。
- **原因**：`dyn Codec` 不是对象安全的，因为 `encode<T: Serialize>` 是泛型方法。枚举分发避免 vtable 开销并支持 `Clone`。

## ADR-005：AI vs 模板使用

- **原则**：Minijinja 模板用于固定逻辑；AI 用于动态生成。
- **原子性**：保持 `Table` 和 `SVG` 原子（直接 HTML 映射）。
