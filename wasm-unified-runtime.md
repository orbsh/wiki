# Wasm 统一运行时：从编译目标到通用后端

**核心命题**：Wasm 长期被视为"第三方插件沙箱"——Rust/C++ 编译产物在隔离环境中运行。但 Wasm 正在从编译目标演变为通用语言后端。这个过程分两步：无 GC 语言（Rust/Go）现在已经能编译到 Wasm 运行；wasm-gc 落地后，需要 GC 的语言（Python/Kotlin/Swift 等）也能顺畅运行，统一运行时才真正成立。

## 双重身份：编译目标 vs 运行时

Wasm 当前有两种用法，常被混淆：

1. **编译目标**：Rust/C/C++/Go 编译到 `.wasm`，Wasm 作为可移植的二进制格式。宿主语言不变，Wasm 只是最终产物。
2. **运行时**：Wasm 作为通用执行环境，多种语言编译到 Wasm 后在同一运行时内运行。Wasm 不再是某一种语言的产物，而是所有语言的公共后端。

当前 Wasm 的角色以 (1) 为主。(2) 的成熟分两步：无 GC 语言现在已经可用（Rust 工具链最成熟）；需要 GC 的语言等 wasm-gc 落地。

## 两个语言梯队

Wasm 对语言的支持不是一个时间点，而是两个梯队：

### 第一梯队：现在已可用（不需要 wasm-gc）

- **Rust**：编写 Wasm 最完善的语言，没有之一。工具链成熟，产物质量高，`wasm32-wasi` target 稳定。门槛在于学习曲线和编译周期，不适合快速迭代。
- **C/C++**：理论可行，但语言本身（手动内存管理、ABI 复杂性、Undefined Behavior）拖后腿，工程体验远不如 Rust。
- **Go**：已能编译到 Wasm（`GOOS=wasip1`），但自带 GC 运行时打包进模块，体积偏大。能用，但不优雅——Go 的 GC 在 Wasm 线性内存里跑，没有利用运行时原生 GC。
- **Kotlin/Native**：AOT 编译路径可输出 Wasm，不走 JVM，不需要 wasm-gc。但生态受限。

第一梯队的语言现在就能覆盖"嵌入式脚本语言"的多数场景——Rust → Wasm 做异步、Go → Wasm 做工具调用，都已经可以跑。Rune 的异步场景现在就被 Rust → Wasm 覆盖，不需要等 wasm-gc。

### 第二梯队：需要 wasm-gc

- **Python**：动态语言，即使有 wasm-gc 也需要解释器（只是解释器可以用 wasm-gc 而非自带 GC）。过渡最慢。
- **Kotlin/JVM**：需要 GC，wasm-gc 落地后可直接编译，不用 AOT 也不用捆绑 JVM。
- **Swift**：ARC + 部分 GC 语义，wasm-gc 后可脱离 Apple 平台运行。
- **Java**：GraalVM 已有 Wasm 路径，wasm-gc 使其更自然。但语言冗长、JAR 部署模型与 Wasm 模块不匹配。
- **C#**：.NET 已有 Blazor（Wasm），wasm-gc 后效率提升（不再捆绑 .NET 运行时的 GC）。

第二梯队是 wasm-gc 的真正受益者——它们在 Wasm 上运行时不再需要自带完整运行时，性能惩罚消除。

## wasm-gc：转折点

wasm-gc（Garbage Collection proposal）让 Wasm 运行时原生支持 GC，而非依赖线性内存自行管理。

- **落地前**：第一梯队语言（Rust/Go/C++）已经能编译到 Wasm 运行。需要 GC 的语言只能自带完整运行时（如 Pyodide 捆绑 CPython）或走 AOT 丧失 eval 能力——在 Wasm 里跑了一个完整的解释器，性能惩罚没有消除。
- **落地后**：需要 GC 的语言可以编译到 Wasm 并共享运行时提供的 GC，不需要自带运行时。Wasm 对第二梯队语言真正可用。

wasm-gc 的意义不是"让更多语言能编译到 Wasm"——第一梯队早已做到。意义是**让需要 GC 的语言在不自带运行时的前提下运行在 Wasm 上**，从而消除"每种语言一个嵌入解释器"的碎片化格局。

## WASI：系统接口标准化

