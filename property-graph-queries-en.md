# SQL/PGQ — Property Graph Queries

> **Languages:** [English](property-graph-queries-en.md) (primary) · [中文](property-graph-queries.md)

*The practical design of SQL Property Graph Queries on top of relational storage.* This article is about "how a graph is defined, queried, and evolved in a relational database," and it complements [Graph Memory](graph-memory-en.md): that one is the paradigm (why use a graph), this one is the carrier (how to build the graph in a Postgres stack).

## 1. What Is SQL/PGQ

SQL/PGQ (ISO/IEC 9075-16) is the **SQL standard's property graph query specification** — not a private syntax of any database vendor. It defines how to **declare a property graph on top of relational tables** and query it with graph match syntax.

Key facts (often misunderstood):
- **It is not tied to PostgreSQL** — Oracle has a complete implementation, and PostgreSQL is making progress but has only experimental support.
  The PG in "PGQ" stands for property graph, not PostgreSQL.
- **A graph is not new storage** — it is a **projected view** over existing tables. The user table, organization table, and edge table are exactly the graph's vertex table and
  edge table; `CREATE PROPERTY GRAPH` is only one layer of logical declaration, and the data stays in the original tables.
- **A graph is not equivalent to a graph database** — a graph database (Neo4j, etc.) is a standalone engine; SQL/PGQ is a
  query extension for relational databases that lets them traverse in graph fashion.

## 2. How a Graph Is Defined: Declare on Tables, Don't Copy Data

```sql
CREATE PROPERTY GRAPH org_graph
  VERTEX TABLES (
    users           KEY id,
    organizations   KEY id,
    skills          KEY id
  )
  EDGE TABLES (
    org_membership KEY id
      SOURCE KEY(org_id) REFERENCES organizations(id)
      DESTINATION KEY(user_id) REFERENCES users(id)
      LABEL org_membership,
    org_hierarchy KEY id
      SOURCE KEY(parent_org_id) REFERENCES organizations(id)
      DESTINATION KEY(org_id) REFERENCES organizations(id),
    user_relation KEY id
      SOURCE KEY(user_a) REFERENCES users(id)
      DESTINATION KEY(user_b) REFERENCES users(id)
  );
```

The `LABEL` clause of an edge table declares a label for the edge — this is exactly the label referenced by `-[e IS org_membership]->` in a graph match pattern; when `LABEL` is omitted, the table name is used as the label by default (so in the example above, writing it or not gives the same result).

A label is not a redundant alias of the table name; the two are names at different layers: the table name belongs to the physical layer, the label to the query layer, and only the label is visible in graph pattern syntax. The point of this decoupling lies in two things:

- **Multiple tables share one label**: two physical tables (such as the manually written `user_org_membership` and the system written `app_org_membership`, with different write controls and different fields) can declare the same label, and one graph pattern traverses both tables at once (the engine unions them internally); conversely, one edge table can also declare multiple labels, and the same batch of edges is referenced under different semantic framings. Filtering by table name cannot achieve either of these.
- **Table restructuring doesn't touch the query**: to split a table, merge tables, or rename a table, you only change the graph declaration in this one place, and the query layer's label vocabulary stays stable.

The `LABEL (rel)` of §7 promotes column values into labels (one physical edge table appears as multiple labels at the graph level), which is the advanced use of this decoupling — there the label is dynamic, whereas the labels in this section are constants.

The graph layer is only responsible for "how to query"; **"who can write" comes back to the underlying table permissions**. This yields a direct corollary: for edges that need strict protection (such as the user-organization membership relation), lock down writes **on the edge table** with FK + table permissions, don't open them to LLM/Agent, and the graph declaration has no side effect on this.

## 3. Core Design Criteria

### 3.1 One Graph, Many Relationships (a graph is not limited to one kind of relationship)

One graph can contain **multiple vertex tables + multiple edge tables (multiple kinds of relationships)**. In `org_graph`, membership, hierarchy, and relation are three different relationships, all within the same graph. One graph match query can traverse multiple kinds of edges:

```sql
-- Find all skills reachable by the sub-organizations a user belongs to (mixed traversal along membership + hierarchy)
SELECT DISTINCT s.name
FROM GRAPH_TABLE (org_graph
  MATCH (u IS users)
       -[e1 IS org_membership]->(o IS organizations)
       -[e2 IS org_hierarchy]->(o2 IS organizations)
       -[e3 IS skill_scope]->(s IS skills)
  WHERE u.id = ?)
AS gt;
```

