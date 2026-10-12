# ECS 实体组件系统（以 Bevy 为例）

> **语言：** [English](entity-component-system-en.md)（主文档）· [中文](entity-component-system.md)
> 2026-08-12 创建

## 范式差异：ECS 相对 OO 是标签，不是树

ECS 与 OO 的分界线不在"有没有对象"，而在**数据与逻辑如何组织**。OO 用类定义把数据封装进对象、逻辑（方法）挂在对象上，身份由类决定；ECS 把数据拆成扁平的组件、逻辑（系统）与数据分离，身份由组件组合决定。结构上，这是一次**从树到标签**的迁移（完整论证见 [现代语言设计](modern-language-design.md)）：

- **OO（class 继承）＝ 树**：class 体系的结构特性在**边**——只跨级连接（父→子），同层不连接，一个类只能处在唯一继承链上，层层向上收敛到唯一根类（Java 的 Object）。class 是**命名**（标识主体"它是谁"），单一且唯一，身份来自它在这棵树上的命名位置。
- **ECS ＝ 标签**：entity 是纯标识（一句柄），component 是标签集合——**一个物体，很多标签**，每个标签描述一个属性/方面，可同时存在（`Player`、`Health`、`Transform`）、自由组合。组合数量是标签种类的**乘积**（一物多标签，即多维）。

**ECS 是 Rust 对 OO 的回应。** Rust 没有 class（详见 [现代语言设计](modern-language-design.md)），无类 + 标签化要落到"大量异质实体持续演化、彼此交互"的领域，就长出了 ECS——准确说，ECS 是"无 class/标签化"对**模拟/游戏**场景的回应。它在概念上与 OO 同级，是通用的数据+逻辑组织范式（agent 模拟、生态、物理、流行病等都能用）；但实践中它高度集中在游戏与高性能实时模拟。这不是偶然，而是**收益画像**决定的——ECS 的每一项优势都在为特定条件服务：

- **缓存局部性**：同类型组件连续存储（SoA），系统批量遍历时走内存顺序访问。只有实体数量大、系统成批扫同型组件时，缓存友好才转化为可感的性能。
- **无状态同步**：状态躺在组件里，系统 `Query` 直读，无需 getter/setter、通知、事件总线传值。数据一变全网可见，天然一致。
- **运行时改身份**：同一句柄增删组件即改类型（活的→尸体），不用换对象。
- **系统可并行**：调度器按系统声明的读写依赖，把无共享状态的系统并行跑。

这四项收益只在**同一个画像**下才划算：**实体数量大**（体积让缓存与并行有意义）、**性能敏感/实时**（帧预算逼你优化缓存与并行）、**实体类型运行时可变**（改身份有实际用途）。这正是游戏/模拟的画像。

离开这个画像，收益消失、成本显形：实体少，缓存局部性无所谓；类型稳定，运行时改身份用不上；非实时，没理由为并行付调度开销。而 ECS 的代价是恒定的——

- **间接性**：实体是匿名行键，要知道"它是什么"必须查组件；OO 的 class 直接告诉你。
- **失去封装**：行为散在作用于组件集的系统里，"方法属于对象"的局部性没了，对象内聚的逻辑更难维护。
- **查询开销**：每次访问都过组件存储，不如 OO 的直接字段访问。

业务 CRUD、低实体数、类型不变——OO 或普通数据结构更简单，ECS 不是普遍替代。

这个区分的本质是 **标签 = 倒排索引**：把"实体"和"分类"解耦——实体用唯一标识定位，分类用标签集合表达，查询走倒排索引求交集。ECS 的 `component` 与 trait/tag/label 是同一类东西：都是"挂在标识上的标签"；区别只是 ECS 的组件比纯标签多承载了数据（`Health(3)`、`Transform`），是"标签 + 数据"。

| ECS 里的角色 | 结构中的位置 |
|:--|:--|
| entity | 纯标识（没有内容，只是一句柄） |
| component | 标签 + 数据（`Player` 是纯标签，`Health(3)` 兼顾数据） |

系统按组件筛选实体（`Query`），本质就是倒排索引求交集——一个实体可同时带 `Player`/`Health`/`Transform` 多组标签，正是"一物多标签"，而非树。标签的组合方式也对应**参数多态**（实现接口，如 Rust 的 `T: A + B`），而非 OO 的**包含多态**（子类型/里氏替换）——这是又一个与 OO 在范式上的分岔。


## 场景树派 vs ECS 派：主流引擎的组织模式

