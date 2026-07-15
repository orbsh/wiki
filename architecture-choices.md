# 架构选择与批判

## 容器网关

**核心哲学**：云原生网关必须是动态的、解耦的、协议无关的。基于 Nginx 的"硬编码"是遗留的，被拒绝。

### Envoy Gateway（标准）
*   **角色**：Kubernetes Gateway API 的参考实现。
*   **为何**：解耦控制平面（Manager）和数据平面（Envoy Proxy）。云提供商的标准。
*   **架构**：两个独立镜像：`envoyproxy/gateway`（控制器）和 `envoyproxy/envoy:distroless-v...`（数据平面）。

### APISIX / OpenResty（拒绝）
*   **批判**：基于 Nginx + Lua VM。由于依赖 Nginx 的多进程模型和 Lua 状态复杂性，"反云原生"。
*   **结论**：Rust/Go 时代的技术倒退。只有在极端启动速度或低资源开销是*唯一*指标时才考虑。

### Traefik / Kong
*   **Traefik**：良好的开发者体验（DX），自动发现，但深度流量治理能力较弱。
*   **Kong**：重型企业标准。如果简单的 Gateway API 支持就够用，则是大材小用。

### Pingora（观察名单）
*   **未来潜力**：Cloudflare 的 Rust 代理框架。零拷贝、内存安全、异步。
*   **相关性**：展示了高性能网络的方向（Rust > C/Nginx）。

## 镜像仓库策略

**核心哲学**：网络级认证 > 应用级 RBAC。简洁 > 功能膨胀。

### Zot + WireGuard（首选）
*   **为何 Zot**：
    *   **特性**：原生垃圾回收、数据保留策略、对象存储（S3）支持。
    *   **可维护性**：单二进制文件。替代了一个脆弱的 OpenResty/Lua 删除脚本（`deletion.lua`），那是维护噩梦。
*   **安全哲学**：
    *   **零认证设计**：人类很少直接触碰仓库。无认证/密码意味着不需要 HTTPS。
    *   **网络边界**：由于没有应用级安全，访问严格限制在内部或通过 WireGuard VPN。
*   **外部集群策略**：
    *   **独立性**：在每个外部集群部署专用的轻量级仓库。
    *   **离线能力**：即使中央链路断开，集群也能工作。
    *   **隔离**：客户镜像的物理分离防止泄露。不需要复杂的、分布式的或 P2P 的"超级仓库"。

### Harbor（拒绝）
*   **结论**："巧克力味的屎"——表面光鲜，架构空洞。
*   **分析**：详见完整案例研究：[harbor-critique.md](./harbor-critique.md)
*   **总结**：CNCF 毕业是政治胜利，不是技术胜利。伪微服务、依赖地狱和安全错位使其成为遗留负担。

## 数据库与缓存立场

**核心哲学**：网络 RTT 是瓶颈，不是查询执行时间。

### SurrealDB（首选）
*   **结论**："数据栈的收敛"。统一引擎（KV + 图 + 关系型）。
*   **分析**：详见完整分析：[unified-data-layer.md](./unified-data-layer.md)
*   **总结**：超越 PG+插件膨胀。SurQL 提供类 Rust/Nu 的人体工程学和图灵完备性。由于网络延迟，与 Redis 的性能差距可以忽略不计。

### Redis（怀疑）
*   **结论**："网络 RAM 陷阱"。为本地状态问题引入高延迟开销。
*   **分析**：详见完整批判：[redis-critique.md](./redis-critique.md) 和 [nginx-critique.md](./nginx-critique.md)
*   **总结**：PHP-FPM 无状态的权宜之计。网络 RTT 主导查询时间。分布式集群只是弱一致性的分片。

### 首选替代方案
*   **Fjall + Openraft**：嵌入式 KV + Raft 共识——具体实现。详见 [ADR](../fjall-openraft-adr.md) 和 [设计文档](./fjall-openraft-design.md)。
*   **SpaceTimeDB**：逻辑在数据库内运行（Wasm），完全消除 RPC/序列化开销。
*   **Aerospike**：高性能，混合内存/SSD，但需要特定硬件。
*   **本地 KV**：Fjall、Redb、RocksDB。"如果跳过网络，0.1ms 查询就很好"。

## 通用"反模式"
*   **"PHP + Redis + MySQL + Nginx"**：遗留技术栈。"补丁式架构"，为了解决同步阻塞模型引起的性能问题而增加复杂性。
*   **膨胀软件**：仅为了运行就需要 Redis/MySQL 的项目（如 Harbor），如果存在更简单、自包含的替代方案，则被拒绝。

---

## 交叉引用

本文档是架构决策的总览，与以下详细分析形成完整的决策闭环：

- **[Redis 批判](redis-critique.md)**：详细论证 Redis 为何是"网络 RAM 陷阱"。
- **[Nginx 批判](nginx-critique.md)**：详细论证 Nginx 为何是"2004 年的遗留物"。
- **[Harbor 批判](harbor-critique.md)**：详细论证 Harbor 为何是"企业膨胀的典型案例"。
- **[阿里云批判](aliyun-critique.md)**：详细论证阿里云的战略误判。
- **[SurrealDB](unified-data-layer.md)**：详细分析 SurrealDB 作为统一数据层的架构哲学。
- **[MySQL 批判](mysql-critique.md)**：详细论证 MySQL 的 SQL 反模式和分片幻觉。
- **[嵌入式脚本语言选型](embedded-script-languages.md)**：Rune/Steel/Koto 的技术对比。
- **[编辑器选型](editor-selection-2026.md)**：Helix vs Neovim 的深度分析。

**统一的第一性原理**：不搞技术崇拜，不吃开源画的大饼，只看真实的硬件物理限制与团队生产力。