### 3.2 The Criterion for Multiple Graphs: Merge If You Can, Don't Split

| Criterion | Decision |
|---|---|
| Same trust domain, semantically related relationships | **Merge into one graph** |
| Semantic domains completely separate and never queried across | Only then split into multiple graphs |

Typical example: the **identity-organization graph** (users/orgs/skills/membership/hierarchy/relation) handles permission reachability; the **memory knowledge graph** (triples) handles semantic retrieval — the two have different domains, so each gets its own graph.

### 3.3 Cross-Graph Queries: Inconvenient in the Standard, Rely on Underlying Tables in Practice

The SQL/PGQ standard **does not provide friendly cross-graph queries** — a graph match acts on a single named graph at a time, and there is no elegant way to jump from graph A to graph B within one query. But because a graph is only a projection of tables, the solution is:
- Multiple relationships in the same domain → merge into one graph, and one match inside the graph gets it done (convenient).
- Across domains → don't use graph syntax; use an **underlying table JOIN**, or first query one graph to get a set of ids and then query the other side.

**In one sentence: don't force a split for the sake of "cross-graph"; merge the same trust domain into one graph, and let genuinely cross-domain work go through the underlying tables.**

**Supporting mechanisms (PostgreSQL's official `CREATE PROPERTY GRAPH`):**

- **One table can belong to multiple graphs** — a graph is not physically materialized; it is a view-style projection declaration (the documentation's own words: "similar to CREATE VIEW... used only when queried"). The same underlying table can be referenced by any number of graphs, none of them claiming it exclusively. On this basis you can **build hierarchical graphs by scope**: with the same set of base tables, declare a sub-scope graph referencing only a few of them, then declare a full-scope graph referencing all of them — sharing the underlying tables, with the hierarchy expressed by "the subset of tables a graph references," and no data copied.
- **A graph cannot reference a graph** — the inputs to `CREATE PROPERTY GRAPH` are only tables (vertex/edge table), and there is no syntax for "taking a graph as a building block." To "merge" multiple graphs into one, you can only create a new, larger graph and list the tables each sub-graph uses once again. The atom of merging is the table, not the graph. This differs from SQL views, which can be layered (a view referencing a view): a PGQ graph is "a layer on top of tables," not "a layer on top of graphs."

Putting the two together: if you want to split multiple graphs by domain and then combine them, combining at the graph level is not possible, so all the more should you merge the same domain into one graph and route genuinely cross-domain work through the underlying tables — this is the structural basis for the "in one sentence" above, not a preference.

## 4. Implementing Permission Reachability

In Graph Memory, permission = graph reachability pruning at search time. SQL/PGQ is exactly the standard query implementation of that criterion — take PGQ as the basis, build one identity-organization graph, and express reachability with graph match syntax:

```sql
-- Starting from a user, traverse along org_membership + org_hierarchy to reachable skills
SELECT DISTINCT s.name
FROM GRAPH_TABLE (org_graph
  MATCH (u IS users)
       -[e1 IS org_membership]->(o IS organizations)
       -[e2 IS org_hierarchy]->(o2 IS organizations)
       -[e3 IS skill_scope]->(s IS skills)
  WHERE u.id = ?)
AS gt;
```

PGQ is the standard; if some deployment environment cannot yet enable a PGQ engine, use DuckDB as a fallback to carry the same set of tables and form graph queries with PGQ semantics. The graph declaration (`CREATE PROPERTY GRAPH`) is equivalent on any PGQ-compatible engine, and once the table structure is designed around graphs it can migrate across engines.

**Design points**: don't build multiple graphs (one identity-organization graph); one graph with many relationships; cross-domain work goes through the underlying tables, not graph syntax;
the user-organization edge is locked to manual writes by table permissions; strictness is assigned per edge.

## 5. Strictness Assigned per Edge (the User-Organization Relationship)

`org_membership` is a trust boundary and must **allow only manual writes (or writes through a strict approval process)**, with no Agent writes. Enforce this structurally with SQL:
- Underlying table permissions: the Agent's connection account has only SELECT on this table, no INSERT/UPDATE/DELETE.
- FK guarantees that endpoints must be genuinely existing users/organizations.
- Whereas the organizational hierarchy and user relations are soft; only if they need to evolve/be maintained automatically is this layer opened to the system.

**Strictness assigned per edge** — different edges in the same graph can have completely different write controls, and the graph layer declaration only determines queries.

## 6. Relationship to Graph Databases / Recursive CTEs

| Approach | Essence | Scenario |
|---|---|---|
| Graph database (Neo4j) | Standalone engine, native graph storage | Heavy graph analysis, needs graph algorithms |
| SQL/PGQ | Query extension on top of relational tables | Already have relational data, want to query it in graph fashion |
| Recursive CTE | Pure SQL, hand-written traversal | Simple reachability/paths, small scale |

SQL/PGQ is the sweet spot of "relational database + graph query": it introduces no standalone engine, yet is more expressive than hand-written recursive CTEs. PGQ is the standard; DuckDB serves as a fallback engine carrying the same set of tables, the graph declaration is equivalent across engines, and a table structure designed around graphs migrates smoothly.

**Placing the graph data model classifications**: the three above differ only in "carrier method"; underneath they share the same "graph semantics" spectrum. Graph models are layered along two axes — "whether nodes/edges carry properties" and "binary/n-ary edges":

| Model | Node | Edge | Property layer | Binary edge | Representative |
|:---|:---|:---|:---|:---|:---|
| Simple graph | ✓ | ✓ | ✗ (pure topology) | ✓ | Foundation of graph algorithms (PageRank, etc.) |
| Label graph | ✓ | ✓ | Edge type labels only | ✓ | — |
| RDF triple | ✓ (subject/object) | ✓ (the predicate is the edge) | No key-value layer, properties encoded as edges | ✓ | Semantic Web / knowledge graph |
| **Property graph (PG)** | ✓ | ✓ | **Both nodes + edges can carry key-value properties** | ✓ | Neo4j, SurrealDB, **PGQ**, hybrid storage |
| Hypergraph | ✓ | ✓ (can connect >2 nodes) | ✓ | ✗ (n-ary edge) | Higher-order relationship modeling |

**PGQ falls in the "property graph" cell** — `CREATE PROPERTY GRAPH` explicitly declares LABEL and PROPERTIES for edges (§2); it follows "property graph" semantics, not RDF triples. The **memory knowledge graph** (triples `(s,p,o)`) is instead RDF-style — relationships entirely open, no predefined labels, zero property declarations, which is exactly the other cell of the table above. This is the semantic basis for the two-cell division of labor in §7 of this document, "permission graph (property graph / closed relationships) vs memory graph (triples / open relationships)."

**"Whether it is a property graph" is not a matter of better or worse, but of semantic-layer belonging**: Neo4j/SurrealDB/PGQ share property graph semantics, and their divergence lies in the carrier method (materialized storage vs relational table projection); a triple graph expresses everything as edges and sets up no property container. All PGQ usage in this document lies within property graph semantics.

### SurrealDB's "Abolishing JOIN": Relationships as Data vs Relationships as Computation

SurrealDB claims to abolish JOIN; what it actually does is turn "fetching data across records" from **explicit computation at query time** into something **built into the data model**, with three layers of mechanism:

- **Record Link**: a field directly stores the id of another record (`author = author:john`); it is a typed pointer rather than a bare foreign key. Fetching associated data uses the single primitive `SELECT * FROM post:001 FETCH author`, replacing a hand-written `LEFT JOIN` every time; a nested field like `author.name` navigates along the pointer to read the value
- **Graph traversal**: `RELATE author:john->wrote->post:001` creates an edge, and `SELECT ->wrote->post FROM author:john` walks along pointers, with variable-length paths expressed natively instead of degenerating into a recursive CTE
- **Engine internal**: `FETCH` is still "take the id → look up the target table → assemble the result," with the same computational cost as a JOIN; the planner has simply written it for the user

Comparison with JOIN:

| Dimension | SQL JOIN | SurrealDB link/graph |
|:--|:--|:--|
| Relationship storage | Bare foreign key values, the type system doesn't care | Typed pointers, built into the model layer |
| Fetching associated data | Explicitly declare how to join on every query | A single primitive, `FETCH`/`->` |
| Variable-length paths / many-to-many | Recursive CTE, hand-written traversal | Native `->wrote->post->{...}` |
| Arbitrary combination queries across entities | Strength: join any table, any condition, any direction | Weak: walk along declared edges; ad hoc joins fall back to an underlying table JOIN |
| Semantics | Directionless row-set operations | Directed navigation |

**What it removes is only the burden of writing JOINs, not the computational essence of JOIN**. A link can navigate because the relationship type was pre-declared in the schema — this is completely isomorphic to the situation of the generic edge table in §7: declared relationships are extremely simple to query and recursive traversal is free; undeclared combinations (joining any table by any condition) still have to fall back to the underlying JOIN it retains, and they lose the SQL planner's freedom to optimize arbitrary joins. The accurate formulation is: **SQL treats relationships as computation (declared each time), SurrealDB treats relationships as data (pre-declared, navigated at query time)**. This determines its scope of applicability: within a domain where relationship types are predictable, use link/graph navigation; for genuinely open triple retrieval (outside the domain), both sides equally fall back to predicate filtering.

## 7. Making Relationship Types Dynamic

`CREATE PROPERTY GRAPH` declares an **edge table (one relationship type)**, not each individual edge. Once a relationship type is declared, adding any number of instances afterwards is a pure INSERT with no need to change the graph definition — **the instance layer is naturally dynamic**. The only case that truly requires changing the graph definition is "introducing a completely new relationship type."

Whether that is troublesome depends on whether the set of relationship types is closed or open:

| Graph | Relationship types | How it is made dynamic | Predefinition cost |
|---|---|---|---|
| Identity-organization permission graph | Small closed set (membership/hierarchy/relation/skill_scope) | Stable types + instance INSERT | A one-time few dozen lines of DDL, acceptable |
| Memory knowledge graph | Completely open (arbitrary triples `(s,p,o)`) | relation as inline data, zero definition | Zero |

If you want to also obtain "relationship types emerging dynamically" inside the permission graph, use a **generic edge table + multiple labels**:

```sql
CREATE TABLE edges (
  id BIGINT KEY DEFAULT,
  src_type TEXT, src_id BIGINT,
  rel      TEXT,              -- relationship type as inline data
  dst_type TEXT, dst_id BIGINT
);

CREATE PROPERTY GRAPH g
  VERTEX TABLES (
    users  KEY id,
    orgs   KEY id,
    skills KEY id
  )
  EDGE TABLE edges
    KEY (src_type, src_id)
      SOURCE KEY (src_id)
      DESTINATION KEY (dst_id)
      LABEL (rel);              -- the relationship type becomes a dynamic label
```

`SOURCE KEY`/`DESTINATION KEY` carry no `REFERENCES`: the endpoints are polymorphic (`src_type` points to any one of the users/orgs/skills tables), an FK can point to only a single table, and constraining endpoints by type cannot be written — endpoint validity can only be guaranteed by the application layer. This is the second thing a generic edge table sacrifices relative to an explicit edge table (§2) (the first being the closedness of the label set).

Adding a new relationship = inserting one row (fill the rel column with a new value); no table creation and no graph change. But mind the boundary: PGQ's label set still has to be predictable to some degree (a graph match `[:rel]` must be written out in full); if rel is a completely open set of emergent values, the graph match syntax cannot cover them all, and you fall back to "predicate filtering on the rel column" — which is the territory of the pure triple table.

A generic edge table does not exclude graph queries: SQL/PGQ allows the same edge table to be declared multiple times in `EDGE TABLES`, each time bound to different endpoint vertex tables and given a different label:

```sql
CREATE PROPERTY GRAPH g
  VERTEX TABLES (
    users  KEY id,
    orgs   KEY id,
    skills KEY id
  )
  EDGE TABLES (
    edges KEY (src_type, src_id)
      SOURCE KEY (src_id) REFERENCES users(id)
      DESTINATION KEY (dst_id) REFERENCES orgs(id)
      LABEL user_org_edges,
    edges KEY (src_type, src_id)
      SOURCE KEY (src_id) REFERENCES users(id)
      DESTINATION KEY (dst_id) REFERENCES skills(id)
      LABEL user_skill_edges
  );
```

After the declaration, `MATCH (u IS users)-[e IS user_org_edges]->(o IS orgs)` is a proper graph query and can traverse. The cost is that **every (source type, target type) combination must be declared as one entry**, and you must know in advance which combinations exist — once there are many type combinations the declarations explode. So the applicability of graph queries narrows as declarations grow: when relationship combinations are few and stable, declare them and query graphs as usual; when relationships are entirely emergent, graph patterns cannot express them (a vertex element can bind only one vertex table, there is no "any vertex," and edge endpoints can only REFERENCES one specific table), and only then do you fall back to a JOIN on the `src_type`/`dst_type` columns.

The two forms on the query side correspond to this dividing line:

```sql
-- Graph match: the label is written into the pattern, still written out in full (when the label set is predictable)
SELECT GRAPH_TABLE (g
  MATCH (u IS users)-[e IS membership]->(o IS orgs)
  COLUMNS (u.name, o.name)
)
WHERE u.name = 'alice';

-- When rel is fully open and the graph match cannot be written out in full, fall back to predicate filtering on the rel column (pure triple table usage)
SELECT * FROM edges
WHERE src_id = 42 AND rel = 'mentored';
```

### Generic Node Tables and the Hybrid Form

A generic edge table usually appears paired with a **generic node table**. Edges need to be dynamic, and nodes have the same need: to use the terminology of [Graph Memory](graph-memory-en.md), **in-domain entities** (users, organizations, products — stable property structure, stable database ids) use explicit tables; **out-of-domain entities** (free knowledge objects, external concepts — open properties, no stable identity) use a generic node table:

```sql
CREATE TABLE nodes (
  id   BIGINT KEY,
  kind TEXT,          -- node type: corresponds to a table name or a logical type name
  data JSON           -- the properties of that type, schema externalized
);
```

Adding `nodes` into `VERTEX TABLES` (`nodes KEY id`) brings it into the graph. **In the hybrid form the generic edge table goes from optional to mandatory**: an edge running from an in-domain table (`users`) to `nodes` is the norm; "declaring the same table multiple times" can cover cross-table edges, but the number of combinations = number of in-domain types × number of out-of-domain types, and out-of-domain types cannot be known in advance, so the declarations still explode. Therefore edges between in-domain entities also uniformly go through the generic edge table, and explicit edge tables are left only for trust boundaries that need FK locking (org_membership, see §5) — the natural corollary of "strictness assigned per edge" in the hybrid form.

The cost of the hybrid form:

- In-domain entities enter the generic edge table, losing FK endpoint constraints (the aforementioned sacrifice expands to all inter-entity relationships)
- The JSON column of `nodes.data` cedes all schema constraints on that type's properties to the application layer, and `data->>'x'` predicates cannot use ordinary column indexes (partially remedied with JSON indexes or materialized views)
- The boundary between "in-domain vs out-of-domain" is fluid: when a class of out-of-domain nodes gains a stable identity and must "graduate" into an in-domain explicit table, this involves data migration and rewriting all historical edges with `src_type='nodes'`/`dst_type='nodes'`, and the cost should be foreseen at design time

**Table form is determined per node**, isomorphic to determining dynamism per graph: in-domain entities go into explicit tables (stable structure, has an id, needs constraints and indexes); out-of-domain entities go into `nodes` (open properties, no stable identity, short-lived). The two coexisting in one graph is the norm, not a compromise.

Three typical queries for the hybrid form:

```sql
-- 1) In-domain → out-of-domain cross-table edge: graph query works as usual (precondition: edges declared the users→nodes combination)
SELECT GRAPH_TABLE (g
  MATCH (u IS users)-[e IS edges]->(n IS nodes)
  COLUMNS (u.name, e.rel, n.kind)
)
WHERE u.id = 42;

-- 2) Filtering out-of-domain node properties: a JSON column predicate (the concrete form of ceding schema to the application layer)
SELECT * FROM nodes
WHERE kind = 'paper' AND data->>'year' = '2026';

-- 3) Open traversal between out-of-domain nodes: graph patterns have no "any vertex," so fall back to an underlying table JOIN
SELECT n2.*
FROM edges e1
JOIN nodes n1 ON e1.src_type = 'nodes' AND e1.src_id = n1.id
JOIN nodes n2 ON e1.dst_type = 'nodes' AND e1.dst_id = n2.id
WHERE n1.kind = 'concept' AND e1.rel = 'contradicts';
```

Each of the three examples corresponds to one point: cross-table edge graph queries work (but are bound by declared combinations), the JSON predicate is the scene of the cost of ceding indexes, and genuinely open traversal can only be expressed with JOIN.

**Conclusion: determine dynamism per graph.** The permission graph has closed relationship types, so option A (generic edge table + multiple labels) is exactly right, and a one-time predefinition of a few dozen lines of DDL is acceptable; the memory graph has open relationships, so use pure triples `(s,p,o)` with zero definition. If you want fully explicit dynamism within a single graph (no declarations of any kind), SQL/PGQ does not support this by nature — types are column constraints — and that is precisely where DuckDB/pure triples is the fallback.

## Cross-References

- [Graph Memory](graph-memory-en.md) — Modeling paradigm: why use a graph, scale criteria, emergent Skills.
- [Memory Architecture](agent-memory-en.md) — The completeness argument for memory system storage selection.