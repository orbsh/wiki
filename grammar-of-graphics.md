# 图形语法（Grammar of Graphics）在 AI 驱动可视化中的应用与选型

## 一、核心理论：图形语法是什么

《图形语法》（The Grammar of Graphics）由统计学家 Leland Wilkinson 于 1999 年提出，把数据可视化解构为一组**通用、可重用**的组件，而非一系列固定的图表类型。

### 传统图表库 vs 图形语法

- **传统思维（ECharts、Chart.js、ApexCharts）**：图表是固定的死结构（折线图、饼图、柱状图）。遇到非标的复合交互需求，须单独写复杂绘制逻辑，或查阅庞大配置字典（Options Mapping）。
- **图形语法（Vega-Lite、AntV G2/F2）**：所有图表在底层都由相同的核心组件组合而成。通过抽象链式调用或声明式结构，开发者像搭积木一样自由发明新图表。

### 核心积木组件

```
[原始数据 Data] →度量 Scale→ [视觉通道 Aesthetics] →几何标记 Geom→ →坐标系 Coord→ [最终图表]
```

- **度量（Scale）**：连接数据空间与屏幕空间——把输入域（Domain，如销量 0~10000）映射到输出范围（Range，如像素 0~500px 或颜色数组）。
- **几何标记（Geometry / Geom）**：数据在屏幕上的物理形状。常见 Interval（柱状）、Line（线）、Point（点）、Area（面积）等。
- **视觉通道（Aesthetics / Attr）**：把数据字段分配给可见属性——position（位置）、color（颜色）、size（粗细/大小）、shape（形状）。
- **坐标系（Coordinate / Coord）**：几何标记铺开的网格系统——Rect（直角）、Polar（极坐标）。

**颠覆性逻辑（以饼图为例）**：在图形语法里没有"饼图"这种独立图表。饼图 = `Interval`（柱状标记）+ `Polar`（极坐标系）。在直角坐标下把柱子堆叠、再把坐标系拧成圆（极坐标），视觉上就变成饼图或南丁格尔玫瑰图。

## 二、为什么图形语法天然适合 AI 代码生成

在 LLM 自动出图（AI Data Agent）场景，遵循图形语法的库生成准确率大幅超越传统配置驱动引擎：

1. **高维度语义逻辑，消除幻觉**：传统库依赖深层嵌套的庞大 JSON 对象，AI 易在深层嵌套中拼错配置项或丢括号。图形语法是链式或扁平声明——AI 只需理解"将销量映射到高度（`.position('month*sales')`）"的语义映射，理解与输出都轻松。
2. **可预测性，支持"发明"图表**：面对未见过的非标图表，传统库无能为力；图形语法下 AI 像写 Prompt 一样拼装积木，代码即可运行：
   ```js
   chart.point().position('x*y').shape('heart').size('heat').color('sentiment'); // 自动生成"情感爱心散点图"
   ```
3. **数据与视觉完全解耦**：AI 只需两步推理——分析数据特征（连续型/离散型）→ 生成语法通道。这条思考链（Chain of Thought）高度契合大模型的底层逻辑。

## 三、轻量开源方案选型：Vega-Lite vs AntV F2

针对"开源免费、体积小（Bundle 敏感）、适配 AI 生成"的要求：

| 维度 | Vega-Lite（国际标准） | AntV F2（国产移动端） |
|------|-----------------------|----------------------|
| 开发背景 | 华盛顿大学交互数据实验室（微软 PowerBI 底层） | 阿里巴巴蚂蚁金服团队 |
| 开源协议 | BSD | MIT |
| 核心形态 | 纯 JSON 声明式（Spec） | JavaScript 链式调用 / 轻量声明 |
| 体积 | 数百 KB，只做编译、不含繁重渲染 | 极小，专为移动端 / 小程序瘦身 |
| AI 预训练友好度 | 强——主流大模型预训练已吞入大量 Vega-Lite 语料 | 高——中文社区友好，适合国内模型做 Few-shot |
| 最佳场景 | Web 桌面 SaaS、Python 数据中台、AI Data Agent 后端交付 | 微信/支付宝小程序、移动端 H5 运营页、手机端周报看板 |