WASI（WebAssembly System Interface）是 Wasm 作为运行时的另一块拼图。没有 WASI，Wasm 模块无法访问文件、网络、时钟等系统资源，只能靠 Host 桥接——每个 Host 实现自己的 API，可移植性是假的。

WASI 标准化的目标：Wasm 模块编译一次，在任何符合 WASI 的运行时中运行，不需要感知底层操作系统。这与"统一后端"的方向一致——语言不关心自己跑在 Linux 还是 Windows，只关心 WASI 接口。

## wasm-gc 后的语言流行度推演

wasm-gc 落地后，第二梯队语言涌入 Wasm 生态。哪些会流行？判断依据：wasm-gc 投入度、语言表达能力、生态迁移成本。

| 语言 | wasm-gc 投入 | 语言表达能力 | 生态迁移成本 | 流行度预判 |
|------|-------------|-------------|-------------|-----------|
| **Kotlin** | 最早（Kotlin/Wasm 直接瞄准 wasm-gc） | 协程、简洁语法、类型安全 | JVM 生态平移路径清晰 | 高——JetBrains 先发优势 |
| **C#** | .NET NativeAOT + Blazor 积累 | 成熟类型系统、async/await | 企业迁移成本低 | 高——Microsoft 推动 |
| **Swift** | Apple 参与 Wasm GC 讨论 | 值语义、ARC、async/await | 脱离 Apple 平台后生态薄 | 中——语言强但生态受限 |
| **Python** | 不适用 | 过于动态，编译到 Wasm 极难甚至不可行 | 生态最庞大 | 不走 Wasm 路径——交互模式不同，PyO3 进程内嵌入是正确形态 |
| **Java** | GraalVM 路径 | 冗长，JAR 部署模型不匹配 Wasm 模块 | 历史包袱重 | 低——语言和部署模型都不适配 |
| **Wasm 原生语言** | MoonBit 等，从零开始 | 不受历史包袱约束 | 生态从零建 | 待定——取决于 wasm-gc 后语言设计爆发 |

**核心判断**：Kotlin 和 C# 最可能成为 wasm-gc 后的 Wasm 主力语言——两者都有大公司投入、现代语法、明确的迁移路径。Python 不走 Wasm 路径——它过于动态，编译到 Wasm 极难甚至不可行，但也不需要：PyO3 进程内嵌入开销低，交互模式（即时执行、无须编译）本身就是 Python 的价值，不是 Wasm 的退化。Java 的语言设计和部署模型都与 Wasm 模块范式不匹配。

Go 是一个特例：它已经在第一梯队（自带 GC 打包进 Wasm），wasm-gc 只是让它更高效（不用自带 GC）。但 Go 的简洁优势在 Wasm 生态中被 Kotlin 的表达力和 C# 的成熟度稀释——后两者更适合 wasm-gc 原生路径。

## Wasm 原生语言

除了主流语言编译到 Wasm，还有专门为 Wasm 设计的语言：

- **MoonBit**：面向 Wasm 的系统语言，语法简洁，编译快。初始设计锐利，但正在朝"什么都想要"的方向演化（标准 Wasm 环境的 WASI 兼容性、GC 支持问题，语言特性堆叠稀释设计锐度）。详见 [编程语言选型](language-selection.md)。
- **后续可能出现更多**：wasm-gc 落地后，设计 Wasm 原生语言的门槛降低——不需要自建 GC 和运行时，语言设计者可以专注语法和语义。

Wasm 原生语言的价值在于：不受历史包袱约束，从一开始就针对 Wasm 的执行模型优化。但当前还处于早期，MoonBit 是少数有实际产出的案例。

## 对嵌入式语言格局的影响

### Rune：异步已被覆盖，价值收窄到快速迭代

Rune（类 Rust 动态语言）的核心卖点是"原生 async/await + 进程内嵌入"。但 Rust → Wasm 现在就已经完全没有问题，Go → Wasm 也已可用——Rune 的异步场景不需要等 wasm-gc，现在就被覆盖。

Rune 仍然有价值的唯一理由：动态语言、即时执行、无需编译等待。Rust 的门槛在于学习曲线和编译周期，不适合快速迭代。wasm-gc 落地后，Kotlin/C# 这类语言也能顺畅跑在 Wasm 上，进一步压缩 Rune 的空间——Kotlin 的协程覆盖异步，语法比 Rust 易上手，编译速度也快于 Rust。

