# 无状态认证方案选型

**结论**：无状态认证（Stateless Authentication）在特定场景下是正确的架构选择，但 JWT 不是正确的实现工具。PASETO、证书认证（Certificate-based Auth）、HMAC 签名在不同粒度上实现了真正的无状态。Session-based 认证在需要服务端控制力的场景下仍然是首选。

## 无状态的本质

无状态认证的核心承诺是：**服务端不需要存储任何认证状态**。验证一个请求只需要请求本身携带的信息 + 服务端持有的密钥/公钥。

```
请求 → 服务端提取凭据 → 本地验证（签名/证书链） → 放行或拒绝
```

这消除了共享状态存储（Redis/数据库 session store）的依赖，在分布式系统中有天然优势：水平扩展不需要 sticky session，服务重启不丢失认证状态。

**但这个承诺有代价**：服务端无法主动撤销一个已签发的凭据（除非引入外部状态——这恰恰打破了无状态的前提）。

## JWT：一个被误用的标准

JWT（JSON Web Tokens）试图解决的问题是对的：用签名的 token 替代服务端 session。但它在设计上存在根本缺陷。

### 规范层面的问题

1. **仅设计用于极短生命周期**（~5 分钟或更短）。JWT 规范明确说 token 应该很快过期。但实际用法几乎全是长生命周期 session token——这不在规范的设计意图内。

