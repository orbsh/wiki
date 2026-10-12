# 标签森林：多互斥维度组合的内容组织模型

> **语言：** [English](tag-forest-en.md)（主文档）· [中文](tag-forest.md)

内容组织通常落向两个极端。一端是**纯多标签**——标签混排，任意点击、任意多选，检索按标签命中。另一端是**单一分类学**——严格的层级归档，每层都互斥，一个条目在每一层只有一个归属。

两端各有缺陷。纯多标签表达不了"当前上下文"这种**必须二选一**的维度：一个页面此刻处于 inbox 还是 work，只能是一，不能是两个。单一分类学则把多归属压成了单继承：一个"工作"页面同时也想属于"项目:x"、属于"新闻源"，跨维度检索（"所有 work 且带 privacy 的页"）在单继承下根本表达不出来。

真实的组织需求在两者之间：既要有**互斥维度**（某维度内单选），又要**跨维度可组合**（不同维度自由搭配、同时命中）。**标签森林**即为这个形态而设计，并让两个极端成为它的退化情形。

## 模型

标签是多棵树组成的森林，**没有单一根节点，一般也不太深**。

- **树之间不互斥**：一个页面可以同时取多棵树的值——多棵树同时命中，即多归属、跨维度组合。
- **树内单选**：一个页面在一棵树**整树**只取一个值。互斥作用于整棵树，而非仅同级节点。
- **退化情形自然覆盖两端**：若**森林里每棵树都只有一个标签**（全是单节点树、树间自由组合）→ 退化成普通可多选标签；若**所有标签都挂在同一棵树上**（单一多层级树）→ 退化成传统分类模式。

因此森林是"每棵树 = 一个可互斥的维度列，维度之间自由组合"的模型，两个极端只是它的特例。

## 树的种类

每棵树可以标注维度属性。以浏览器会话（mudra）为例，树大概分三类：

- **situation（必选、单选、默认 inbox）**：子节点 `inbox / work / personal / privacy`。"当前上下文"，决定内容归属与隔离。因必选，通常设默认值（inbox），再手动调整。
- **importance / urgency（两级、rank 排序）**：子节点以 ☆ 命名（`☆..☆☆☆☆☆`），`rank` 字段排星序。评分的自然形态就是"离散等级"——一个页面属于某个重要度等级，恰是一个互斥维度。用 ☆ 作叶 tag 并入森林，就**不需要独立的"评分字段"**。
- **topic（普通、可多选）**：`项目:x / 新闻 / 技术` 等，可多选归属。

importance/urgency 并入树是最大的结构简化——整个内容组织只存在**一种东西：标签森林**。inbox、situation、重要性、紧急性、话题全都是树，结构统一，只是维度属性不同。

## 显示与 rank

带 rank 的维度显示等级：`importance:☆☆☆☆☆` 或直接 `☆☆☆☆☆`。`rank` 在树内子节点间提供稳定排序，避免依赖插入顺序。

## 评分：字段 vs 树（两种方案）

importance / urgency 有两种形态，scratch 实证了字段方案可用：

- **字段**（`important INT` / `urgent INT`，scratch 即此）：`ORDER BY important DESC` 直给，过滤/排序最省；弱点是细粒度、层级、多语义弱——只能一个连续值，不能表达与主题标签的复合。
- **树**（`importance` 树 + ☆ 叶 + `rank`，本项目采用）：并入 tag 森林统一，可层级、可 alias、与其他 tag 一样可隔离/检索；代价是"未评"要用根/默认值显式表达——未评 ≠ 0 分。

取舍：追求简单排序 → 字段；追求与标签体系统一、多维度组合 → 树。若用树，须保持**未评哨兵（-1）**三态语义（见下 schema）——默认值指向未赋值，否则未评内容被当最低分误伤（借鉴 scratch `important/urgent DEFAULT -1`）。

## schema（邻接表）