在 Aura 架构中，Rune 当前保留。随着 Rust → Wasm 生态成熟（现在已可用）和 wasm-gc 后 Kotlin/C# 涌入，Rune 降级为备选。快速迭代场景由 Python（PyO3）或 Steel 覆盖。

### Steel（Lisp）：持久价值

Steel 的核心价值不在运行时性能，而在 S-表达式的确定性边界——括号物理锁定作用域，无空格歧义。这个价值与 Wasm 无关：即使所有语言都能编译到 Wasm，策略引擎和 DSL 仍需要一种"语法即 AST"的语言。Lisp 的同构性（Homoiconicity）是 Wasm 不能替代的。

### PyO3（Python）：不需要迁移到 Wasm

Python 的核心价值是生态垄断（HuggingFace/PyTorch/Numpy）和交互模式——简单快速、无须编译、即时执行。总有场景需要这样的脚本语言，Python 是最合适的选择。

Python 编译到 Wasm 不是"过渡慢"的问题，而是**交互模式根本不同**。Python 过于动态（运行时修改类型、monkey-patch、eval/exec），静态编译到 Wasm 极难甚至实现不了。但这不是缺陷——Python 不需要编译到 Wasm，因为 PyO3 的进程内调用方式开销本身就低：零拷贝指针直传，没有序列化开销，没有模块边界。Wasm 运行时的沙箱隔离对 Python 来说是额外的墙，反而增加调用成本。

因此 PyO3 不受 wasm-gc 影响，不需要过渡——进程内嵌入 CPython 是 Python 在 Aura 架构中的正确形态，不是 Wasm 的退化替代。

### 统一运行时的愿景

wasm-gc + WASI 都成熟后，理想状态：

- **一种运行时**（Wasm）替代多种嵌入式解释器（Rune VM、Steel VM）——但 CPython 不在此列，Python 的交互模式与 Wasm 的编译模型是互补而非替代关系
- **多种语言**编译到同一运行时，共享 GC、内存安全、沙箱隔离
- **Host 只需对接 Wasm**，不需要为每种语言集成不同的运行时

这不是消灭语言多样性，而是消灭运行时碎片化。语言之间的差异回归到语法和语义层面，运行时层面统一。

## 时间线判断

| 阶段 | wasm-gc | WASI | Wasm 在嵌入式语言中的角色 |
|------|---------|------|-------------------------|
| **当前** | 提案推进中，部分实现 | 基础接口稳定，异步/网络不成熟 | 第一梯队（Rust/Go）已可用；Rune 异步场景已被覆盖，快速迭代仍需动态语言 |
| **近期**（1-2 年） | 主流运行时支持 | WASI 异步 I/O 成熟 | Rust → Wasm 生态成熟，Go → Wasm 可用；第二梯队语言仍需自带运行时 |
| **中期**（wasm-gc 落地） | Kotlin/C#/Swift 可原生编译 | 系统接口完整 | 第二梯队语言涌入，Rune 空间进一步压缩；Python 不受影响（不走 Wasm 路径） |
| **远期** | Wasm 原生语言成熟 | 生态稳定 | 统一运行时覆盖编译型语言；Python/Steel 作为交互模式语言保留 |

时间线不确定——wasm-gc 的落地速度取决于各运行时（Wasmtime/Wasmer/浏览器）的实现进度，WASI 的异步接口还在演进。但方向明确：Wasm 从"编译目标"向"通用后端"演进，是大一统趋势。

---

## 交叉引用

- **[嵌入式脚本语言选型](embedded-script-languages.md)**：Wasm 当前作为四种嵌入式选项之一（插件沙箱），本文档分析其演变为统一运行时的路径。
- **[现代编程语言设计](modern-language-design.md)**：Wasm 作为新后端，是复杂度守恒的又一例证——后端复杂度从每门语言下沉到统一运行时。
- **[Aura 架构](aura-architecture.md)**：Aura §6 的多语言运行时。Rust → Wasm 现在已覆盖 Rune 异步场景；wasm-gc 后 Kotlin/C# 涌入进一步压缩 Rune 空间。Steel 和 PyO3 保留（各有不可替代价值）。
- **[编程语言选型](language-selection.md)**：MoonBit 作为 Wasm 原生语言的案例分析和选型判断。
