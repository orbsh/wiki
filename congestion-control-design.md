# 并发控制与背压设计

Embedding 生成场景（SurrealDB 透传调用 Ollama）的并发控制设计。核心原则：并发控制是**调用方的职责**（端到端原则），不是 DB 层的。

## 信号链架构

```
                    ┌─────────────────────────────┐
                    │        Ollama (瓶颈)          │
                    └──────────┬──────────────────┘
                               │ 过载 → 延迟上升 / 超时
                    ┌──────────▼──────────────────┐
                    │    SurrealDB (透传层)          │
                    │  http::post() 返回结果/错误    │
                    └──────────┬──────────────────┘
                               │ 延迟 + 错误码
                    ┌──────────▼──────────────────┐
                    │    Python 调用方               │
                    │                              │
                    │  信号采集 ──→ 决策引擎 ──→ 并发调节 │
                    │  (背压感知)   (AIMD)    (执行层)  │
                    └─────────────────────────────┘
```

## 信号层（背压感知）

| 信号 | 含义 | 动作 |
|:---|:---|:---|
| 响应延迟 < 目标阈值 T | Ollama 空闲 | 允许加性增 |
| 响应延迟 >= 目标阈值 T | Ollama 接近饱和 | 暂停增长，保持当前并发（软背压） |
| 超时 / 5xx | Ollama 过载 | 触发乘性减（硬背压） |
| 连续 K 次超时 | Ollama 瘫痪 | 进入冷却期（circuit breaker open） |

**软背压 vs 硬背压**：延迟超过阈值但未超时 = 软信号，告诉调用方"别加了"但不惩罚。这比等到超时再反应更平滑，避免震荡。

## 决策层（AIMD）

并发度 C，目标延迟 T（如 200ms）：

```
每次请求返回时：
  if 响应成功 && 延迟 < T:
      C += 1                    # 加性增：缓慢探测上限
  elif 响应成功 && 延迟 >= T:
      pass                     # 软背压：停止增长，但不惩罚
  elif 超时 or 5xx:
      C = max(C * 0.5, 1)     # 乘性减：快速退让
  elif 连续 K 次失败:
      C = 1; 进入冷却期 T_cool  # Circuit breaker open
```

**为什么组合优于单层**：

| 方案 | 缺陷 |
|:---|:---|
| 只有拥塞控制，没有背压 | 等到超时才知道过载，反应慢 |
| 只有背压，没有拥塞控制 | 知道过载但不知道退多少，要么退太多（浪费）要么退不够（震荡） |
| 组合 | 软信号（延迟）平滑调节 + 硬信号（错误）快速退让 + 冷却期防震荡 |

## 伪代码

```python
class EmbeddingCongestionController:
    def __init__(self, initial=5, target_latency_ms=200, cooldown_sec=10, failure_threshold=3):
        self.concurrency = initial
        self.target = target_latency_ms
        self.cooldown = cooldown_sec
        self.failure_threshold = failure_threshold
        self.consecutive_failures = 0
        self.next_probe_time = 0

    def acquire(self) -> bool:
        """是否允许发送新请求"""
        if time.now() < self.next_probe_time:
            return False  # 冷却期内拒绝
        return True

    def on_response(self, latency_ms, success):
        """每次请求返回后调用"""
        if not success or latency_ms is None:
            self.consecutive_failures += 1
            self.concurrency = max(self.concurrency * 0.5, 1)
            if self.consecutive_failures >= self.failure_threshold:
                self.next_probe_time = time.now() + self.cooldown
                self.consecutive_failures = 0
        else:
            self.consecutive_failures = 0
            if latency_ms < self.target:
                self.concurrency += 1
            # latency >= target: 不加不减（软背压信号）
```

## SurrealDB 的监控接口

SurrealDB 暴露的监控能力比 Python 自行埋点更直接：

| 接口 | 用途 | 优势 |
|:---|:---|:---|
| `/metrics` | Prometheus 格式指标（查询数、延迟、内存） | 标准协议，Grafana 直接接入 |
| `info()` 系统表 | 实时查询数、内存占用、运行时间 | 无需额外依赖，SQL 直接查询 |

**关键洞察**：Python 埋点观测的是"Python → SurrealDB"这段延迟，但瓶颈在"SurrealDB → Ollama"。直接读 SurrealDB 的 `/metrics`，观测的是端到端延迟，信号更准确。

## 背压的本质

让瓶颈节点（Ollama）的过载信号沿调用链反向传播。信号链是 `Ollama 过载 → SurrealDB 的 http::post() 超时 → Python 收到错误 → 退让`。Python 虽然不直接与 Ollama 通信，但因为 `fn::ollama::embed()` 是透传调用（无额外处理），SurrealDB 的响应延迟和错误直接反映 Ollama 的状态，信号失真极小。

DB 层做背压反而引入震荡：`Ollama 过载 → SurrealDB 感知 → SurrealDB 拒绝 → Python 重试 → SurrealDB 再次调用 Ollama → 打爆`。多一次往返，且重试请求又打回瓶颈节点。Python 端做背压只需一次往返就能完成退让。

## 交叉引用

- **[统一数据层](unified-data-layer.md)**：SurrealDB 计算下推架构与 Embedding 生成案例，本文档为其并发控制细节的展开。
