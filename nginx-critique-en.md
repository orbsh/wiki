# Nginx: The Myth of the "High-Performance Gateway"

> **Languages:** [English](nginx-critique-en.md) (primary) · [中文](nginx-critique.md)

**Conclusion**: Obsolete in modern architectures.  
**Status**: Mediocre performance, primitive configuration, design assumptions no longer hold.

## Executive Summary

Nginx is often praised as "high-performance," but this is a **myth from 2004**. In modern cloud-native environments, compared with alternatives such as Envoy or Pingora, Nginx's performance is **mediocre**. Worse still, Nginx's configuration model is **primitive** — static files, a non-standard format, no API-driven management — making it incompatible with modern orchestration systems.

**Core insight**: Nginx's design assumptions (saving every KB of memory, minimizing connections, manual tuning) were reasonable in an era when servers had only 512MB RAM and handled 10,000 requests/second. Today, a single JavaScript bundle exceeds 1MB, servers have 128GB RAM, and we handle millions of requests/second. Nginx's "optimizations" are **solutions to problems that no longer exist**.

## The Black Marks

### 1. The Performance Myth

**Claim**: "Nginx is the fastest web server."

**Reality**: In 2004 this was true. In 2026 it is not.

Nginx's event-driven model was revolutionary at the time — far more efficient than Apache's process-per-connection model. But modern alternatives use the same architectural idea, combined with more modern languages and runtimes, so their performance can only be better, never worse.

**Cloudflare's empirical evidence** (after replacing Nginx with Pingora):
- Resource consumption dropped to **1/3** of Nginx's (CPU + memory)
- Median TTFB reduced by **5ms**, P95 reduced by **80ms**
- New connections dropped to **1/3** of Nginx's (connection reuse rate rose from 87.1% to 99.92%)
- Saves users **434 years** of handshake time every day

The performance advantage comes from two layers: **architecturally**, multithreading + work stealing achieves global connection sharing, whereas Nginx's multi-process model leads to connection pool isolation (each worker has its own connection pool), a flaw that cannot be fixed at the architectural level. **Linguistically**, Rust + Tokio is naturally superior to C + fork — zero-cost abstractions eliminate runtime overhead, async/await natively supports high concurrency, and there are no GC pauses. Combined, Pingora reduces resource consumption to 1/3 of Nginx's.

**Nginx's structural bottlenecks**:
- **Multi-process overhead**: each worker process duplicates the configuration, SSL context, and connection pool. 16 worker processes = 16 copies of the same data.
- **Connection pool isolation**: Nginx's connection pool is per-worker. Increasing the number of workers lowers the connection reuse rate — this is an architectural flaw.
- **No work stealing**: Nginx's requests are bound to a fixed worker and cannot be load balanced across cores.

### 2. No Hot Reload or API-Driven Management

**Problem**: Nginx configuration is **static files**. Changing routes requires:
1. Edit the configuration file
2. Test the configuration (`nginx -t`)
3. Reload Nginx (`nginx -s reload` or `kill -HUP`)

**Consequences**:
- **Connection drops**: during a reload, in-flight connections may be discarded
- **Latency spikes**: a reload causes new connections to be queued for a brief period
- **No atomic updates**: complex route changes require multiple reloads
- **No API-driven configuration**: modern systems (Kubernetes, service mesh) expect dynamic configuration via API/gRPC

**Modern alternatives**:
- **Envoy**: the xDS protocol for dynamic configuration (CDS, EDS, RDS, LDS). Routes update in real time without reloads.
- **Pingora**: Rust-based, supports hot reload and programmatic configuration
- **Caddy**: file-based configuration, automatically reloads on change

**The irony**: Nginx Plus (the commercial version) added some dynamic configuration features, but it is **still file-based** and requires reloads. The open-source version has **zero** dynamic configuration support.

**Conclusion**: Nginx's static configuration adds operational friction in cloud-native environments. It is incompatible with GitOps, CI/CD pipelines, and service mesh architectures.

### 3. Configuration Experience: Non-Standard Format and Over-Tuning

#### 3.1 Non-Standard Format: Cannot Be Processed by the Toolchain

Nginx uses a custom configuration format — not JSON, not YAML, not TOML. It cannot be parsed by standard tools, there is no schema validation, no programmatic generation, no IDE support. A simple reverse proxy:

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

The same configuration in standard YAML (Envoy):

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

Yes, Envoy is more verbose. But it is standard YAML — parseable by any tool, programmatically generatable, schema-validatable. Nginx's custom format means every operator becomes an "Nginx configuration engineer" — manual editing, manual testing, manual reloading. Envoy's YAML is generated automatically by the control plane, so users never need to hand-write it.

