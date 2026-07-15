# Nginx："高性能网关"的神话

**结论**：现代架构中已过时。  
**状态**：性能平庸，配置原始，设计假设不再成立。

## 执行摘要

Nginx 常被称赞为"高性能"，但这是 **2004 年的神话**。在现代云原生环境中，与 Envoy 或 Pingora 等替代方案相比，Nginx 的性能**平庸**。更糟糕的是，Nginx 的配置模型**原始**——静态文件、非标准格式、无 API 驱动管理——使其与现代编排系统不兼容。

**核心洞察**：Nginx 的设计假设（节省每一 KB 内存、最小化连接、手动调优）在服务器只有 512MB RAM、处理 1 万请求/秒的时代是合理的。今天，单个 JavaScript 包超过 1MB，服务器有 128GB RAM，我们处理数百万请求/秒。Nginx 的"优化"是**已不存在问题的解决方案**。

## 黑点

### 1. 性能神话

**声称**："Nginx 是最快的 Web 服务器。"

**现实**：2004 年这是真的。2026 年不是。

Nginx 的事件驱动模型在当年是革命性的——比 Apache 的每连接进程模型高效得多。但现代替代方案用同样的架构思路，配合更现代的语言和运行时，性能只会更好不会更差。

**Cloudflare 的实证**（Pingora 替换 Nginx 后）：
- 资源消耗降至 Nginx 的 **1/3**（CPU + 内存）
- 中位数 TTFB 降低 **5ms**，P95 降低 **80ms**
- 新连接数降至 Nginx 的 **1/3**（连接复用率从 87.1% 提升到 99.92%）
- 每天为用户节省 **434 年**的握手时间

性能优势来自两层：**架构上**，多线程 + work stealing 实现全局连接共享，Nginx 的多进程模型导致连接池隔离（每个 worker 独立连接池），这是架构层面无法修复的缺陷。**语言上**，Rust + Tokio 天然优于 C + fork——零成本抽象消除运行时开销，async/await 原生支持高并发，无 GC 停顿。两者叠加，Pingora 在资源消耗上降到 Nginx 的 1/3。

**Nginx 的结构性瓶颈**：
- **多进程开销**：每个工作进程复制配置、SSL 上下文、连接池。16 个工作进程 = 同一数据的 16 份副本。
- **连接池隔离**：Nginx 的连接池是 per-worker 的。增加 worker 数量会降低连接复用率——这是架构层面的缺陷。
- **无 work stealing**：Nginx 的请求绑定到固定 worker，无法跨 core 负载均衡。

### 2. 无热重载或 API 驱动管理

**问题**：Nginx 配置是**静态文件**。更改路由需要：
1. 编辑配置文件
2. 测试配置（`nginx -t`）
3. 重载 Nginx（`nginx -s reload` 或 `kill -HUP`）

**后果**：
- **连接断开**：重载期间，进行中的连接可能被丢弃
- **延迟尖峰**：重载导致短暂时期新连接被排队
- **无原子更新**：复杂路由更改需要多次重载
- **无 API 驱动配置**：现代系统（Kubernetes、服务网格）期望通过 API/gRPC 进行动态配置

**现代替代方案**：
- **Envoy**：xDS 协议用于动态配置（CDS、EDS、RDS、LDS）。路由实时更新，无需重载。
- **Pingora**：基于 Rust，支持热重载和编程式配置
- **Caddy**：基于文件的配置，更改时自动重载

**讽刺**：Nginx Plus（商业版本）添加了一些动态配置功能，但它**仍然基于文件**且需要重载。开源版本**零**动态配置支持。

**结论**：Nginx 的静态配置在云原生环境中增加了运维摩擦。它与 GitOps、CI/CD 管道和服务网格架构不兼容。

### 3. 配置体验：非标准格式与过度调优

#### 3.1 非标准格式：无法被工具链处理

Nginx 使用自定义配置格式——不是 JSON、不是 YAML、不是 TOML。无法被标准工具解析，无模式验证，无编程生成，无 IDE 支持。一个简单反向代理：

```nginx
server {
    listen 80;
    server_name example.com;
    location /api {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

同样的配置用标准 YAML（Envoy）：

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 80 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["example.com"]
              routes:
              - match: { prefix: "/api" }
                route: { cluster: backend_cluster }
          http_filters:
          - name: envoy.filters.http.router
```

是的，Envoy 更冗长。但它是标准 YAML——可被任何工具解析、编程生成、模式验证。Nginx 的自定义格式意味着每个运维人员都成了"Nginx 配置工程师"——手动编辑、手动测试、手动重载。Envoy 的 YAML 由控制面自动生成，用户永远不需要手写。

