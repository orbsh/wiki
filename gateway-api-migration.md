# Gateway API 迁移指南（Istio -> Envoy）

本指南记录从 Istio `VirtualService` 迁移到 Kubernetes Gateway API `HTTPRoute`（Envoy Gateway），以及稳定性的关键架构模式。

## 1. 资源映射（Istio vs Gateway API）

基于迁移提交 `04df3ace`。

| Istio VirtualService（旧）| Gateway API HTTPRoute（新）| 备注 |
|:---|:---|:---|
| `kind: VirtualService` | `kind: HTTPRoute` | 标准 K8s 资源 |
| `spec.hosts` | `spec.hostnames` | 重命名字段 |
| `spec.gateways` | `spec.parentRefs` | 指向 Gateway 资源 |
| `spec.http[].match[].uri.prefix` | `spec.rules[].matches[].path` | 需要显式 `type`（如 `PathPrefix`）|
| `spec.http[].route[].destination` | `spec.rules[].backendRefs` | 简化的服务引用 |
| `port.number: 8080` | `port: 8080` | 端口是直接整数 |

### 1.1. 详细字段变更

**ParentRefs（Gateway 绑定）：**
```yaml
# Istio
spec:
  gateways:
  - istio-system/web

# Gateway API
spec:
  parentRefs:
  - name: envoy-gateway
    namespace: envoy-gateway-system
    sectionName: http  # 或 'https' 取决于 Listener
```

**路径匹配：**
```yaml
# Istio
- match:
  - uri:
      prefix: /api

# Gateway API
- matches:
  - path:
      type: PathPrefix  # 必须显式：PathPrefix、Exact 或 PathRegularExpression
      value: /api
```

**后端路由：**
```yaml
# Istio
- route:
  - destination:
      host: my-service
      port:
        number: 80

# Gateway API
- backendRefs:
  - name: my-service
    port: 80
```

---

## 2. 关键 Gateway 设计模式

### 2.1. 64-Listener 限制（避免"Listener 爆炸"）

**问题**：Gateway API 规范强制每个 Gateway **64 个 listener** 的硬限制。

**反模式**：在 Helm values 中循环每个 URL/Service 创建单独的 Listener。这会生成 `Gateway-TLS-ServiceA-Url1`、`Gateway-TLS-ServiceB-Url2` 等，很快超过限制。

**解决方案**：**折叠 Listener**。

为每个协议（HTTP/HTTPS）定义单个**通配符 Listener** 接受所有流量，让 `HTTPRoute` 资源处理域名/路径路由。

**正确的 Gateway 模板逻辑：**
```yaml
# 不要在这里循环 URL！
# 只为 HTTPS 定义一个 listener
listeners:
- name: gw-https
  port: 443
  protocol: HTTPS
  allowedRoutes:
    namespaces:
      from: All
  tls:
    mode: Terminate
    certificateRefs: ...
```

### 2.2. 模板空值安全

使用 Helm 生成 `HTTPRoute` 资源时，确保不生成无效 YAML 如 `filters: null`。

*   **Filters**：只有存在实际过滤器（rewrite、redirect）时才包含 `filters` 块。
*   **BackendRefs**：只有定义后端时才包含 `backendRefs`。如果使用 `directResponse`（如 404/503）而无后端，省略 `backendRefs`。

```yaml
# 带守卫的正确用法
{{- if .backend }}
backendRefs:
- name: {{ .backend.name }}
  port: {{ .backend.port }}
{{- end }}
```

---

## 2.3. Gateway vs VirtualService：主机名耦合

**Istio 缺陷**：`VirtualService` 和 `Gateway` 资源都需要显式 `hosts` 列表。

这创建**紧耦合**和配置重复。要添加新域名，你必须修改基础设施层（`Gateway`）白名单和应用层（`VirtualService`）路由。这违反"关注点分离"。

**Gateway API 解决方案**：完全解耦。

*   **Gateway（基础设施）**：只定义 listener（端口、协议、TLS 证书）。它是**域名无关的**。它接受所有匹配 TLS SNI/端口的流量。
*   **HTTPRoute（应用）**：定义 `hostnames` 和路由规则。应用声明自己的域名而无需基础设施变更。

**结果**：你可以将 100 个应用与 100 个不同域名部署到同一 Gateway，Gateway 配置需要**零变更**。

---

## 3. 迁移清单

- [ ] 更新 `apiVersion` 为 `gateway.networking.k8s.io/v1`。
- [ ] 将 `kind` 改为 `HTTPRoute`。
- [ ] 将 `spec.hosts` 重命名为 `spec.hostnames`。
- [ ] 用指向 `envoy-gateway` 的 `spec.parentRefs` 替换 `spec.gateways`。
- [ ] 将逻辑从 `spec.http` 移到 `spec.rules`。
- [ ] 为路径匹配添加 `type: PathPrefix`。
- [ ] 将 `route[].destination` 改为 `backendRefs`（使用 `name` 和 `port`）。
- [ ] **验证**：确保 Gateway 没有按 URL 创建 listener。
- [ ] **验证**：确保 Helm 模板优雅处理缺失后端。