## 四、Rust 生态方案：编译 WASM 或直接输出 HTML

上文聚焦 JS 生态。若技术栈是纯 Rust——服务端无 JS 引擎、或前端用 WASM——同样有遵循图形语法的方案。

### 横向总览

| crate | GoG？ | WASM | 输出 HTML/SVG | 备注 |
|-------|:-----:|:----:|:------------:|------|
| **`ggplot-rs`** 0.15 | ✅ 纯 Rust 实现 ggplot2 | ✅ ~310KB wasm | ✅ SVG / PNG 内存渲染 | **唯一全面满足的方案** |
| `plotlars` 0.12.6 | ❌ Builder 模式 | ✅（经 Plotly.js） | ✅ HTML 内嵌 Plotly | 深度绑定 Polars，命令式 |
| `vl-convert-rs` 2.0-rc1 | ✅ Vega-Lite | ❌ 内嵌 Deno v8 | ✅ SVG / PNG / HTML | v8 无法编译 WASM，仅服务端 |
| `plotly` 0.14 | ❌ 传统图表 | ✅ | ✅ HTML 内嵌 Plotly.js | 配置字典驱动，非图形语法 |
| `plotters` 0.3.7 | ❌ 底层绘图 | ✅ | ✅ SVG / Canvas | ggplot-rs 的底层渲染后端 |
| `gguppy` 0.4 | ✅ | ❓ | ❓ | 纯 Rust GoG，但较新、文档少 |

### ggplot-rs：纯 Rust 图形语法，WASM + 原生双栖

`ggplot-rs`（MIT/Apache-2.0）是 ggplot2 的 Rust 移植，渲染基于纯 Rust 的 `plotters` + 自研 `SvgBackend`，**不依赖任何 JS 引擎**。直方图/密度/堆叠/QQ 等计算层对照 R ggplot2 4.0.3 输出校验。

**能力一览**：

- **40+ Geom**：`geom_point/line/bar/histogram/boxplot/violin/smooth/density/area/ribbon/contour/hex/tile/...`
- **完整 GoG 组件链**：Scale（linear/log10/sqrt/Box-Cox…）、Coord（polar/flip/fixed/trans）、Facet（wrap/grid + free scales）、Stat（bin/density/smooth/ecdf/qq/…）
- **主题系统**：`theme_gray/bw/minimal/dark/classic/...` + 运行时品牌色注入
- **空间数据**：`geom_sf` 读 WKT 几何 + 投影（Mercator 等），`coord_sf` 自动等比
- **数据输入**：polars（默认）/ Apache Arrow RecordBatch / 纯 Rust Vec — polars 可完全关闭

**原生渲染（服务端 SVG → 嵌入 HTML）**：

```rust
use ggplot_rs::prelude::*;

let svg: String = GGPlot::new(df)
    .aes(Aes::new().x("sepal_length").y("sepal_width").color("species"))
    .geom_point()
    .render_svg()?;  // 直接内存出 SVG，嵌入 HTTP 响应即可
```

**WASM 浏览器渲染**（`wasm` feature，无 polars、无字体文件，~310KB `.wasm`）：

```sh
wasm-pack build --target web --no-default-features --features wasm
```

```js
import init, { render_geo } from "./pkg/ggplot_rs.js";
await init();
// 返回 SVG 字符串，每个 mark 是真实 DOM 元素 → hover tooltip 开箱即用
document.getElementById("plot").innerHTML = render_geo(JSON.stringify({
  geometry: [/* WKT */], fill: [/* 数值 */], label: [/* 名称 */],
}));
```

