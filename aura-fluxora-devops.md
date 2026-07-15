# Aura + Fluxora DevOps：脚本即代码的工程实践

**状态**：架构设计完成
**日期**：2026-07-06
**核心哲学**：Script-as-Code — 嵌入式脚本与 Rust 代码同等对待，纳入 Git 管理

---

## 概述

当 Fluxora 从 Kafka 迁移到 Aura 后，业务逻辑从独立的 Rust 服务变为嵌入式脚本（Steel/PyO3/Wasm）。这带来了全新的工程管理挑战：

**核心问题**：
- 脚本如何版本控制？
- 如何测试？
- 如何部署？
- 如何回滚？
- 如何审计？

**解决方案**：脚本即代码（Script-as-Code）— 将嵌入式脚本视为一等公民，纳入 Git 仓库，通过 `set()` API 自动化管理。

→ 详见 [Aura 架构](aura-architecture.md)、[Fluxora 架构](projects/fluxora-architecture.md)

---

## 一、架构设计：脚本即代码（Script-as-Code）

### 1.1 脚本分类

| 类型 | 存储位置 | 更新方式 | 适用场景 |
|------|---------|---------|---------|
| **核心逻辑** | 编译进二进制 | 重新部署 | 稳定的基础功能 |
| **动态策略** | Fjall `actor_defs` 分区 | `set()` API 热部署 | 频繁变化的业务规则 |
| **用户脚本** | 用户上传 | Web UI | 自定义扩展 |

### 1.2 脚本持久化：Fjall 存储

脚本不再从文件系统读取，而是存储在 Fjall 的 `actor_defs` 分区中。Actor 定义本身是持久化、可复制的状态：

```
fjall partition: "actor_defs"
  key:   <actor_type_name>
  value: CBOR { lang, script_bytes, version, content_hash, committed_at, committed_by }
```

- **版本化**：Fjall 的 LSM-Tree 天然支持版本化，每次 `set()` 保留新版本，旧版本可回滚
- **去重**：`set()` 提交前先计算 `script_bytes` 的哈希，与最新版本比较——相同则忽略，不写入新版本
- **跨节点同步**：Openraft 自动将脚本同步到所有节点，不需要在每个节点上手动放置脚本文件
- **实例激活**：`on()` handler 在 Actor 实例激活时从 Fjall 读取最新版本脚本，加载到对应 VM 执行；实例驱逐后，下次激活重新读取

### 1.3 端点发现：`interface_schema()` 约定

**不需要清单文件（YAML manifest）**。每个脚本导出 `interface_schema()`，返回 receives/emits 声明：

```python
def interface_schema():
    return {
        "receives": {
            "add_to_cart": {
                "key": "user_id",
                "params": { "type": "object", "properties": { ... } },
                "returns": { "type": "object", "properties": { ... } }  # 可选
            }
        },
        "emits": {
            "cart_updated": {
                "schema": { "type": "object", "properties": { ... } }
            }
        }
    }
```

- **脚本即文档**：看到 `interface_schema()` 就知道这个 Actor 接收什么事件、发射什么事件
- **路由表构建**：Host 启动时调用 `interface_schema()`，构建事件路由表（事件名 → partition key → JSON Schema → Actor 定义 → on handler）
- **热重载入口**：脚本修改 → 重新加载 → 重新 `interface_schema()` → 路由表更新，无需重编译 Rust host
- **Rust actor**：`#[aura::schema]` 宏在编译期自动生成，不需要手写

### 1.4 `set()` API：热部署

```python
# 运行时提交或更新 Actor 定义
set("order_actor", lang="python", script=script_bytes)
```

`set()` 定义的是**类型**，不是实例。Actor 实例由 Arena 根据 partition key 按需激活。运行时调用 `set()` 可热替换 Actor 实现——不仅换行为，还换语言。

### 1.5 架构拓扑

```
[Fluxora 项目仓库]
├── ui/                    # UI 层（Brick DSL）
│   ├── templates/         # UI 模板
│   └── components/        # 组件定义
├── scripts/               # 业务逻辑脚本
│   ├── chat/              # 聊天业务
│   │   └── on_message.py  # Python 脚本（含 interface_schema()）
│   ├── crm/               # CRM 业务
│   │   └── on_lead.scm    # Steel Lisp 脚本
│   └── analysis/          # 分析业务
│       └── on_event.wasm  # Wasm 插件
├── tests/                 # 脚本测试
│   ├── chat_test.rs       # 使用 ArenaHarness
│   └── crm_test.rs
└── .github/workflows/     # CI/CD
    └── deploy-scripts.yml
```

---

## 二、GitOps 工作流

### 2.1 完整流程