上一节的"树 vs 标签"不是抽象对立，它精确对应主流开源引擎的组织模式选择。四个知名引擎恰好分两拨（数据经官网/README 核实于 2026-08）：

```
    树（继承/场景）                    标签（ECS）
 Godot ── fyrox ───────────────────── │ ── Ambient ─── Bevy
  组合场景树       继承树              │    ECS+实时库    纯 ECS
```

### fyrox —— 场景图节点继承树（树/继承派）

- 核心是 `Node` trait + 场景图继承层级：`Node` 为根，派生出 `RigidBody`、`Sprite`、`MeshNode`、`Light`、`Camera` 等节点类型，用 `TypeId` 注册，scene 反序列化时按类型还原。
- 行为靠 `Script` trait（类似组件，但挂在节点上，每帧 `on_update`），生命周期挂靠节点树。
- **不是 ECS**：选了"继承树 + 脚本"的组合模型，而不是数据驱动的 ECS。
- 自带场景编辑器，是"自带工具链的完整引擎"。
- 四者里最接近 **OO 树** 的一端：节点类型就是层次体系，身份由"在继承树上处于哪个节点类型 + 挂哪些脚本"共同决定。

### Godot —— 组合式场景树（树/组合派）

- 核心是 **Scene（场景）+ Node（节点）树**：一个游戏对象不是单个节点，而是一棵节点子树（`Sprite2D` + `CollisionShape2D` + `AudioStreamPlayer`…组合挂在一起）。
- 脚本（GDScript 的 `extends Node2D`）是**真实类继承**，但节点组合本身是**一物多标签**式的（往对象上挂各种功能子节点）——即便它用树承载，组合层面已带上标签的意味。
- 通信靠 **Signal（信号）**。
- **没有 ECS**——明确不用 ECS，靠"浅继承 + 节点组合 + 信号"，比 fyrox 的深层资源型继承要"平坦"。
- 核心引擎 C++，脚本可 GDScript/C#/Rust（gdext）。

### Ambient —— ECS 当"实时数据库"（标签派，已暂停）

- **纯 ECS 架构**：entity = 句柄，component = 数据标签，正是本章开头"标签（一物多标签）"那套。
- **关键现状：开发已无限期暂停**（README 顶部挂警告，团队转向 Braindump 项目）。历史架构仍有参考价值，但作为技术选型已无意义。
- 独特点：把 ECS 当作**权威的实时数据库**——服务器端持有全套 entity/component，客户端自动同步，网络层就是把 ECS 的变更同步出去。多人游戏的"状态同步"问题被"同步整个 ECS"消化掉了。这正呼应了后面"ECS 心智模型像数据库"的定位——Ambient 把这个比喻做成了字面架构。
- 游戏逻辑用 Rust/C/C# 编译到 WASM，在 runtime 外隔离执行。Rust + WASM + WebGPU，目标 web/desktop 多人游戏。

### Bevy —— 纯 ECS（标签派，最纯粹）

- `bevy_ecs` 自研 ECS，系统 + 组件是最底层心智，无内置编辑器（Bevy Editor 还在建设中），代码驱动 + 即时预览。
- 本文其余 Bevy API（`Query`/`Commands`/`Message`）全是这套。

### 一句话对比

| 引擎 | 语言 | 组织模式 | 编辑器 | 现状 |
|---|---|---|---|---|
| **Godot** | 核心 C++，脚本 GDScript/C#/Rust | 场景树(组合) + 信号 | 成熟 | 活跃 |
| **fyrox** | Rust | 场景图继承树 + 脚本 | 有 | 维护中 |
| **Ambient** | Rust (+WASM 逻辑) | ECS 当实时数据库 | 有(Studio) | **已暂停** |
| **Bevy** | Rust | 纯 ECS | 建设中 | 最活跃 |

**架构成语**：Bevy 是标签派的最纯粹形态；Ambient 是"标签派 + 网络同步即数据库"的激进尝试；Godot 是平坦的组合树；fyrox 是四者里唯一把"继承层级当一等公民"的树派（也最接近 OO）。

这个分布反过来印证了上一节：**ECS 不是游戏引擎的默认答案**。Bevy/Amethyst 同时代把 ECS 变成 Rust 游戏的刻板印象，但 Godot 和 fyrox 证明"场景树 + 脚本"在成熟产品路径上依然成立——增益画像决定立场：ECS 强在高性能批量模拟，树 + 脚本强在内容作者上手和编辑器的直接性。

