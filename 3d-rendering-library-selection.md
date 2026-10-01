# 3D 渲染库选型：API 风格与 WASM 体积

可视化落库分两个方向。数据图表方向应选遵循图形语法的库（见 `grammar-of-graphics.md`），本篇不重复；本篇处理第二个方向——**自定义空间展示**：在 3D 空间里画网络拓扑、大规模点云、deck.gl 式的地理/关系场景。对比以 **Rust 侧为主体**（three-d、Bevy），three.js 与 deck.gl 作为 JS 传统方案的**参照系**——校准 API 风格坐标和体积基准，不是平行候选。主线是 API 风格，辅以实测体积。

**结论先行**：three-d 是唯一同时满足"API 不高不低、可嵌入宿主事件循环、wasm 体积可接受"的选择，实测净增量 +341KB（wasm-opt 后）/ +94KB（gzip），对体积敏感的 wasm 应用可行；Bevy 实测 18.9MB（wasm-opt 后），体积只是表象，根因是它垄断事件循环的游戏引擎形态与嵌入式渲染冲突；three.js 与 deck.gl 体积与 three-d 同数量级或更小，但停留在 JS 侧，编码不在 wasm 内。

## 一、选型标准：嵌入性是 dividing line

自定义空间展示的渲染库要挂进已有 UI（React/leptos/fluxen 的 canvas 元素），由宿主框架的 requestAnimationFrame 驱动每帧渲染。这条**嵌入性**标准把候选分成两类：

- **可嵌入**：库只提供"给一个 GL context 和场景数据，画一帧"的能力，循环归宿主。three.js、three-d、deck.gl 都可以（three-d 显式提供从外部 WebGL2 handle 构造 Context 的路径，绕开自带 Window 模块）。
- **不可嵌入**：库自己拥有主循环和事件系统。Bevy 的 `App::run()` 接管 winit 事件循环，与宿主框架争夺控制权——这不是配置问题，是引擎形态。硬接要拆 `MinimalPlugins` 加手动 schedule 驱动，等于逆流而上。

其余标准：API 不要太底层（不写裸 WebGL/glow 调用）也不要太高级（不引整个游戏引擎），支持 2D（正交相机、线、点、实例化图元是空间展示的基本盘）。

## 二、API 风格对比

四个库在"低层↔高层"轴上的位置：

```
裸 WebGL < glow ≈ three-d::context < three-d::core < three-d::renderer ≈ three.js < deck.gl 图层 < Bevy 引擎
（控制力递减，抽象递增）
```

三种心智模型的分裂点不在功能集合，在**每帧渲染时控制权怎么交出去**：

**three-d::renderer——无场景图，显式渲染调用**。没有 Scene 对象收纳一切；每帧把 camera、对象列表、灯光列表明确传给 render。类型系统强制你把渲染输入摆在明面上，没有"忘了一个 add 导致对象消失/残留"这类场景图状态陷阱：

```rust
// 从宿主已有的 canvas 建 context（嵌入路径，不碰 three-d 的 Window/winit）
let gl = canvas.get_context("webgl2")?.dyn_into::<WebGl2RenderingContext>()?;
let ctx = three_d::core::Context::from_gl_context(Arc::new(glow::Context::from_webgl2_context(gl)))?;

// CPU 数据 → GPU（数据变了才重做）
let model = Gm::new(Mesh::new(&ctx, &cpu_mesh), ColorMaterial::default());

// 宿主 rAF 回调里，每帧显式 render
let rt = RenderTarget::screen(&ctx, w, h);
rt.clear(ClearState::color_and_depth(1., 1., 1., 1., 1.))
  .render(&camera, [&model], &[]);   // &[] = 无灯光；ColorMaterial 不吃光，顺带省掉灯光管线代码
```

2D 支持是内建的：`Camera::new_orthographic` 或 `Camera2D`、`Line`/`Particles`/`Circle` 几何、`Control2D`（平移/缩放手势），点云直接 `Positions::F32` 进 `Particles`，不用为 2D 换库。自定义投影走 `Viewer` trait——只实现 `view()/projection()/viewport()`，就能把 3→2 投影数据塞进去而不引灯光层。

**three.js——场景图，命令式**。`scene.add(mesh)` 建立隐式的全局状态，渲染是 `renderer.render(scene, camera)` 对这幅状态快照求值。API 面向对象（BoxGeometry/PerspectiveCamera），WebGL 细节完全封装，社区语料和 loader 生态最深。交互范式是 Raycaster 拾取。与 three-d 的风格差异：three-d 的"显式传参"把场景组装责任推给调用方（自己维护 Vec<Object>），three.js 的场景图帮你记账（Group 嵌套、traverse）——前者少魔法、后者少样板。

```js
const renderer = new THREE.WebGLRenderer({ canvas });       // 复用宿主 canvas，rAF 自己驱动
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(45, w / h, 0.1, 100);
scene.add(new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshBasicMaterial()));
requestAnimationFrame(function loop() { renderer.render(scene, camera); requestAnimationFrame(loop); });
```