2. **算法协商是安全灾难**。JWT 的 `alg` 头允许客户端指定验证算法。攻击者可以把 `alg` 改成 `none`（无签名）或 `HS256`（用公开的 HMAC 密钥签名），服务器如果不做严格校验就会接受伪造的 token。[Paragonie 的分析](https://paragonie.com/blog/2017/03/jwt-json-web-tokens-is-bad-standard-that-everyone-should-avoid)详述了这个问题。

3. **payload 可读不可改**。JWT 的 payload 是 base64 编码而非加密。任何人都能解码看到内容。虽然签名保证了完整性，但**信息泄露**是另一个维度的风险。

### "无状态"是个谎言

JWT 号称无状态，但安全地使用 JWT 必须引入状态：

| 场景 | 无状态？ | 实际需要 |
|:---|:---|:---|
| Token 吊销（用户登出/密码泄露） | ❌ | 黑名单（Redis/DB） |
| Token 刷新 | ❌ | Refresh token 存储 |
| 权限变更 | ❌ | 重新签发 or 查库验证 |
| 并发登录控制 | ❌ | 服务端 session 记录 |

**结论**：JWT 在理论上是无状态的，在实践中不得不引入状态来弥补安全缺陷。既然必须有状态存储，不如直接用 session-based 方案——更简单、更安全、更灵活。

### Google 的真实做法

Google 在浏览器用户 session 中**不用 JWT**。JWT 只用于 SSO 跨域传输（login session 从一个 host 转移到另一个 host）。他们的 JWT 有专门的安全团队维护，和普通开发者的 JWT 实现完全不同。

## PASETO：JWT 的正确替代

PASETO（Platform-Accountable Security Tokens）是为了解决 JWT 的设计缺陷而诞生的。

### 核心设计差异

| 特性 | JWT | PASETO |
|:---|:---|:---|
| 算法选择 | 客户端指定（`alg` 头） | 服务端决定（版本号隐含） |
| 签名算法 | 多种，含不安全选项 | 仅安全算法（Ed25519, RSA-PSS, AES-Poly1305） |
| Header | 可自定义（攻击面） | 固定结构（零攻击面） |
| 过期管理 | 可选 claim | 强制本地过期字段 |

PASETO 的哲学是：**不给开发者犯错的机会**。没有 `alg` 协商，没有可选的不安全算法，没有自定义 header。

### 局限

PASETO 仍然是 token-based 认证。它解决了 JWT 的安全设计缺陷，但没有解决 token-based 认证的根本问题：**服务端无法主动撤销**（除非引入状态）。

适用场景：短期 token（API 请求签名、临时授权码）、跨服务 token 传递。

## 证书认证：天然的无状态

证书认证（Certificate-based Authentication）是真正意义上的无状态——验证只需要证书链 + CA 公钥，不需要任何外部状态。

### mTLS（双向 TLS）

客户端和服务器互相验证证书。常见于：
- **服务间通信**（service mesh，如 Istio/Linkerd）
- **API 认证**（比 API Key 更安全）
- **Kubernetes API Server 认证**

```
客户端出示证书 → 服务端验证证书链 → 服务端出示自己的证书 → 双向信任建立
```

无状态的本质：验证过程完全是密码学的，不需要查库、不需要 session、不需要 token 存储。证书的吊销通过 CRL（Certificate Revocation List）或 OCSP 处理——这是在 CA 层面引入状态，而非认证层面。

### SSH 证书

SSH 证书（不是 SSH key）是另一个被低估的无状态认证方案：

- CA 签发短期证书（几小时到几天）
- `ssh-keygen -s ca_key -I user_id -V +8h user_key.pub`
- 服务端只需信任 CA 公钥（`TrustedUserCAKeys` in sshd_config）
- 证书过期自动失效，不需要吊销列表

**对比传统 SSH key**：key 无法过期、无法吊销（除非改 authorized_keys），多人共享 key 时无法审计。证书解决了所有这些问题。

### 适用场景

证书认证最适合：
- **基础设施层**（SSH、服务间通信）
- **高安全要求的 API**（金融、医疗）
- **短期授权**（用证书的过期时间天然控制生命周期）

不适合：
- **浏览器端用户认证**（用户体验差，需要证书安装）
- **快速迭代的应用**（证书轮换的运维成本）

## 何时该用有状态方案

无状态不是万能的。以下场景 **session-based 认证更合适**：

- **需要服务端控制力**：强制登出、并发登录限制、权限实时变更
- **浏览器用户 session**：cookie session 的浏览器支持最成熟（CsrfToken、Secure、HttpOnly、SameSite）
- **简单应用**：不需要分布式 token 验证，一台服务器搞定

**session 不等于 cookie**。session 是认证状态的概念，cookie 只是传输载体。session ID 可以通过 cookie、Authorization header、hidden field 等多种方式传递。选择 session-based 方案时，transport 是独立的实现决策。

## 选型矩阵

| 方案 | 无状态 | 服务端控制力 | 复杂度 | 适用场景 |
|:---|:---|:---|:---|:---|
| Cookie Session | ❌ | ✅ 完全 | 低 | 浏览器应用（默认选择） |
| JWT | ⚠️ 理论上 | ❌ 极差 | 高（安全陷阱多） | **不推荐** |
| PASETO | ⚠️ 理论上 | ❌ 极差 | 中 | 短期 token、跨服务传递 |
| mTLS | ✅ | ❌ | 高 | 服务间通信、高安全 API |
| SSH 证书 | ✅ | ✅（CA 签发控制） | 中 | 基础设施访问 |
| API Key + HMAC | ✅ | ❌ | 低 | 机器对机器、webhook |

## 实践建议

1. **浏览器用户登录**：用 cookie session，不要用 JWT。简单、安全、浏览器原生支持。

2. **服务间认证**：用 mTLS 或 PASETO。无状态，水平扩展友好。

3. **基础设施访问**：用 SSH 证书替代 SSH key。短期有效，自动过期，可审计。

4. **临时授权**：用 PASETO token。比 JWT 安全，比证书轻量。

5. **Webhook 验证**：用 HMAC 签名。简单、无状态、双方都可验证。

**不要因为"现代"或"流行"而选择 JWT。** 选择方案的标准是：它是否解决了你的具体问题，而不是它在 GitHub 上有多少 star。

---

## 交叉引用表

| 文档 | 关联 |
|:---|:---|
| `architecture-choices.md` | 认证是架构选型的一部分 |
| `ai-friendly-infrastructure.md` | 声明式配置与证书管理 |