**其他知名 Rust 引擎/框架**（按完整引擎 vs 库分）：完整引擎带场景/工具链——Amethyst/Legacy（specs 系 ECS 框架，基本搁置，历史意义上是早期把 ECS 带进 Rust 游戏的代表）；2D 游戏库无场景图、无编辑器、偏框架——macroquad、ggez、Tetra、nannou（创造性编码）；纯 ECS 库只解决"数据 + 系统组织"，可单独取用——`bevy_ecs`、`hecs`、`specs`、`legion`、`shipyard`。权威索引见 [Are We Game Yet](https://arewegameyet.rs/)（Ambient README 也指向它）。

## 定义：三层模型（数据库心智模型）

ECS 的心智模型不是对象树。对象树有两种，别混淆：一是 OO 的**类继承树**（单主体命名）——ECS 正是它的替代；二是引擎的**场景图**（`Parent`/`Child`），一棵用于空间寻址的独立树。ECS 的数据模型是**扁平的标签库，像数据库**：

- **实体（Entity）**＝ 行键：把多维数据联结成一个可寻址的单元，本身无内容
- **组件（Component）**＝ 可查询的维度（标签/列）：存数据或作身份标记
- **系统（System）**＝ 一条查询 + 对结果集统一执行的操作

这样 component 和 system 是**自然推导**出来的：世界是一张扁平的多维表，component 是你要查询的维度，system 是对"查询结果集"统一做的批操作。**实体是什么，完全由它身上的组件组合决定**，不来自任何类定义。

同一套结构，从两个方向看：

- **自上而下（引擎/数据）**：system 是作用在组件结果集上的查询；entity 是行键
- **自下而上（实体/游戏）**：entity 是一个存在，component 是它的属性，system 是施加于它的世界（重力、AI、伤害、渲染）

（entity 的"视口投影"角色：带 `Transform` + 渲染组件的实体，会把它的多维属性投影到 2D/3D 视口上，成为你看到的东西。但这只是实体的一种表现——不是所有实体都可见，计时器、相机、游戏状态这类纯逻辑实体没有视口投影。）

## 组件 vs 实体：数据 vs 组合

实体与组件是两个半面：

- **组件是纯粹的数据，不带行为**
- **实体是组件的一次组合**（一个句柄 + 一组组件）

```rust
#[derive(Component)]
struct Health(u32);   // 数据

#[derive(Component)]
struct Player;        // 空标记，用来标识身份

// 一个实体 = 句柄 + 组件集合
commands.spawn((Player, Health(3), Speed(3.0), Transform::from_xyz(0.0, 0.0, 0.0), Sprite::default()));
```

与面向对象的本质差异：OOP 里对象类型由类定义固定，ECS 里实体可以在**运行时通过增删组件改变身份**——同一个句柄，把 `Health` 移除、加一个 `Dead` 标记，它就从"活的"变成"尸体"，无需换对象。

## 为什么：数据与逻辑分离的收益

**数据与逻辑分离**是 ECS 收益的结构前提，不是"数据驱动"。"数据驱动"太宽泛——OO 相对命令式/过程式同样是围绕数据组织的（对象就是数据 + 方法），所以它不能把 ECS 和 OO 分开。真正的分野在**逻辑与数据的关系**：

- **OO**：逻辑（方法）绑在数据（对象）上，数据拥有自己的操作
- **ECS**：逻辑（系统）从数据（组件）剥离，系统作用于数据，数据只是被操作的状态

这个分离带来四个结构性收益（不是锦上添花）：

1. **无状态同步**：状态躺在组件里，任何系统 `Query` 到它就读到最新值——不需要 getter/setter、不需要通知、不需要事件总线传值。"状态同步"在这里成为伪问题，正是这个根因
2. **组合远多于继承**：想要"会飞的、会攻击的、红色敌人"，给实体拼几组组件即可，不用建一堆子类。组件种类的笛卡尔积就是实体种类，类型爆炸被化解
3. **缓存局部性**：同类型组件连续存储，系统批量遍历时利用内存连续性
4. **显式依赖 = 可并行**：系统间的读写依赖由调度器显式声明，无共享状态的系统可安全并行

## Bevy：心智模型与主要 API（0.19）

**心智模型**：Bevy 把整个游戏做成一个 **App**——App 里有一个 ECS **World**（所有实体/组件/资源都住在里面），外加控制"系统何时运行"的 **Schedule**。你写的不是对象，而是**系统**：每个系统是一个函数，它声明自己要什么（`Query` 读哪些组件、要不要 `Commands`、要不要全局 `Res`），然后被调度到某个 Schedule 里反复跑。**数据在 World 里、逻辑在 System 里、时机在 Schedule 里**，三者分离。

核心类型与 ECS 角色的对应：

| Bevy 类型 | 角色 |
|:--|:--|
| `App` | 顶层容器：World + Schedules + Plugins 的装配 |
| `World` | 一个 ECS 世界：所有 Entity/Component/Resource 的存储 |
| `Entity` | 句柄（行键） |
| `Component` | 挂在实体上的数据块（`#[derive(Component)]`） |
| `Resource` | 不挂实体的全局单例数据（`#[derive(Resource)]`：`Time`、`AssetServer`、`Input`…） |
| `System` | 一个函数，参数声明依赖（`Query`/`Res`/`ResMut`/`Commands`/`MessageWriter`…） |
| `Query` | 按组件类型读/写实体（可加过滤 `With`/`Without`/`Changed`） |
| `Commands` | 延迟的实体操作（spawn/despawn/insert），帧末统一生效 |
| `Message` | 系统间一次性消息（0.19 取代旧 `Event`） |
| `Schedule` | 系统的运行时机（`Startup`/`Update`/`FixedUpdate`/自定义） |
| `Plugin` | 打包系统+资源的可复用模块（`DefaultPlugins` 或自定义 `Plugin`） |
| `State` | 应用状态机（菜单/游玩/暂停），按状态切换运行哪些系统 |

为什么说这是"心智模型"而非记忆题：**你从不写"对象方法"，你写系统**。给玩家加移动——写 `fn move_player(q: Query<&mut Transform, With<Player>>)`，它作用于所有带这两个组件的实体。加重力、加伤害——各写一个系统。每个系统只声明它要的组件，Bevy 调度器负责把它们接到正确时机、并在无冲突时并行跑。这就是"把游戏当数据库"在引擎里的落地。

最小骨架：

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)                     // 渲染/输入/时间等内置系统
        .init_resource::<Score>()                        // 注册全局资源
        .add_systems(Startup, setup)                     // 启动只跑一次
        .add_systems(Update, (move_player, update_score)) // 每帧跑
        .run();                                           // 启动 App
}