**大数据量**：`canvas` feature 提供纯 Rust RGBA 光栅后端（`render_rgba` / `render_png_raster`），50 万点秒级渲染，wasm 兼容。

**Feature Flags 速查**：

| Feature | 默认 | 用途 |
|---------|:----:|------|
| `polars` | ✅ | DataFrame 输入（可关闭减重） |
| `arrow` | ❌ | Arrow RecordBatch 输入（DuckDB 直连） |
| `wasm` | ❌ | 浏览器 SVG 渲染绑定 |
| `canvas` | ❌ | 大数据量 RGBA 光栅后端 |
| `sf` | ❌ | `geom_sf` 空间几何渲染 |
| `geojson` | ❌ | GeoJSON → WKT 读取 |
| `cli` | ❌ | CLI 工具（Parquet/CSV/DuckDB SQL → SVG/PNG） |

> Live demo：<https://sipemu.github.io/ggplot-rs/>

### plotlars：Polars 生态桥接，Builder 模式（非图形语法）

`plotlars`（MIT）专为 Polars DataFrame → 图表而生，采用 Rust 链式 Builder 模式（`.set_x()` / `.set_y()`），底层封装 Plotly（交互式 HTML）和 Plotters（静态图）。22+ 内置图表类型含 3D、地图、金融 K 线。深度绑定 Polars，内置 CSV/Parquet 读取器。

**与 ggplot-rs 的本质区别**：

| 维度 | `ggplot-rs` | `plotlars` |
|------|-------------|------------|
| 设计哲学 | 图形语法（声明式 encoding 映射） | Builder 模式（命令式 `.set_x()`） |
| 语法风格 | R 风格图层叠加 `ggplot(df) + geom_point()` | Rust 链式调用 `Scatter::new().set_x(...)` |
| 渲染后端 | plotters（静态 SVG/PNG） | Plotly（交互式 HTML）+ Plotters（静态） |
| Polars 依赖 | 可选（可完全关闭） | 深度绑定，核心卖点 |
| 数据未知场景 | AI 只出 encoding JSON，引擎自动算 Scale/刻度 | AI 需预判 chart type + 调用具体 API |
| 输出可序列化为声明式 config | 天然 JSON Spec | 命令式调用，难以抽离为纯配置 |
| 互动性 | 静态出版图；WASM SVG 有 DOM hover | Plotly 后端天生交互式（缩放/悬浮/3D） |

**选型**：需要交互式 HTML / 3D / 地图 / 深度 Polars 工作流 → `plotlars`；需要纯 Rust GoG + WASM + 声明式配置 → `ggplot-rs`。但在 NL-to-SQL 架构中（详见第六章），Rust 不负责渲染，两者均非最优——前端 Vega-Lite / G2 才是渲染方。

### vl-convert-rs：Vega-Lite 的 Rust 封装（仅服务端）

`vl-convert-rs`（BSD-3-Clause）内嵌 Deno v8 运行时执行 Vega-Lite 的 JavaScript，输出 SVG/PNG/HTML，支持 Vega-Lite 5.8–6.4 多版本。**但 v8 引擎无法编译 WASM**，冷启动重（Deno + 字体加载），适合服务端批量转换场景，不适合前端 WASM。配套的 `vl-convert-canvas2d` 用纯 Rust（tiny-skia）实现 Canvas 2D API，但核心编译仍依赖 JS。

### 选型结论

- 纯 Rust 技术栈、WASM 前端、服务端无 JS 引擎 → **ggplot-rs**（唯一 WASM + SVG/HTML 双栖的纯 Rust GoG）
- Polars 工作流、需要交互式 HTML / 3D → **plotlars**（接受非 GoG 的 Builder 模式）
- 需要完整 Vega-Lite 兼容、多版本 spec、服务端批量转图 → **vl-convert-rs**（接受 v8 重量级代价）
- 不要求图形语法、只要传统图表 + WASM → `plotly` 或 `plotters`