```sql
tag(id, parent_id, name, alias, isolated, required, rank, hidden, note)
page_tag(page_id, tag_id)
```

- 邻接表 `parent_id` + 递归 CTE（WITH RECURSIVE）查询树；**根节点用 `parent_id = -1` 根哨兵**（不用 NULL），递归统一 `where parent_id = -1`。
- `alias`：标签别名（显示/检索名、缩写/多语言），避免同名、前缀补全更顺。
- `hidden`：隐藏标签（内部/过时 tag，过滤）。
- `isolated`：命中独立实例（见下节）。
- `required`：必选维度（如 situation），app 层保证至少一个值；默认值也在此层表达。
- 树内单选：app 层业务约束，schema 不强加。
- `rank`：同父节点排序字段（☆ 星序）。
- `created`/`updated` 用 sqlite `DEFAULT (strftime(...))` 自动时间戳（借鉴 scratch）。
- **软删除**：条目删除用 `deleted` 标记（scratch 默认 ''），非硬删——可恢复、可过滤；区别于 mudra page 的 `closed_at`（只标时间）。

## 通用查询模式（邻接表递归 CTE）

以 `tag(id, parent_id, name, ...)` 与关联表 `page_tag(page_id, tag_id)` 为例（scratch 为 `scratch_tag`，同名模式）。五条查询覆盖 tag 树的主题操作，源自 scratch 的 `tag_base.nu`，可移植到任何邻接表标签关系：

**① 取某节点及其全部后代（自顶向下子树）** —— 用于隔离实例的页面归属、删除整棵子标签：
```sql
-- 来源：scratch `scratch-tags-children`（递归收集后代 id 集合）。
with recursive g(id) as (
  select id from tag where id = :root
  union all
  select t.id from tag t join g on g.id = t.parent_id
)
select id from g;
```

**② 按层级路径下探，name 链 → 叶 id** —— 用户输入 `inbox/work` 这种 `:` 分隔路径，逐层匹配 tag 名得到目标节点；数据源为多组路径（`:root 之后的 name 序列`）时可带 `gr` 分组逐组独立下探（scratch `scratch-tag-paths-id`）：
```sql
-- 命名对齐来源 scratch `scratch-tag-paths-id`：input = (组号, 该级 name) 序列；
-- v 给每级编号(lv)；g 逐层下探，g.lv+1 对应 v.lv，每层 join v 取该层应匹配的 name。
with recursive input(gr, name) as (
  values (0, 'inbox'), (0, 'work')        -- 组号, 层级 name(自顶向下)；多路径给不同 gr
),
v as (
  select row_number() over (partition by gr) as lv, gr, name from input
),
g(lv, gr, parent_id, id, name) as (
  select 1 as lv, v.gr, t.parent_id, t.id, t.name from tag t
  join v on v.lv = 1 and t.name = v.name
  union all
  select (g.lv + 1) as lv, g.gr, t.parent_id, t.id, t.name from tag t
  join v on v.lv = (g.lv + 1) and v.gr = g.gr
  join g on g.id = t.parent_id
  where t.name = v.name
)
select id, name from g order by gr, lv;
```
缺失层级可判别：结果行数 < 输入层数 → 路径不存在，可触发"按路径创建"。

**③ 整树带路径名（拼接祖先）** —— 显示 `importance:☆☆☆☆☆`、折叠前缀补全：
```sql
-- 来源：scratch `tag_tree`（递归把祖先名拼成完整路径）；根用 parent_id = -1 根哨兵。
with recursive tree(id, parent_id, name, path) as (
  select id, parent_id, name, name from tag where parent_id = -1
  union all
  select t.id, t.parent_id, t.name, tree.path || ':' || t.name from tag t
  join tree on tree.id = t.parent_id
)
select * from tree;
```

**④ 关联检索：按 tag（含其后代）找页面** —— 树内单选使"维度值"是一个节点，用 ① 取子树即覆盖该维度全部分支：
```sql
select p.* from page p
join page_tag pt on pt.page_id = p.id
where pt.tag_id in (select id from g);   -- g 来自 ①
```