#[derive(Resource)]
struct Score(u32);

#[derive(Component)]
struct Player;

fn setup(mut cmds: Commands) {
    cmds.spawn((Player, Transform::default())); // 生成实体 = 一组组件
}

fn move_player(q: Query<&mut Transform, With<Player>>) {
    for mut t in &q { /* 每帧移动 */ }
}

fn update_score(mut s: ResMut<Score>, input: Res<ButtonInput<KeyCode>>) {
    if input.just_pressed(KeyCode::Space) { s.0 += 1; }
}
```

## Bevy 实践（0.19）

以下 API 全部经 Bevy 0.19.0 实测编译通过。

### 生命周期

生成、销毁、改类型都经 `Commands`（缓冲延迟执行，避免迭代中修改崩溃）：

```rust
let e = commands.spawn((Player, Health(3))).id();   // 生成，拿到实体句柄

commands.entity(e).despawn();               // 销毁（延迟）
commands.entity(e).insert(Dead);            // 运行时改身份
commands.entity(e).remove::<Health>();
```

### 状态共享与消息

- **本机状态**：`Query` 直读组件，天然一致，无需同步
- **跨系统一次性消息**：Bevy 0.19 用 `Message` 取代旧 `Event`

```rust
#[derive(Message)]
struct HitEvent { target: Entity, damage: u32 }

fn a(mut w: MessageWriter<HitEvent>, q: Query<Entity, With<Player>>) {
    let Ok(e) = q.single() else { return };     // e 来自查询
    w.write(HitEvent { target: e, damage: 1 });
}
fn b(mut r: MessageReader<HitEvent>) { for ev in r.read() { /* 消费 */ } }
```

### 反馈即组件

把"闪白""飘字"做成组件，受实体生命周期管理，可叠加、可被系统统一处理：

```rust
#[derive(Component)]
struct FlashColor { timer: Timer }

// 命中时插入，一个系统读它、调亮度、结束后移除
commands.entity(e).insert(FlashColor { timer: Timer::from_seconds(0.15, TimerMode::Once) });
```

## 交叉引用

- [现代语言设计](modern-language-design.md)：ECS 的 `component` 与 `trait` 同属"标签（一物多标签 = 倒排索引）"结构；组件组合对应参数多态而非 OO 的包含多态
- [AI 辅助视频制作与游戏引擎](ai-creative-production-workflow.md)：fabricario 管线中 Bevy 作为交互式出口的定位