#### 3.2 资源调优：在 GB 世界里抠 KB

Nginx 的配置细到每一个缓冲区和超时都要手动设置。这是 90 年代内存按 MB 算、1G RAM 算海量时的产物：

```nginx
# 内存调优：每个连接节省几 KB——有意义吗？
client_body_buffer_size 16k;
client_header_buffer_size 1k;
large_client_header_buffers 4 8k;
worker_connections 1024;          # 限制总连接数
keepalive_timeout 65;             # 快速关闭空闲连接
keepalive_requests 100;           # 限制每连接请求数
```

```nginx
# 超时调优：每个 location 手动配一遍
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

# WebSocket 需要单独调
location /ws {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_read_timeout 3600s;     # 长连接要长超时
}
```

**问题的核心**：现代服务器有 128GB RAM，一个 JavaScript 包超过 1MB。Nginx 花精力优化的 16KB buffer，还不如一个请求体的零头。这些配置项在 2004 年是合理的——当时服务器只有 512MB RAM。在 2026 年，它们是**噪音**。

不同应用需要不同的超时（API 30s、WebSocket 3600s、文件上传 600s），Nginx 要求为每个 location 手动配置，一个错误的超时 = 连接断开或资源耗尽。事件驱动异步 I/O（epoll/kqueue）本身擅长处理慢连接——空闲 fd 不阻塞事件循环，连接数不是瓶颈。Nginx 超时调优的真正问题不是资源耗尽，是**配置复杂性**：每个 location 要单独设，不同应用要逐个调，运维成本高。Envoy/Pingora 的做法是**宽松默认值 + 声明式覆盖**：默认超时足够宽松，需要精确控制时通过控制面按 route/cluster 配置，不需要逐个 location 手动改。

#### 3.3 缺少控制面/数据面分离

**问题**：Nginx 的配置是**静态文件**，没有控制面/数据面的架构分离。更改路由需要编辑文件、测试、重载——每次变更都是手动操作。

**Envoy 的架构**：Envoy 是**数据面**——用户不直接写 YAML 配置。控制面（Istio、Consul、自定义 xDS 服务器）通过 API 生成配置并推送给 Envoy。配置的复杂性被控制面吸收，数据面只负责执行。

**Nginx 缺少这个分层**：Nginx 没有原生的控制面概念。Nginx Plus（商业版）添加了一些动态配置功能，但仍然基于文件且需要重载。开源版本零动态配置支持。结果是：每个运维人员都成了"Nginx 配置工程师"——手动编辑文件、手动测试、手动重载。

**讽刺**：Nginx 的自定义配置格式（非 JSON/YAML/TOML）在 2004 年是合理的设计选择。但在 2026 年，它意味着无法被标准工具解析、无模式验证、无编程生成、无 IDE 支持。Envoy 的 YAML 虽然冗长，但它是标准格式，可被控制面自动生成——用户永远不需要手写。

### 4. 角色分离：网关、Web 容器、后端

Nginx 的问题需要按角色拆解，因为不同角色的需求完全不同。

#### 网关层：Nginx 不在候选名单

现代 API 网关的选择是 Envoy（数据面 + 控制面分离，xDS 协议）或 Traefik（自动服务发现，Kubernetes 原生）。Nginx 没有控制面/数据面分离，不适合这个角色。

#### Web 容器层：ferron/reproxy 等专用工具更优

如果需要一个静态文件服务器 + 反向代理（PHP-FPM、前端构建产物），ferron（Rust）和 reproxy（Go）等专用工具在性能上不逊于 Nginx，但配置方便性天差地别——零配置或声明式配置，不需要手写 nginx.conf。ferron 也支持 PHP（FastCGI 代理），各方面优于 Caddy，但比 Caddy 更小众。Caddy 也支持 PHP，但有些臃肿和慢。

#### 后端层：现代后端不需要 web 容器

Rust/Go/Node.js 后端自带 HTTP 服务器（Actix、axum、net/http、Express）。在这些语言的生态中，Nginx 作为反向代理的角色已经被进程内中间件或 sidecar 替代。只有 PHP-FPM 这类没有内置 HTTP 服务器的语言才需要 Nginx 作为前端。

**Nginx 的真实生态位**：PHP-FPM 的反向代理 + 静态文件服务器。这是一个正在缩小的市场。

### 5. 缺少现代特性

**Nginx 缺少现代架构的关键特性**：