#### 3.2 Resource Tuning: Scrimping KBs in a GB World

Nginx's configuration is so granular that every buffer and timeout has to be set manually. This is a relic of the 90s, when memory was counted in MB and 1G RAM was considered enormous:

```nginx
# Memory tuning: saving a few KB per connection — does it matter?
client_body_buffer_size 16k;
client_header_buffer_size 1k;
large_client_header_buffers 4 8k;
worker_connections 1024;          # limits total connections
keepalive_timeout 65;             # quickly closes idle connections
keepalive_requests 100;           # limits requests per connection
```

```nginx
# Timeout tuning: configured manually for every location
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;

# WebSocket requires separate tuning
location /ws {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_read_timeout 3600s;     # long connections need long timeouts
}
```

**The heart of the problem**: modern servers have 128GB RAM, and a single JavaScript bundle exceeds 1MB. The 16KB buffer that Nginx spends effort optimizing is less than even a fraction of a single request body. These configuration items were reasonable in 2004 — when servers had only 512MB RAM. In 2026, they are **noise**.

Different applications need different timeouts (API 30s, WebSocket 3600s, file upload 600s), and Nginx requires configuring this manually for every location; one wrong timeout = dropped connections or resource exhaustion. Event-driven asynchronous I/O (epoll/kqueue) is inherently good at handling slow connections — an idle fd does not block the event loop, and the connection count is not a bottleneck. The real problem with Nginx timeout tuning is not resource exhaustion, but **configuration complexity**: each location must be set separately, each application tuned one by one, making operational costs high. Envoy/Pingora's approach is **lenient defaults + declarative overrides**: the default timeouts are lenient enough, and when precise control is needed, it is configured per route/cluster through the control plane, without manually changing each location.

#### 3.3 Missing Control Plane/Data Plane Separation

**Problem**: Nginx's configuration is **static files**, with no architectural separation of control plane and data plane. Changing routes requires editing files, testing, and reloading — every change is a manual operation.

**Envoy's architecture**: Envoy is the **data plane** — users do not write YAML configuration directly. The control plane (Istio, Consul, a custom xDS server) generates configuration via an API and pushes it to Envoy. Configuration complexity is absorbed by the control plane, and the data plane is only responsible for execution.

**Nginx lacks this layering**: Nginx has no native concept of a control plane. Nginx Plus (the commercial version) added some dynamic configuration features, but it is still file-based and requires reloads. The open-source version has zero dynamic configuration support. The result: every operator becomes an "Nginx configuration engineer" — manually editing files, manually testing, manually reloading.

**The irony**: Nginx's custom configuration format (not JSON/YAML/TOML) was a reasonable design choice in 2004. But in 2026, it means it cannot be parsed by standard tools, there is no schema validation, no programmatic generation, no IDE support. Envoy's YAML, though verbose, is a standard format and can be generated automatically by the control plane — users never need to hand-write it.

### 4. Separation of Roles: Gateway, Web Container, Backend

Nginx's problems need to be broken down by role, because the requirements of different roles are completely different.

#### Gateway Layer: Nginx Is Not on the Shortlist

The choice for a modern API gateway is Envoy (data plane + control plane separation, xDS protocol) or Traefik (automatic service discovery, Kubernetes-native). Nginx has no control plane/data plane separation and is not suitable for this role.

#### Web Container Layer: Dedicated Tools such as ferron/reproxy Are Better

If you need a static file server + reverse proxy (PHP-FPM, frontend build artifacts), dedicated tools such as ferron (Rust) and reproxy (Go) are no worse than Nginx in performance, but the convenience of configuration is worlds apart — zero configuration or declarative configuration, no need to hand-write nginx.conf. ferron also supports PHP (FastCGI proxy), is superior to Caddy in every respect, but is more niche than Caddy. Caddy also supports PHP, but it is somewhat bloated and slow.

#### Backend Layer: Modern Backends Do Not Need a Web Container

Rust/Go/Node.js backends come with their own HTTP servers (Actix, axum, net/http, Express). In the ecosystems of these languages, Nginx's role as a reverse proxy has been replaced by in-process middleware or sidecars. Only languages such as PHP-FPM, which have no built-in HTTP server, still need Nginx as a front end.

**Nginx's true niche**: reverse proxy for PHP-FPM + static file server. This is a shrinking market.

### 5. Missing Modern Features

**Nginx lacks key features of modern architectures**:

| Feature | Nginx | Envoy | Pingora | Caddy |
|------|-------|-------|---------|-------|
| **HTTP/3** | ⚠️ Experimental | ✅ Native | ✅ Native | ✅ Native |
| **gRPC** | ⚠️ Limited | ✅ Full support | ✅ Full support | ⚠️ Basic |
| **Service Discovery** | ❌ None | ✅ Native (xDS) | ❌ None | ❌ None |
| **Circuit Breaking** | ❌ None | ✅ Native | ✅ Native | ❌ None |
| **Distributed Tracing** | ❌ None | ✅ Native (OpenTelemetry) | ✅ Native | ❌ None |
| **Metrics** | ⚠️ Basic (access logs) | ✅ Prometheus | ✅ Prometheus | ⚠️ Basic |
| **Automatic HTTPS** | ❌ Manual | ❌ Manual | ❌ Manual | ✅ Let's Encrypt |
| **WebSocket** | ⚠️ Complex (timeout tuning) | ✅ Native | ✅ Native | ✅ Native |

**Nginx Ingress Controller**: the community edition has been archived, and Nginx officially has also released its own Ingress Controller — which precisely shows that Nginx is being replaced in the Kubernetes ecosystem. Envoy (via Istio/Gloo) and Traefik are the current mainstream Ingress solutions.

## Scenarios Where Nginx Still Makes Sense

Despite these criticisms, Nginx is still a reasonable choice in certain scenarios:

1. **Simple reverse proxy**: if you just need to proxy requests to a backend, Nginx is simple, stable, and well documented.
2. **Static file serving**: Nginx's static file serving is highly optimized and efficient.
3. **Legacy systems**: if you already have Nginx configuration and your team is familiar with it, the migration cost may exceed the benefit.
4. **Low-traffic sites**: for sites with <10,000 QPS, Nginx's inefficiency is negligible.
5. **PHP-FPM reverse proxy**: this is Nginx's last real niche. But even in this scenario, [ferron](https://github.com/ferronweb/ferron) (Rust, 2k★) also supports FastCGI proxying to PHP-FPM, with better performance and security, just a more niche community. Caddy also supports PHP, but it is somewhat bloated and slow compared to ferron.

## When to Migrate

Consider migrating to modern alternatives when:
- You need **dynamic configuration** (Kubernetes, service mesh)
- You need **high performance** (>100,000 QPS, low latency)
- You need **modern protocols** (HTTP/3, gRPC, WebSocket)
- You need **observability** (distributed tracing, metrics, logs)
- You need **advanced features** (circuit breaking, service discovery, rate limiting)
- You are building a **greenfield project** (no legacy constraints)

## Conclusion

Nginx is a **legacy tool** — it works, but it is not the right tool for modern architectures. Its "high performance" is a historical advantage from 2004, its configuration model lacks control plane/data plane separation, and its design assumptions no longer hold in modern hardware and cloud-native environments.

**Nginx's true niche**: reverse proxy for PHP-FPM + static file server. This is a shrinking market. For new projects, modern alternatives (Envoy, Pingora, Caddy) are superior in performance, observability, and cloud-native integration.

→ **Constructive alternatives**:
- **API gateway**: Envoy (data plane + control plane separation, xDS protocol, service mesh integration)
- **High-performance proxy**: Pingora (Rust-based, programmable)
- **Simple static sites**: Caddy (automatic HTTPS, zero configuration)
- **PHP-FPM reverse proxy**: ferron (Rust, 2k★) is lighter and more secure than Nginx, Caddy can do it too but is more bloated

---

## Cross-References

This document forms a complete decision loop with the following architectural analyses:

- **[Redis: the "Network RAM" Trap](redis-critique-en.md)**: Redis as the "Network RAM Trap," a 2000s legacy tool just like Nginx.
- **[MySQL Critique](mysql-critique-en.md)**: MySQL's SQL anti-patterns and sharding illusion share the same origin as Nginx's configuration complexity.
- **[Unified Data Layer](unified-data-layer-en.md)**: a modern multi-model database, philosophically aligned with Nginx's modern gateway alternatives (Envoy/Pingora).
- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: the selection philosophy of Rune/Steel/Koto is consistent with the Nginx vs Envoy selection — reject legacy design, embrace modern architecture.
- **[Flux Architecture](flux-architecture-en.md)**: the dynamic layer of reactive architecture depends on WebSocket push, and Nginx's WS proxy configuration complexity contrasts with the "push-first" pattern of reactive architecture.

**A unified first principle**: no technology worship, no swallowing the pie that open source paints, only real physical hardware limits and team productivity.
