# Agent Stack Roadmap（跨仓路线图）

> **语言**：中文（技术术语保留英文）。本文件是**跨仓总览**，不是各仓 PLAN 的替代——
> phase 细节、ADR 引用、实施偏差永远归各仓 `docs/PLAN.md`；这里只维护三件事：
> 依赖方向、每仓当前状态（链回各仓 PLAN）、跨仓裁决日志。
>
> **更新纪律**：状态行只有"正在动的仓"需要更新；落地完成的 phase 细节写进该仓
> PLAN 并在此标记完成 + 提交号。跨仓裁决（改名、形态变更、接口契约）必须记入
> 底部日志。

## 栈总览（谁依赖谁）

```
                ┌────────────────┐
                │  Prism（入口）  │  WS 网关：鉴权 + 解析 + turn 投递
                └───────┬────────┘
                        │ turn 事件（realm events）
                        ▼
┌────────────────┐   ┌────────────────┐    invoke    ┌────────────────┐
│ Krystallizer   │◄──┤ Gravity（turn） │◄─────────────┤ Aura（场域）    │
│ 记忆：会话+图   │   │ 取会话→跑→append │              │ actor 引擎      │
└────────────────┘   └───────┬────────┘              └───────┬────────┘
                             │ tool call（ctx.invoke）       │ spawn/托管
                             ▼                               ▼
                     ┌────────────────┐              ┌────────────────┐
                     │ Effector（执行）│              │ okm（存储底座）  │
                     └────────────────┘              └────────────────┘
```

- gravity 依赖 krystallizer（fetch/append）与 aura（booth 绑定后）
- prism 依赖 aura（turn 事件 = realm events）
- effector 被 aura 托管（bgi 进程 / 嵌入载体）
- k10r 与 AI 端点同类：外部依赖，URL 可达即可，不在栈图内
- okm 被全部依赖（git 依赖，跟随 main）

## 当前状态一览

| 仓 | 阶段 | 下一步 | 阻塞于 |
|---|---|---|---|
| [aura](../../../world/aura/docs/PLAN.md) | 事件面终态落地（Phase 4.13/4.17-4.19）；ctx.timer 落地 | A3（部署态运维）、A4（USAGE 扫尾，低优） | — |
| [krystallizer](../../../world/krystallizer/docs/PLAN.md) | Phase 2 最小切片 + mem-server HTTP 三端点落地 | Phase 2 后续切片（threads/cursors/策略引擎） | — |
| [gravity](../../../world/gravity/docs/PLAN.md) | Phase 0 落地（MemorySurface seam + 双 surface 契约测试） | Phase 1：provider booth + LLM 注入面 | — |
| [prism](../../../world/prism/docs/PLAN.md) | Phase 1 收尾（logout + server-side revocation 已落地） | turn verbs（gravity M-A 已解锁） | — |
| [effector](../../../world/effector/docs/PLAN.md) | pyo3 0.29 + 改名完成 | A4 USAGE 双语扫尾（低优） | — |
| [okm](../../../world/okm/docs/PLAN.md) | 全部 shipped phases 稳定 | 跟随消费者需求 | — |

**关键路径**：krystallizer channel log → gravity Phase 0/1 → prism turn verbs → gravity Milestone B（booth 绑定）

## 跨仓裁决日志

只记**跨仓影响**的裁决与事件；单仓内的设计决策在该仓 ADR/PLAN。

### 2026-10-09

- **executor 项目改名 probe → effector**。理由：aura = 场，executor = 场中把
  决策变成作用的部件，field/effector 意象轴胜出（actuator 的 robotics 驱动器
  语义偏窄且与 actor 无呼应）。三仓 + okm 文档全扫；`probe_field`（扫描路由
  机制名）保留。aura `a517789`、effector `79d278f`、prism `5eae08a`。
- **pyo3 0.25 → 0.29.3**（okm `68f3a6f` → effector → aura `513b720`）：
  python 载体在 CPython 3.14 本机首次全绿。附带修复：aura `EventRouter::on`
  空 key = Singleton（原 `Field("")` 把单例订阅的精确事件吞进死信）。
  本机注意：nix libpython 不在默认链接路径，link 需 `RUSTFLAGS="-L <python-LIBDIR>/lib"`。
- **prism P1/P2/P5**（`de07db0`）：nushell feature 退役（nu 骑 bgi 载体）；
  python 留 default features（P2 按"不解决"关闭）；`InstanceKey::Named` 单点构造。
- **prism logout 落地**（`60c5830`）：设备解绑 + 连接 auth 字段同臂清零。
- **HANDOFF 待办状态合并入 aura PLAN**（`156d1e2`）：待办快照表接替
  HANDOFF 的 A*/P\* 编号，HANDOFF 保留叙述职能。
- **aura ctx.timer 落地**（`4d15674`，ADR-0016 §3b）：`ctx_timer_register/cancel`
  宿主函数 + wheel 的 `CancelTarget` 拆分（修复 run_job 边界吞掉投递条目的
  隐患）；durable 层为 recorded residual（StateStore 已被 ADR-0026 §3 退役）。

### 2026-10-10

- **k10r 定位收窄：纯 HTTP 服务，与 aura 完全解耦**（ADR-0007 双语再修订已落地）。
  撤销 2026-10-09 的 bgi 裁决：k10r 不是 aura 公民——不注册摊位、不被 spawn、
  不占 ns、bgi 形态出局；aura 世界的公民是 gravity（Milestone B booth 绑定），
  k10r 与 AI 端点同类（外部依赖，URL 可达即契约）。ADR-0007 三访问模式收敛：
  HTTP（mem-server）为唯一跨进程形态；进程内 mem-core = 本地同机实现细节；
  FFI 保留。mem-server 从 deferred 转正；aura PLAN Phase 6.6（Storage Booth /
  NestStorage executor）随之出局。gravity 侧影响：MemorySurface trait（进程内/
  HTTP 两 impl）+ HTTP 模式（store_url）待落地。