**deck.gl——声明式图层，数据即 prop**。与上面两者不同类：它不给你逐个 mesh 的命令式把手，而是"图层 = 表"——`ScatterplotLayer({ data, getPosition, getRadius, ... })`，每条记录直接映射为 GPU attribute，框架管理实例化和 redraw。这是对 GoG 哲学（encoding 映射，引擎自动算 scale）在空间展示方向的同构延伸：数据规模十万百万级时，three.js/three-d 的"每对象一个 JS/Rust 对象"成为瓶颈，deck.gl 从设计上跳过对象层。缺 Rust 移植，geo-layers（H3/地形/3D Tiles）生态无替代。

```js
new Deck({
  layers: [
    new ScatterplotLayer({ data: nodes, getPosition: d => d.xyz, getRadius: d => d.load, pickable: true }),
    new LineLayer({ data: edges, getSourcePosition: d => d.a, getTargetPosition: d => d.b }),
  ],
});   // 图层数组是纯数据；re-render 由 diff 图层 props 触发
```

**Bevy——ECS + schedule，游戏引擎**。渲染是世界的涌现结果：spawn `Camera3d` 实体、加 `Sprite3D`/`PbrBundle` 组件、gizmos 画线，全部经由 Entity-Component 与固定 schedule 提交。写数据可视化要绕开引擎的默认关切（物理、音频、动画、资产热重载），换来的收益（多线程渲染调度、WGSL 管线）在"画一帧网络拓扑"场景里兑现不了。API 抽象层级本身不是问题，问题是**层级不对口**。

## 三、体积：实测与公布值

| 库 | 形态 | 裸产物（opt 后） | gzip 传输 | 数据性质 |
|:--|:--|--:|--:|:--|
| three-d 0.19 | wasm | 867KB | 211KB | 本机实测 |
| ├ 净增量（扣除基线） | | +341KB | **+94KB** | |
| └ 基线：wasm-bindgen + web-sys WebGL2 绑定 | | 526KB | 118KB | 本机实测 |
| Bevy 0.17.3 | wasm | 18.9MB | 5.5MB | 本机实测 |
| three.js | JS | ~600KB min | ~102–125KB | 官方 CI 公布（tree-shaken 最小渲染器） |
| deck.gl 9.x | JS | ~505KB min | ~147KB | 官方文档公布（core+Layer，WebGL-only 144KB） |

实测方法说明：探针工程引用完整渲染面（`Context::from_gl_context → Camera → CpuMesh → Mesh → Gm → RenderTarget::screen().render()`），`default-features = false`，`opt-level="s" + lto="fat"`，trunk 自带 wasm-opt `-O` 压缩，与 ui_leptos 同口径。Bevy 探针为 3d+pbr+gizmos+sprite+winit+asset+default_font+png+webgl2 feature 集 + 一个空相机启动——40.9MB 编译产物经 wasm-opt 也只剩 18.9MB，与 three-d 差 22 倍。JS 两项为官方口径（min+gzip），含 tree-shaking 后最小集，全量引入更大（three.js full ESM ~180KB gz）。

体积对 fluxen 现状（wasm 2.64MB / gz ~1.5MB）的含义：three-d 是 raw +13%、传输 +6%，比此前引用的 plotters-canvas「~300KB 声称值」略高，且这是最小面——开灯光/PBR/`text`（swash+lyon 字体轮廓）还会叠加。Bevy 的 5.5MB gz 单独就超过现有整个应用，无需再讨论。

## 四、结论

**数据图表方向**：坚持图形语法路线（Vega-Lite / ggplot-rs / poloto，详见 `grammar-of-graphics.md`），四个候选渲染库都不是这个方向的工具——它们没有 encoding→scale 的自动推断。

**自定义空间展示方向**（Rust 主线）：**three-d**。中间层 API 满足"不太底层也不太高级"，显式渲染调用天然适配宿主 rAF，内建 2D（正交相机、Line/Particles/Control2D），+94KB gz 实测可接受。对象数在万级以内直接用它；十万级批绘要自己拿实例化几何搭（`InstancedMesh` + attribute buffer），这部分工程投入是离开 deck.gl 图层模型的真实代价。JS 侧的 deck.gl/three.js 不进入渲染层，但"图层=表"的 props 模型（数据直接映射 attribute、框架管 redraw）是自定义数据区的参照形态。在宿主 UI 框架（fluxen/Accrete）里的落位是两个组件的分工：**Chart** 专管数据可视化，不变量是"数据+定义→图"，渲染器可替换（ApexCharts → 图形语法 spec）；**Canvas** 是自由度容器——组件携带 CDN 模块 URL + 自定义数据区，模块动态拉资产、自持渲染循环（three-d 编成按需 wasm 模块即落位于此），主包不供养任何固定渲染逻辑。**Bevy 出局**：不是质量判断，是形态判断——垄断事件循环的引擎与嵌入式渲染的需求结构冲突，体积只是这个冲突最容易看见的症状。