**⑤ 确保路径存在（逐级插入，唯一约束兜底）** —— scratch `scratch-ensure-tags`：按 ② 下探，缺层即插入并 `returning id`，逐级建到叶：
```sql
insert into tag(parent_id, name) values (:parent, :name)
on conflict(parent_id, name) do update set parent_id = excluded.parent_id
returning id;
```
（`parent_id,name` 上加唯一约束。）

**选择表达式（应用层语法）**：scratch `tags-group` 用前缀表达标签组合——`+tag`=必须（and）、`^tag`=排除（not）、`:tag`=任选（or）。多标签检索即这些表达式的并集/交集，本模型可直接复用。

## 隔离：isolated 与实例

一个 tag 标记 `isolated=true`，则该 tag 下的内容运行在**独立实例**——独立 profile / cookie / 进程边界，用于 cookie/登录态/隐私的隔离。

- 一个 page 命中**多个** isolated tag 时，现取"第一个匹配的实例"。这是**未定义行为**，不做特殊处理——当前只有 situation 的四个子节点需要隔离（`inbox/work/personal/privacy`），互为互斥，不会出现真冲突。

## 信息流：inbox 分流与 PWA 常驻

- 新信息默认落在 `situation=inbox`（必选默认值）——一个暂存收件箱。
- 处理、调整后**移动到对应的互斥维度值**（work/personal/…），即内容在实例间"流动"。
- **pin / 常驻**：某些内容固定在某实例长期常驻（如 IM、RSS），像原生应用一样独立窗口运行——对应**渐进式 Web 应用（PWA）**：可安装、常驻、独立窗口。pin 实例就是"常驻应用槽"。

## 通用化与库化

标签森林是一个**内容组织范式**，不绑定具体对象类型：浏览器页面（mudra）、任务（scratch）、知识图节点（graph-memory）都是它的实例——区别只在"被贴 tag 的对象是什么"以及"后端如何存"。

抽象边界：**模型层与适配器层分离**。模型层是本文档（森林 = 多树、树间组合、树内单选、rank 排序、isolated 隔离、五条递归 CTE）；适配器层是具体方案如何对接——一个 `ForestBackend` 接口（`nodes`/`tag_ids`/`assign`/`create`/`items`）即可让库复用到任何方案。交互（评分轴、胶囊路径、批量指派）与显示（自定义 item 视图、点击/悬停动作）同样做成接口，由各方案注入。

- **落点**：mudra 的面板实现与抽象化需求文档在 mudra 仓内（工程向）；本文件是通用模型。
- **原则**：库不绑 mudra 的名字 / 表 / CDP / niri；mudra 只是第一个适配器。

## 与现有实现的一致性

- **scratch**（任务管理）已用标签树（邻接表）组织任务——本模型是其泛化与命名，见 [任务管理](task-management.md)。
- **mudra**（浏览器会话管理）：用标签森林组织页面，`importance/urgency` 即重要性/紧急性树。见具体应用文档。

## 去 session：mudra 的应用

会话 session 承担了三个角色：**内容组织**（主题归属）、**空间展示**（workspace)、**运行载体**（proxy/cookie 隔离、生命周期）。标签森林取代的是**内容组织**这一层——单归属的 session 被多归属的森林替代，且更强。运行隔离归到 isolated 实例，与内容组织解耦。空间展示退化为"视图"（按标签/评分过滤路由），不再硬编码 workspace。

## 后续：ML 辅助打标签

- 用**朴素贝叶斯 / 逻辑回归**从历史手动选择学习"哪个维度该选哪个值"——生成规则供审阅落表，而非黑箱推断。不用 LLM 这类重模型（每页面推断不可控、成本高）。
- 域名/子域名规则（如 infoq 主域高重要、用户贡献子域名低重要）→ 自动给 importance/urgency 叶 tag 打默认分，手动覆盖。