## 五、被淘汰的竞品与边界

图形语法解决的是"数据→图表"的编码问题；当需求越过平面图表、进入**自定义空间展示**（3D 网络拓扑、大规模点云、deck.gl 式地理场景）时，选型逻辑换轨——那是渲染库的 API 风格与体积之争，不再有 encoding→scale 的自动推断可用，见 `3d-rendering-library-selection.md`。

- **ECharts / AntV G2 原版**：含完整渲染引擎（如 ZRender），体积通常 MB 级，对包体积敏感的太重。
- **Chart.js**：体积小，但属配置字典驱动，AI 极易盲猜参数出错。
- **D3.js**：底层操作工具，无图表抽象，AI 需写上百行 DOM 操作，出错率极高。

### ApexCharts：典型的"模版 + 配置字典"流派（非图形语法）

ApexCharts 是一套优秀的**图表模版与配置字典驱动**的传统库。其配置也是巨大的 options JSON，表面与 Vega-Lite 的声明式 JSON 相似，但底层架构与 AI 适配度有本质区别。

**语义区别：调用模版 vs 自由组装**
- ApexCharts：显式指定 `chart.type: 'bar'`，再查该类型下怎么配颜色、怎么配粗细——厚重的配置字典。
- 图形语法：没有柱状/折线的概念，只有 `interval` / `line` 等几何标记；换坐标系配置图表就自行变异，所有图表都是临时组合。

**数据格式区别：行列分离 vs 结构化记录**
- ApexCharts：Series + Labels 架构，数据拆成一列 X（categories）和一列 Y（data）；AI 改图需写复杂算法把原始数据转置、对齐、裁剪，出错率高。
- 图形语法：只吃扁平化 JSON 数组（Tidy Data），无需预处理，AI 只调整 encoding 映射关系。

**横向对比（ApexCharts 7.x vs Vega-Lite）**

| 维度 | ApexCharts | Vega-Lite |
|------|-----------|-----------|
| 颜值与动画 | 默认 UI 现代、动效细腻 | 朴素，学术界/科学计算风格，需调校 |
| 功能经引 | 18+ 常见图表开箱即用 | 靠 mark+encoding 自由拼装，理论无限、常见要自配 |
| 渲染后端 | 纯 SVG（≥万点卡顿） | Canvas/SVG 两栖，海量数据可切 Canvas |
| 包体积 | 曾经近 1MB，新版支持 Tree-shaking | 数百 KB，无沉重渲染包 |
| AI 友好度 | 中——常规图表可，非标复合图表因配置冲突幻觉 | 高——天然契合推理链，国际主流大模型御用底层 |

## 六、工业级落地架构：NL-to-SQL → AI 生成展示意图 → 前端渲染

### 架构全景

在 NL-to-SQL 场景中，AI 的核心任务不是预测数据数值，而是推断**数据的结构与意图**。推荐"AI 同时输出 SQL + 图表展示意图（图形语法配置）、后端只做数据搬运"的架构：