| 特性 | Nginx | Envoy | Pingora | Caddy |
|------|-------|-------|---------|-------|
| **HTTP/3** | ⚠️ 实验性 | ✅ 原生 | ✅ 原生 | ✅ 原生 |
| **gRPC** | ⚠️ 有限 | ✅ 完整支持 | ✅ 完整支持 | ⚠️ 基础 |
| **服务发现** | ❌ 无 | ✅ 原生（xDS）| ❌ 无 | ❌ 无 |
| **熔断** | ❌ 无 | ✅ 原生 | ✅ 原生 | ❌ 无 |
| **分布式追踪** | ❌ 无 | ✅ 原生（OpenTelemetry）| ✅ 原生 | ❌ 无 |
| **指标** | ⚠️ 基础（访问日志）| ✅ Prometheus | ✅ Prometheus | ⚠️ 基础 |
| **自动 HTTPS** | ❌ 手动 | ❌ 手动 | ❌ 手动 | ✅ Let's Encrypt |
| **WebSocket** | ⚠️ 复杂（超时调优）| ✅ 原生 | ✅ 原生 | ✅ 原生 |

**Nginx Ingress Controller**：社区版已存档（archived），Nginx 官方也推出了自己的 Ingress Controller——这恰恰说明 Nginx 在 Kubernetes 生态中正在被替代。Envoy（通过 Istio/Gloo）和 Traefik 是当前主流的 Ingress 方案。

## Nginx 仍然有意义的场景

尽管有这些批评，Nginx 在某些场景中仍然是合理的选择：

1. **简单反向代理**：如果你只需要将请求代理到后端，Nginx 简单、稳定且文档完善。
2. **静态文件服务**：Nginx 的静态文件服务高度优化且高效。
3. **遗留系统**：如果你已经有 Nginx 配置且团队熟悉它，迁移成本可能超过收益。
4. **低流量站点**：对于 <1 万 QPS 的站点，Nginx 的低效可以忽略不计。
5. **PHP-FPM 反向代理**：这是 Nginx 最后的真实生态位。但即便是这个场景，[ferron](https://github.com/ferronweb/ferron)（Rust，2k★）也支持 FastCGI 代理到 PHP-FPM，性能和安全性更优，只是社区更小众。Caddy 也支持 PHP，但相比 ferron 有些臃肿和慢。

## 何时迁移

在以下情况下考虑迁移到现代替代方案：
- 你需要**动态配置**（Kubernetes、服务网格）
- 你需要**高性能**（>10 万 QPS、低延迟）
- 你需要**现代协议**（HTTP/3、gRPC、WebSocket）
- 你需要**可观测性**（分布式追踪、指标、日志）
- 你需要**高级特性**（熔断、服务发现、速率限制）
- 你在构建**全新项目**（无遗留约束）

## 结论

Nginx 是**遗留工具**——能用，但不是现代架构的正确工具。它的"高性能"是 2004 年的历史优势，它的配置模型缺少控制面/数据面分离，它的设计假设在现代硬件和云原生环境中不再成立。

**Nginx 的真实生态位**：PHP-FPM 的反向代理 + 静态文件服务器。这是一个正在缩小的市场。对于新项目，现代替代方案（Envoy、Pingora、Caddy）在性能、可观测性和云原生集成上都更优。

→ **建设性替代方案**：
- **API 网关**：Envoy（数据面 + 控制面分离，xDS 协议，服务网格集成）
- **高性能代理**：Pingora（基于 Rust，可编程）
- **简单静态站点**：Caddy（自动 HTTPS，零配置）
- **PHP-FPM 反向代理**：ferron（Rust，2k★）比 Nginx 更轻量安全，Caddy 也能做但更臃肿

---

## 交叉引用

本文档与以下架构分析形成完整的决策闭环：

- **[Redis 批判](redis-critique.md)**：Redis 作为"网络 RAM 陷阱"，与 Nginx 同为 2000 年代遗留工具。
- **[MySQL 批判](mysql-critique.md)**：MySQL 的 SQL 反模式和分片幻觉，与 Nginx 的配置复杂性同源。
- **[SurrealDB](unified-data-layer.md)**：现代多模型数据库，与 Nginx 的现代网关替代方案（Envoy/Pingora）哲学一致。
- **[嵌入式脚本语言选型](embedded-script-languages.md)**：Rune/Steel/Koto 的选型哲学与 Nginx vs Envoy 的选型一致——拒绝遗留设计，拥抱现代架构。
- **[反应式架构](reactive-architecture.md)**：反应式架构的动态层依赖 WebSocket 推送，Nginx 的 WS 代理配置复杂性与反应式架构的"推送优先"模式形成对比。

**统一的第一性原理**：不搞技术崇拜，不吃开源画的大饼，只看真实的硬件物理限制与团队生产力。