```
[开发者修改脚本]
    ↓
[git commit]
    ↓
[GitHub Actions CI]
    ├── 1. 语法检查（linter per language）
    ├── 2. 单元测试（ArenaHarness 沙箱）
    ├── 3. 集成测试（模拟 Aura 环境）
    └── 4. 调用 set() API 部署脚本
    ↓
[PR 审查]
    ↓
[合并到 main]
    ↓
[ArgoCD 自动部署]
    ├── 1. 拉取最新脚本
    ├── 2. 调用 Aura set() API
    ├── 3. Aura 验证 interface_schema() 结构
    ├── 4. 脚本写入 Fjall（版本化）
    └── 5. Openraft 同步到所有节点
    ↓
[Aura 运行新脚本（热重载）]
```

### 2.2 CI/CD 配置示例

```yaml
# .github/workflows/deploy-scripts.yml
name: Deploy Scripts to Aura

on:
  push:
    branches: [main]
    paths: ['scripts/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Scripts
        run: |
          # Steel Lisp 脚本
          steel-lint scripts/**/*.scm
          # Python 脚本
          ruff check scripts/**/*.py

      - name: Run Script Tests
        run: |
          cargo test --features aura-test

      - name: Deploy to Aura
        run: |
          # 遍历 scripts/ 目录，逐个调用 set() API
          for f in scripts/**/*.py; do
            actor_name=$(basename "$f" .py)
            auractl arena set "$actor_name" python "$f"
          done
          for f in scripts/**/*.scm; do
            actor_name=$(basename "$f" .scm)
            auractl arena set "$actor_name" steel "$f"
          done
```

### 2.3 ArgoCD 配置

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - aura-deployment.yaml

configMapGenerator:
  - name: aura-config
    files:
      - config.toml

# 不同环境使用不同的脚本版本（通过 Aura API 控制）
patches:
  - target:
      kind: Deployment
      name: aura
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: "scripts-v1.2.3"  # 生产环境
```

---

## 三、脚本测试

### 3.1 ArenaHarness 测试框架

```rust
#[cfg(test)]
mod tests {
    use aura::test::ArenaHarness;

    #[aura::test]
    async fn test_chat_on_message() {
        let mut harness = ArenaHarness::new();

        // 加载脚本
        harness.set("chat_actor", "python", include_bytes!("../scripts/chat/on_message.py"));

        // 投递事件
        harness.emit("on_message", serde_json::json!({
            "user_id": "user-001",
            "message": "Hello"
        }));

        // 验证发射的事件
        let emitted = harness.collect_emits("send_response");
        assert_eq!(emitted.len(), 1);
        assert!(emitted[0]["response"].as_str().unwrap().contains("Hello"));
    }
}
```

### 3.2 多语言测试

ArenaHarness 不关心脚本语言——它通过 `set()` 加载脚本，通过 `emit()` 投递事件，通过 `collect_emits()` 验证结果。同一个测试可以覆盖 Python、Steel Lisp、Wasm 写的 Actor。

---

## 四、脚本依赖管理

### 4.1 共享库结构

```
scripts/
├── lib/                 # 共享库
│   ├── utils.py         # Python 工具函数
│   └── db.scm           # Steel Lisp 数据库操作
├── chat/
│   └── on_message.py    # import lib.utils
└── crm/
    └── on_lead.scm      # (import "lib/db")
```

### 4.2 依赖导入

```python
# scripts/chat/on_message.py
from lib.utils import log_info, send_response

def interface_schema():
    return { "receives": { ... }, "emits": { ... } }

def on_message(ctx, event):
    user_id = event["user_id"]
    log_info(f"Message from {user_id}")
    ctx.emit("send_response", {"user_id": user_id, "text": "Hello!"})
    ctx.commit()
```

### 4.3 依赖解析

脚本通过 `set()` 部署到 Fjall 时，Host 解析 import 语句，递归加载依赖到 Fjall。跨节点同步时，依赖关系一并复制。

---

## 五、脚本调试

### 5.1 本地开发

```bash
# 单文件启动——不需要 Docker、不需要 etcd、不需要数据库
$ aura dev order_actor.py

# 输出：
# [Aura] Actor order_actor 已启动
# [Aura] 场域: default
# [Aura] 状态: Fjall 本地模式 (./data/actors/)
# [Aura] 热重载: 监听文件变化，修改即生效
```

文件变化自动触发重新加载 → 重新 `interface_schema()` → 路由表更新。

### 5.2 追踪 API

```bash
# 启用追踪
curl -X POST https://aura.example.com/api/debug/trace \
  -d '{"actor": "chat_actor", "enabled": true}'

# 获取执行日志
curl https://aura.example.com/api/debug/log?actor=chat_actor
```

---

## 六、脚本回滚

### 6.1 版本管理

Fjall 的 LSM-Tree 保留每次 `set()` 的版本。回滚即指定旧版本号重新 `set()`：

```bash
# auractl 回滚
auractl arena rollback chat_actor --to-version 3

# 或通过 API
curl -X POST https://aura.example.com/api/actors/rollback \
  -d '{"actor": "chat_actor", "version": 3}'
```

### 6.2 版本历史

```bash
# 查看版本历史
auractl arena history chat_actor

