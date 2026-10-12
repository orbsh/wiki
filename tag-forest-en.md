# Tag Forest: Organizing Content by Combining Mutually Exclusive Dimensions

> **Languages:** [English](tag-forest-en.md) (primary) · [中文](tag-forest.md)

Content organization usually gravitates toward two extremes. At one end is **pure multi-tagging** — tags mixed together, clicked freely, multi-selected freely, retrieval by tag hits. At the other end is a **single taxonomy** — strict hierarchical filing, each level mutually exclusive, an entry belonging to only one place at each level.

Both ends have their flaws. Pure multi-tagging cannot express a dimension like "current context" that **must choose one of two**: whether a page is currently in inbox or work can only be one, not both. A single taxonomy, by contrast, compresses multiple belonging into single inheritance: a "work" page that also wants to belong to "project:x" and to "news source" simply cannot express cross-dimension retrieval ("all pages that are work and carry privacy") under single inheritance.

Real organizational needs lie between the two: there must be **mutually exclusive dimensions** (single choice within a dimension) and also **cross-dimension composability** (different dimensions freely combined, hitting simultaneously). The **tag forest** is designed for exactly this shape, and makes the two extremes its degenerate cases.

## Model

Tags are a forest composed of multiple trees, **with no single root node, and generally not very deep**.

- **Trees are not mutually exclusive with each other**: a page can take values from multiple trees at the same time — multiple trees hit simultaneously, i.e. multiple belonging and cross-dimension combination.
- **Single choice within a tree**: a page takes only one value in a tree **as a whole**. Exclusivity applies to the whole tree, not merely to sibling nodes.
- **The degenerate cases naturally cover both extremes**: if **every tree in the forest has only one tag** (all single-node trees, freely combined across trees) → it degenerates into ordinary multi-selectable tags; if **all tags hang on the same tree** (a single multi-level tree) → it degenerates into the traditional classification mode.

Thus the forest is a model in which "each tree = one column of a mutually exclusive dimension, with dimensions freely combined"; the two extremes are merely its special cases.

## Tree Kinds

Each tree can be annotated with dimension attributes. Taking browser sessions (mudra) as an example, the trees fall roughly into three kinds:

- **situation (required, single-choice, default inbox)**: child nodes `inbox / work / personal / privacy`. "Current context", determining content belonging and isolation. Because it is required, a default value (inbox) is usually set, then adjusted manually.
- **importance / urgency (two levels, rank-ordered)**: child nodes are named with ☆ (`☆..☆☆☆☆☆`), and the `rank` field orders the star sequence. The natural shape of a rating is exactly a "discrete grade" — a page belongs to some importance grade, which is precisely a mutually exclusive dimension. Using ☆ as leaf tags and merging them into the forest means an **independent "rating field" is not needed**.
- **topic (ordinary, multi-selectable)**: `project:x / news / tech` etc., multiple belonging allowed.

Merging importance/urgency into the tree is the greatest structural simplification — the entire content organization contains only **one thing: the tag forest**. inbox, situation, importance, urgency and topic are all trees, structurally uniform, differing only in dimension attributes.

## Display and rank

Dimensions with rank display their grade: `importance:☆☆☆☆☆` or simply `☆☆☆☆☆`. Within a tree, `rank` provides stable ordering among child nodes, avoiding reliance on insertion order.

## Rating: Field vs Tree (Two Approaches)

importance / urgency has two forms; scratch empirically demonstrated that the field approach is usable:

- **Field** (`important INT` / `urgent INT`, which is what scratch uses): `ORDER BY important DESC` is direct, filtering/sorting is the cheapest; the weakness is weak fine granularity, hierarchy and multi-semantics — only one continuous value, unable to express composition with topic tags.
- **Tree** (`importance` tree + ☆ leaves + `rank`, adopted by this project): merged into the tag forest for uniformity, can be hierarchical, can have an alias, and can be isolated/retrieved like any other tag; the cost is that "unrated" must be expressed explicitly with a root/default value — unrated ≠ 0 points.

