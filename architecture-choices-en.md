# Architecture Choices and Critiques

> **Languages:** [English](architecture-choices-en.md) (primary) · [中文](architecture-choices.md)

## Container Gateway

**Core philosophy**: A cloud-native gateway must be dynamic, decoupled and protocol-agnostic. Nginx-based "hardcoding" is legacy and rejected.

### Envoy Gateway (standard)
*   **Role**: The reference implementation of the Kubernetes Gateway API.
*   **Why**: It decouples the control plane (Manager) and the data plane (Envoy Proxy). The cloud-provider standard.
*   **Architecture**: Two separate images: `envoyproxy/gateway` (controller) and `envoyproxy/envoy:distroless-v...` (data plane).

### APISIX / OpenResty (rejected)
*   **Critique**: Built on Nginx + a Lua VM. Because it depends on Nginx's multi-process model and Lua's state complexity, it is "anti-cloud-native".
*   **Conclusion**: A technical regression in the Rust/Go era. Considered only when extreme startup speed or low resource overhead is the *only* metric.

### Traefik / Kong
*   **Traefik**: Good developer experience (DX), auto-discovery, but weaker deep traffic-governance capability.
*   **Kong**: A heavyweight enterprise standard. Overkill when simple Gateway API support suffices.

### Pingora (watch list)
*   **Future potential**: Cloudflare's Rust proxy framework. Zero-copy, memory-safe, async.
*   **Relevance**: It shows the direction of high-performance networking (Rust > C/Nginx).

## Image Registry Strategy

**Core philosophy**: Network-level authentication > application-level RBAC. Simplicity > feature bloat.

### Zot + WireGuard (preferred)
*   **Why Zot**:
    *   **Features**: Native garbage collection, data-retention policies, object storage (S3) support.
    *   **Maintainability**: A single binary. It replaces a fragile OpenResty/Lua deletion script (`deletion.lua`) that was a maintenance nightmare.
*   **Security philosophy**:
    *   **Zero-auth design**: Humans rarely touch the registry directly. No auth/password means no HTTPS is needed.
    *   **Network boundary**: Since there is no application-level security, access is strictly limited to the internal network or through the WireGuard VPN.
*   **External cluster strategy**:
    *   **Independence**: Deploy a dedicated lightweight registry in each external cluster.
    *   **Offline capability**: The cluster keeps working even if the central link goes down.
    *   **Isolation**: Physical separation of customer images prevents leakage. No complex, distributed or P2P "super-registry" is needed.

### Harbor (rejected)
*   **Conclusion**: "Chocolate-flavoured shit" — glossy on the surface, hollow in architecture.
*   **Analysis**: See the full case study: [Harbor Critique](./harbor-critique-en.md)
*   **Summary**: CNCF graduation is a political victory, not a technical one. Pseudo-microservices, dependency hell and misplaced security make it a legacy burden.

## Database and Cache Positions

**Core philosophy**: Network RTT is the bottleneck, not query execution time.

### SurrealDB (preferred)
*   **Conclusion**: "The convergence of the data stack". A unified engine (KV + graph + relational).
*   **Analysis**: See the full analysis: [Unified Data Layer](./unified-data-layer-en.md)
*   **Summary**: It goes beyond PG + plugin bloat. SurrealQL offers Rust/Nu-like ergonomics and Turing completeness. Because of network latency, the performance gap against Redis is negligible.

### Redis (skeptical)
*   **Conclusion**: The "network RAM" trap. It imports high-latency overhead into what is a local-state problem.
*   **Analysis**: See the full critique: [Redis: the "Network RAM" Trap](./redis-critique-en.md) and [Nginx Critique](./nginx-critique-en.md)
*   **Summary**: A stopgap for PHP-FPM's statelessness. Network RTT dominates query time. A distributed cluster is just weakly consistent sharding.

### Preferred alternatives
*   **KV storage engine (Fjall + Raft)**: Embedded KV + Raft consensus — a concrete implementation. See [Aura Architecture §5](aura-architecture-en.md) and [KV Storage Engine](./kv-storage-engine-en.md).
*   **SpaceTimeDB**: Logic runs inside the database (Wasm), eliminating RPC/serialization overhead entirely.
*   **Aerospike**: High performance, hybrid memory/SSD, but it requires specific hardware.
*   **Local KV**: Fjall, Redb, RocksDB. "If you skip the network, a 0.1ms query is just fine".

## Common "Anti-Patterns"
*   **"PHP + Redis + MySQL + Nginx"**: A legacy stack. "Patch-style architecture": complexity added to work around the performance problems caused by a synchronous blocking model.
*   **Bloatware**: Projects that need Redis/MySQL merely to run (such as Harbor) are rejected when a simpler, self-contained alternative exists.

---

## Cross-references

This document is the overview of the architecture decisions; together with the detailed analyses below it closes the decision loop:

- **[Redis Critique](redis-critique-en.md)**: The detailed argument for why Redis is a "network RAM trap".
- **[Nginx Critique](nginx-critique-en.md)**: The detailed argument for why Nginx is a "relic of 2004".
- **[Harbor Critique](harbor-critique-en.md)**: The detailed argument for why Harbor is "a textbook case of enterprise bloat".
- **[Aliyun Critique](aliyun-critique-en.md)**: The detailed argument for Aliyun's strategic misjudgment.
- **[Unified Data Layer](unified-data-layer-en.md)**: A detailed analysis of SurrealDB's architecture philosophy as a unified data layer.
- **[MySQL Critique](mysql-critique-en.md)**: The detailed argument for MySQL's SQL anti-patterns and sharding illusion.
- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: A technical comparison of Rune/Steel/Koto.
- **[Editor Selection Analysis](editor-selection-2026-en.md)**: A deep analysis of Helix vs Neovim.

**Unified first principles**: No technology worship, no swallowing the pie that open source promises; only real physical hardware limits and team productivity matter.