# 输出：
# version 5  python  2026-07-06T15:30:00Z  deployed_by=alice  git=abc123
# version 4  python  2026-07-06T14:20:00Z  deployed_by=bob    git=def456
# version 3  steel   2026-07-05T10:00:00Z  deployed_by=alice  git=789abc
```

---

## 七、审计日志

### 7.1 变更记录

Fjall 中每次 `set()` 自动记录元数据（`committed_at`、`committed_by`、`version`）。结合 Git commit SHA，形成完整的审计链：

```bash
# 查询脚本变更历史
curl https://aura.example.com/api/audit/actors/chat_actor

# 响应示例
{
  "versions": [
    {
      "version": 5,
      "lang": "python",
      "committed_at": "2026-07-06T15:30:00Z",
      "committed_by": "alice@example.com",
      "git_commit": "abc123def456"
    },
    {
      "version": 4,
      "lang": "python",
      "committed_at": "2026-07-06T14:20:00Z",
      "committed_by": "bob@example.com",
      "git_commit": "def456ghi789"
    }
  ]
}
```

---

## 八、最佳实践

### 8.1 分支策略

```
main          ← 生产环境
  ↓
staging       ← 预发布环境
  ↓
feature/*     ← 功能分支
```

### 8.2 环境隔离

| 环境 | 脚本版本 | 部署方式 |
|------|---------|---------|
| **开发** | feature 分支 | `aura dev` 本地启动 |
| **测试** | staging 分支 | CI 自动 `set()` |
| **生产** | main 分支 | ArgoCD 自动 `set()` |

### 8.3 脚本质量门禁

```yaml
# .github/workflows/quality-gate.yml
name: Script Quality Gate

on:
  pull_request:
    paths: ['scripts/**']

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint
        run: |
          steel-lint scripts/**/*.scm
          ruff check scripts/**/*.py

      - name: Test
        run: cargo test --features aura-test

      - name: Validate interface_schema
        run: |
          # 加载脚本并验证 interface_schema() 返回有效结构
          auractl arena validate scripts/**/*.py
          auractl arena validate scripts/**/*.scm
```

### 8.4 脚本性能监控

```rust
// Aura 脚本性能指标
pub struct ScriptMetrics {
    pub execution_time: Duration,
    pub memory_usage: usize,
    pub error_count: u64,
}

impl ScriptMetrics {
    pub fn record(&self, actor_name: &str) {
        metrics::histogram!("script_execution_time", self.execution_time.as_millis() as f64, "actor" => actor_name);
        metrics::gauge!("script_memory_usage", self.memory_usage as f64, "actor" => actor_name);
        metrics::counter!("script_errors", self.error_count, "actor" => actor_name);
    }
}
```

---

## 九、工程管理挑战与解决方案

| 挑战 | 解决方案 |
|------|---------|
| **版本控制** | 脚本放在 Git 仓库，通过 commit SHA 版本化；Fjall 保留运行时版本 |
| **测试** | ArenaHarness 沙箱测试 + CI 自动化 |
| **依赖管理** | 共享库目录 + import 机制，部署时递归加载到 Fjall |
| **调试** | `aura dev` 文件监听热重载 + 追踪 API |
| **回滚** | Fjall LSM-Tree 版本化，`auractl arena rollback` 指定旧版本 |
| **GitOps** | ArgoCD 自动调用 `set()` + 环境隔离 |
| **审计** | Fjall 元数据 + Git commit SHA 双重追踪 |
| **跨节点同步** | Openraft 自动复制，无需手动部署脚本文件 |
| **端点发现** | `interface_schema()` 约定，无需 YAML 清单文件 |
| **性能** | Prometheus/Grafana 监控脚本执行指标 |

---

## 十、关键设计原则

1. **脚本即代码**：脚本和 Rust 代码同等对待，纳入 Git 管理
2. **CI/CD 自动化**：所有脚本变更必须通过 CI 测试
3. **热重载**：`set()` API 热部署，`interface_schema()` 热更新路由表，无需重启
4. **审计追踪**：Fjall 版本元数据 + Git commit SHA 双重记录
5. **环境隔离**：不同环境使用不同版本的脚本
6. **质量门禁**：脚本必须通过 lint、测试、`interface_schema()` 验证
7. **性能监控**：实时监控脚本执行时间和内存使用
8. **无清单文件**：`interface_schema()` 约定替代 YAML manifest，脚本自描述端点
9. **存储即复制**：脚本存 Fjall，Openraft 自动跨节点同步，无需手动部署

---

## 交叉引用

本文档是 Aura + Fluxora DevOps 的完整设计，与以下详细分析形成完整的决策闭环：

- **[Aura 架构](aura-architecture.md)**：存算一体的现代分布式 Actor 引擎。
- **[Fluxora 架构](projects/fluxora-architecture.md)**：事件驱动 UI 框架，含未来迁移到 Aura 的路径。
- **[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md)**：Fjall + Arrow + Polars 全链路存算一体。

**统一的第一性原理**：不搞技术崇拜，不吃开源画的大饼，只看真实的硬件物理限制与团队生产力。