Trade-off: if you want simple sorting → field; if you want unity with the tag system and multi-dimension combination → tree. If you use the tree, you must preserve the **unrated sentinel (-1)** three-state semantics (see the schema below) — the default value points to unassigned, otherwise unrated content is mistakenly penalized as the lowest score (borrowing from scratch's `important/urgent DEFAULT -1`).

## schema (Adjacency List)

```sql
tag(id, parent_id, name, alias, isolated, required, rank, hidden, note)
page_tag(page_id, tag_id)
```

- Adjacency list `parent_id` + recursive CTE (WITH RECURSIVE) to query the tree; **root nodes use the `parent_id = -1` root sentinel** (not NULL), so recursion uniformly uses `where parent_id = -1`.
- `alias`: tag alias (display/retrieval name, abbreviation/multi-language), avoiding duplicate names and making prefix completion smoother.
- `hidden`: hidden tags (internal/obsolete tags, filtered out).
- `isolated`: hits an independent instance (see the next section).
- `required`: required dimension (such as situation), the app layer guarantees at least one value; the default value is also expressed at this layer.
- Single choice within a tree: an app-layer business constraint, not imposed by the schema.
- `rank`: ordering field among the same parent (☆ star sequence).
- `created`/`updated` use sqlite `DEFAULT (strftime(...))` automatic timestamps (borrowing from scratch).
- **Soft delete**: entry deletion uses a `deleted` marker (scratch defaults to ''), not a hard delete — recoverable and filterable; distinct from mudra page's `closed_at` (which only marks a time).

## Generic Query Patterns (Adjacency List Recursive CTE)

Using `tag(id, parent_id, name, ...)` and the association table `page_tag(page_id, tag_id)` as an example (scratch uses `scratch_tag`, the same schema pattern). Five queries cover the thematic operations on the tag tree, originating from scratch's `tag_base.nu`, and are portable to any adjacency-list tag relationship:

**① Fetch a node and all its descendants (top-down subtree)** — used for page belonging in isolated instances and for deleting an entire sub-tag:
```sql
-- Source: scratch `scratch-tags-children` (recursively collect the set of descendant ids).
with recursive g(id) as (
  select id from tag where id = :root
  union all
  select t.id from tag t join g on g.id = t.parent_id
)
select id from g;
```

**② Descend by hierarchical path, name chain → leaf id** — the user inputs a `:`-separated path such as `inbox/work`, matching tag names level by level to obtain the target node; when the data source is multiple groups of paths (the `name sequence after :root`), a `gr` group can carry each group descending independently (scratch `scratch-tag-paths-id`):
```sql
-- Naming aligned to source scratch `scratch-tag-paths-id`: input = sequence of (group number, name at that level);
-- v numbers each level (lv); g descends level by level, g.lv+1 corresponds to v.lv, each level joins v to take the name that level should match.
with recursive input(gr, name) as (
  values (0, 'inbox'), (0, 'work')        -- group number, level name (top-down); multiple paths get different gr
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
A missing level can be detected: result row count < input level count → the path does not exist, which can trigger "create by path".

**③ Whole tree with path names (concatenating ancestors)** — for displaying `importance:☆☆☆☆☆` and collapsed prefix completion:
```sql
-- Source: scratch `tag_tree` (recursively concatenate ancestor names into a full path); roots use the parent_id = -1 root sentinel.
with recursive tree(id, parent_id, name, path) as (
  select id, parent_id, name, name from tag where parent_id = -1
  union all
  select t.id, t.parent_id, t.name, tree.path || ':' || t.name from tag t
  join tree on tree.id = t.parent_id
)
select * from tree;
```

**④ Association retrieval: find pages by tag (including its descendants)** — single choice within a tree makes a "dimension value" a single node; using ① to fetch the subtree covers all branches of that dimension:
```sql
select p.* from page p
join page_tag pt on pt.page_id = p.id
where pt.tag_id in (select id from g);   -- g comes from ①
```

**⑤ Ensure a path exists (insert level by level, with a unique constraint as a fallback)** — scratch `scratch-ensure-tags`: descend by ②, insert any missing level and `returning id`, building level by level down to the leaf:
```sql
insert into tag(parent_id, name) values (:parent, :name)
on conflict(parent_id, name) do update set parent_id = excluded.parent_id
returning id;
```
(A unique constraint is added on `parent_id,name`.)

**Selection expressions (application-layer syntax)**: scratch `tags-group` uses prefixes to express tag combinations — `+tag` = required (and), `^tag` = excluded (not), `:tag` = optional (or). Multi-tag retrieval is the union/intersection of these expressions, and this model can reuse them directly.

## Isolation: isolated and Instances

If a tag is marked `isolated=true`, content under that tag runs in an **independent instance** — an independent profile / cookie / process boundary, used for isolating cookies/login state/privacy.

- When a page hits **multiple** isolated tags, it currently takes the "first matching instance". This is **undefined behavior** and is not specially handled — currently only the four child nodes of situation need isolation (`inbox/work/personal/privacy`), which are mutually exclusive, so no real conflict arises.

## Information Flow: inbox Routing and PWA Residency

- New information lands by default in `situation=inbox` (the required default value) — a staging inbox.
- After processing and adjustment, it is **moved to the corresponding mutually exclusive dimension value** (work/personal/…), i.e. content "flows" between instances.
- **pin / resident**: certain content is fixed to an instance and stays resident long-term (such as IM, RSS), running in an independent window like a native application — corresponding to a **Progressive Web App (PWA)**: installable, resident, independent window. A pin instance is a "resident application slot".

## Generalization and Library-ization

The tag forest is a **content organization paradigm**, not bound to a specific object type: browser pages (mudra), tasks (scratch) and knowledge graph nodes (graph-memory) are all its instances — the only difference being "what the tagged object is" and "how the backend stores it".

Abstraction boundary: **separation of the model layer from the adapter layer**. The model layer is this document (forest = multiple trees, cross-tree combination, single choice within a tree, rank ordering, isolated isolation, five recursive CTEs); the adapter layer is how a specific approach connects — a single `ForestBackend` interface (`nodes`/`tag_ids`/`assign`/`create`/`items`) lets the library be reused for any approach. Interaction (rating axis, capsule path, batch assignment) and display (custom item views, click/hover actions) are likewise made into interfaces, injected by each approach.

- **Landing point**: mudra's panel implementation and abstraction requirement documents live in the mudra repo (engineering-oriented); this file is the generic model.
- **Principle**: the library is not bound to mudra's names / tables / CDP / niri; mudra is merely the first adapter.

## Consistency with Existing Implementations

- **scratch** (task management) already organizes tasks with a tag tree (adjacency list) — this model is its generalization and naming, see [Task Management](task-management-en.md).
- **mudra** (browser session management): organizes pages with the tag forest, where `importance/urgency` is the importance/urgency tree. See the specific application documents.

## De-session: mudra's Application

The session plays three roles: **content organization** (topic belonging), **spatial presentation** (workspace), and **runtime carrier** (proxy/cookie isolation, lifecycle). What the tag forest replaces is the **content organization** layer — the single-belonging session is replaced by the multi-belonging forest, and more powerfully so. Runtime isolation is assigned to isolated instances and decoupled from content organization. Spatial presentation degenerates into "views" (routing filtered by tag/rating), no longer hard-coding workspace.

## Later: ML-Assisted Tagging

- Use **Naive Bayes / logistic regression** to learn "which dimension should choose which value" from historical manual selections — generating rules for review and entry into the table, rather than black-box inference. Heavy models like LLMs are not used (per-page inference is uncontrollable and costly).
- Domain/subdomain rules (e.g. the infoq main domain has high importance, user-contributed subdomains have low importance) → automatically assign default ratings to importance/urgency leaf tags, with manual override.