```
┌──────────────────────────────────────┐
│           AI 层推理结果               │
│  NL → SQL + 图表 encoding 配置       │
└──────────────────┬───────────────────┘
                   ▼
┌──────────────────────────────────────────┐
│  后端 Rust：执行 SQL，组装统一 JSON 响应  │
│  { sql_data: [...], chart_config: {...} }│
└──────────────────┬───────────────────────┘
                   ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  SQL 查询结果     ├────►│  前端渲染引擎     ├────►│  交互式图表       │
│  (原始 Data JSON) │     │  (Vega-Lite/G2)  │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

关键设计：**数据与配置分离交付**。AI 不直接渲染数据数值，只生成"映射关系"；后端 Rust 不画图，只组装 JSON。

### 为什么图形语法在数据未知场景下是绝对优势

NL-to-SQL 时 AI 无法预判 SQL 查出的数值范围，图形语法的三个特征恰好化解这一困难：

1. **声明式配置天然是 JSON**：AI 只输出映射关系（"revenue 映射到 Y 轴"），无需调用 API 或预判 chart type，配置即 JSON 结构，前端直接消费。

2. **Scale 自动推断容错性极高**：AI 只需声明"把 revenue 映射到高度"，前端引擎自动根据实际数据计算 min/max、生成刻度（Ticks）和颜色渐变。AI 完全不需要知道数据范围。

3. **图层叠加解耦复杂业务**：用户说"展示销售额并标出平均线"——AI 只需在 encoding 中加一层 `"y": {"aggregate": "mean", "field": "revenue"}`，计算和绘制全由前端运行时完成。

### 统一 JSON 响应格式

后端将 SQL 结果与 AI 生成的图表配置打包为统一响应：

```json
{
  "sql_data": [
    { "department": "Tech", "headcount": 45, "budget": 90000 },
    { "department": "HR", "headcount": 12, "budget": 25000 }
  ],
  "chart_config": {
    "description": "AI 自动生成的部门预算与人数关系图",
    "mark": "point",
    "encoding": {
      "x": { "field": "headcount", "type": "quantitative" },
      "y": { "field": "budget", "type": "quantitative" },
      "color": { "field": "department", "type": "nominal" }
    }
  }
}
```

### 前端消费

引入 Vega-Lite 或 AntV G2 等图形语法前端库，分别注入数据和配置：

```js
// Vega-Lite
vegaEmbed('#vis', { ...response.chart_config, data: { values: response.sql_data } });

// AntV G2
const chart = new Chart({ container: 'container' });
chart.data(response.sql_data);
// 将 AI 给的 encoding 配置动态应用
chart.encode('x', 'headcount').encode('y', 'budget').encode('color', 'department')
     .mark('point')
     .render();
```

天然规避 `eval()` 执行动态 JS 的安全风险——配置是纯数据，非代码。

### Rust 后端的角色：极简搬运工

此架构下 Rust 后端**不需要任何图表渲染库**（不需要 ggplot-rs、不需要 plotlars），只做三件事：

1. 执行 AI 生成的 SQL，拿到结果集 → 转为扁平 JSON 数组（Tidy Data）
2. 透传 AI 生成的图形语法配置 JSON（校验 schema，不渲染）
3. 组装 `{ sql_data, chart_config }` 返回前端

ggplot-rs / plotlars 的价值在于**需要服务端渲染**（无前端 JS、导出 PDF/报告、WASM 离线应用）的备选路径，而非此架构的主线。

### AI 侧的结构化 Prompt（模板）

> 你是一个数据分析专家。请根据用户问题完成两件事：
> 1. 生成 SQL 查询语句
> 2. 生成 Vega-Lite 图表配置 JSON
>
> `[数据库 schema]` … `[用户问题]` … `[输出约束]` 只输出纯净 JSON，包含 `sql` 和 `chart_config` 两个字段，不做 markdown 包裹。

## 七、总结与选型落地建议

- **人类手写、追求颜值/动画/开箱即用的管理系统** → 坚持 ApexCharts，用 7.x 核心按需引入控制体积。
- **AI 代理主导、自动解析数据生成不可预测自定义图表的系统** → 切换到 Vega-Lite。
- **NL-to-SQL 架构、前端渲染** → AI 生成 Vega-Lite encoding JSON + 原始数据分离交付；Rust 后端只做搬运工，不渲染。
- **纯 Rust 技术栈、WASM 前端或服务端无 JS 引擎** → ggplot-rs（详见第四章）。
- **Polars 工作流、需交互式 HTML/3D** → plotlars（非 GoG，但开箱即用）。

> 注：上文体积、版本号、功能列举基于 2026 年的库版本，具体以官方文档与装机实测为准；引用来源未逐一核验。
