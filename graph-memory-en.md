# Graph Memory

> **Languages:** [English](graph-memory-en.md) (primary) · [中文](graph-memory.md)

*The computational-timing spectrum of memory architectures, and where the property graph sits within an agent memory system.*

## The Computational-Timing Spectrum

The essential difference between memory designs is not the data structure, but **which stage the computation happens at**.

```
Write-heavy ────────────────────────── Read-heavy

LLM Wiki        KG (property graph)      KV pattern         Vector pattern
Memory Palace   PGQ                      Postgres           Postgres + pgvector
                (property graph)         (relation)         (embedding index)

Write: organize/embed/infer topology   Write: simple chunk/store
Read:  direct load / traverse          Read:  match / rank / vector search
```

Position on the spectrum does not imply different storage products—Postgres covers the entire range through three forms: graph, relational, and vector. Position depends on **how it is used**: a property graph (PGQ) is write-time heavy, a relational table is read-time heavy, and an embedding index sits in the middle.

### Write-Time Heavy

The write stage invests substantial compute in organizing, embedding, and building indexes. Reads load directly, with zero matching overhead.

**Representative**: LLM Wiki, memory palace, precomputed knowledge graph.

**Advantage**: reads are highly deterministic and miss nothing. One read yields the complete answer; no search or ranking is needed.

**Cost**: write cost is high. Every update requires re-organizing the topological relations and re-embedding. Incremental updates are hard—adding one new piece of knowledge may affect the global index.

**Applicable**: scenarios where knowledge changes infrequently but is read frequently. Document-style knowledge bases, project architecture decision records.

### Read-Time Heavy

The write stage performs only simple storage (chunking, vectorization). Reads dynamically retrieve through matching algorithms.

**Representative**: vector retrieval (RAG), semantic search.

**Advantage**: write cost is extremely low. New knowledge is written and immediately usable; no index rebuild is needed.

**Cost**: reads are probabilistic. Vector matching may miss knowledge that is semantically relevant but lexically mismatched. Retrieval quality depends on the embedding model and query quality.

**Applicable**: scenarios where knowledge changes frequently and reads require fuzzy matching. Conversation history, code snippet retrieval.

### The Middle Zone: Property Graph

At write time it establishes entity relations and attributes, but performs no deep topological inference. At read time it supports exact traversal and path queries, without the compute overhead of vector matching.

**Representative**: PGQ (`CREATE PROPERTY GRAPH`), property graph databases, weighted atomic fact storage (see [Property Graph Queries](property-graph-queries-en.md) for details).

**Advantage**: moderate write cost (just build relations), high read determinism (graph traversal is deterministic). Supports multi-hop reasoning (the transitive relation A → B → C), which vector retrieval cannot do.

**Cost**: entity alignment and deduplication require extra logic. The same-name-different-meaning problem across documents requires disambiguation.

**Applicable**: scenarios where knowledge has clear structural relations and reasoning chains are needed. Architecture decision chains, causal tracking, technology-selection exclusion chains.

## The Essence of the Spectrum: Index Timing

The difference between the four designs can be compressed into one variable: **when the index is built**.

| Design | Index timing | Index content | Read method |
|:--|:--|:--|:--|
| Wiki / memory palace | Write time | Full text + structure + topology | Direct load |
| Property graph | Write time | Entity + relation + attribute | Graph traversal |
| Vector retrieval | Read time (implicit) | embedding vector | ANN search |
| KV pattern | Write time | key-value pairs | Exact lookup |

"Write-time heavy" means the index is built at write time and reads query the index directly. "Read-time heavy" means only raw data is stored at write time, and the index is built dynamically at read time (vectorization + approximate search).

**Index timing determines the trade-off between update cost and query quality**:

- Write-time index: high update cost (changing one item requires rebuilding the index), high query quality (the index is exact)
- Read-time index: low update cost (writing completes it), bounded query quality (the ceiling of the embedding model)

## Why Graphs Work: the Existence Proof in Code

One fact deserves to stand as an argument rather than an anecdote: AI is
especially effective at programming, and one root reason is that **code is
inherently a graph** — a call site jumps to its definition, a definition
references other functions, a function may be referenced from many places.
Without an LSP, a name still finds its definition. Natural language cannot
do that.

The code domain hands over all four requirements of graph retrieval for
free:

1. **Identity is declared and unique** — one definition, one canonical
   name; no "one concept, ten synonyms".
2. **Relations are typed and directed** — calls, imports, references, plus
   the reverse "referenced by".
3. **Resolution is exact and cheap** — name → definition is a lookup, not a
   similarity computation. This is the decisive difference from natural
   language.
4. **Multiplicity and reverse edges are first-class** — N callers of one
   function = one reverse scan, with no extra structure.

Natural language lacks exactly (1) and (3): one concept has no canonical
name (synonyms, pronouns, paraphrase), so resolution can only be
approximate. That is the reason vector retrieval exists in a graph memory:
it is not a substitute for the graph but the entrance that covers identity
resolution; once identity is settled, everything downstream is
deterministic graph work.

The cost conclusion: **the cost of graph memory sits almost entirely in
BUILDING identity** (extraction → entity convergence → canonical names and
aliases), not in retrieval or traversal. In the code domain the language
and compiler pay that bill; the natural-language domain must pay it itself
— which is where the investment belongs, rather than in fancier retrieval
algorithms.

## Where the Property Graph Sits in Agent Memory

Agent memory must satisfy two contradictory requirements at once:

1. **Exactness**: architecture decision chains cannot be fuzzy-matched. "We rejected the microservice split" must be retrieved exactly, and must not return "we recommended microservices" because of semantic similarity.
2. **Evolvability**: knowledge changes. New decisions supersede old ones, and weights must adjust dynamically. (This means content goes stale and needs lifecycle management—not timeliness in the sense of "write latency"; for the latter see the section "Proactive vs. Full Extraction".)

Vector retrieval satisfies evolvability but not exactness. Wiki satisfies exactness but not evolvability (high update cost). The property graph sits in the middle—it builds exact relations at write time and dynamically ranks by weight at read time.

### Document vs. Atomic Fact

Document-style memory has three structural defects, which atomic facts resolve one by one:

**Semantic drift**: after a document is repeatedly modified, its topic shifts. Links/references that originally pointed to it still exist, but the target is no longer the same thing. A document written as "evaluation of plan A" becomes "implementation record of plan B" after a few edits, and every reference to "evaluation of plan A" in other documents forever points at the wrong target. Atomic facts do not have this problem—edges are immutable, new knowledge = new edges, and the semantic position of an old edge never changes.

**Granularity dilemma**: coarse granularity (a whole document) has low information density, with 80% of the content irrelevant to the current query; fine granularity (paragraph/sentence) degenerates into RAG mode but without an embedding index, making search more costly than vector retrieval. Atomic facts sit at the optimal granularity: one record = one fact, zero redundancy, zero noise. Queries are graph traversal (deterministic) and do not need vector search (probabilistic).

**Relation implicitness**: relations in documents are established by the reader's inference—"A caused B" may be spread across two paragraphs. Atomized extraction may lose implicit relations, but this can be mitigated in two ways: identify implicit causal chains during extraction; support multi-hop traversal at query time (transitive reasoning A → B → C).

**Update complexity**: a document update is an in-place modification, and the whole document must be read before the modification site can be found. Documents grow with updates, so each read grows, and the total read volume over n updates is O(n²). Atomic facts are append-first: new knowledge is written directly without reading old knowledge, so the total write volume over n updates is O(n). This is not only a performance issue but a correctness issue—the content read during a document update may already be stale (concurrent modification, cache inconsistency), whereas atomic appends have no race conditions.

| Dimension | Document | Atomic fact |
|:--|:--|:--|
| Semantic stability | Drifts (modification changes the semantic position) | Immutable (new knowledge = new edge) |
| Information density | Low (lots of noise) | High (1 fact = 1 record) |
| Query method | Keyword/vector (probabilistic) | Graph traversal (deterministic) |
| Relation explicitness | Implicit (requires inference) | Explicit (stored directly) |
| Reference accuracy | Degrades with modification | Permanently accurate |
| Update complexity | O(n²) (in-place modification requires reading the full text) | O(n) (append-first, no reading of the old) |

### The Retrieval Dilemma of Atomic Facts

Atomic facts solve the storage problem but introduce a new one: **retrieval cost**.

The consequence of too-fine granularity is not merely "fragmentation"—retrieval must traverse a large number of scattered edges to assemble a complete judgment, costing more than vector retrieval. A single architecture decision may consist of 10-20 edges, all of which must be pulled out and pieced together at retrieval time, so read cost grows linearly with decision complexity.

**Fundamental contradiction**: **storage must be atomized (exact, no redundancy), retrieval must be contextualized (coherent, with context).** A single granularity cannot satisfy both requirements at once.

### The Two-Layer Model

The solution is not to choose "atomic facts" or "summary documents" but to do both—the storage layer is atomized, the retrieval layer aggregates into clusters.

```
Write layer (atomic):      one edge = one fact, append-first
↓
Organize layer (cluster):  related edges grouped by entity/event/topic
↓
Retrieve layer (view):     return by cluster, not loose edges
```

Analogous to a database: raw rows are atomic, materialized views are aggregated. Queries go to the view, writes go to the row.

### The Two-Stage Extraction Pipeline

Extracting atomic facts directly from spoken/multi-party dialogue lets the signal drown in noise. A two-stage pipeline is used: normalize first, then classify and extract.

#### Why Direct Extraction Gets Messy

| Problem | Example | Result of direct extraction |
|:---|:---|:---|
| Spoken filler words | "Um, you know... I think this thing is pretty simple" | hedging mixed into the atomic fact, subject blurred |
| Coreference resolution | "He said that one won't work" | Who is "he"? What is "that one"? Cannot form an atomic fact |
| Multi-party attribution | A says X, B rebuts Y, C summarizes Z | Loses "who said it"; facts and opinions mixed together |
| Repetition/fragments | The same view stated three times, each with different wording | Three redundant atomic facts, hard to deduplicate |
| Implicit logic | "Wasn't that plan useless last time?" | Rhetorical question → negative judgment; direct extraction loses the semantics |

#### Stage 1: Normalization (Not Summarization)

Key: **normalization is lossless, summarization is lossy**.

Normalization does:
- ✅ Coreference resolution (he → Zhang San)
- ✅ Deduplication and merging (the same view three times → one)
- ✅ Attribution tagging (what A said vs. what B said vs. consensus)
- ✅ Spoken → written ("this thing is pretty simple" → "the task has low complexity")
- ✅ Category tagging: fact / pattern / logic / preference

Normalization does **not**:
- ❌ Summarize ("he gave three reasons in detail..." → the three reasons cannot be dropped and only one sentence kept)
- ❌ Rewrite semantics ("may" cannot become "certain")
- ❌ Over-filter (seemingly meaningless speech may carry tone/attitude signals)

#### Stage 2: Classified Extraction

Different categories use different extraction templates:

| Category | Atomic fact pattern | Example |
|:---|:---|:---|
| **Fact** | `(entity, relation, entity)` | `(Postgres, supports, pgvector)` |
| **Pattern** | `(condition, implies, result)` | `(high concurrency + write-intensive, implies, use PG not SQLite)` |
| **Logic** | `(premise, therefore, conclusion)` | `(permission = graph reachability, therefore, use PGQ property graph query for pruning)` |
| **Preference** | `(user, prefers, thing)` | `(user, prefers, conservative operations)` |

Preference weights are handled specially: `tool_invoke_count` does not apply (a preference is not a tool invocation), but `read_count` will be high (injected every round). A preference's `time_decay` should be slower—preferences are stable.

#### When to Skip Stage 1

Not all input needs normalization. If the input is already structured (code, configuration, formal documents, wiki), it goes straight to stage 2:

```
Input classifier (lightweight, can be rules rather than an LLM)
  ├── structured input → skip stage 1 → extract directly
  └── unstructured input → stage 1 normalize → stage 2 extract
```

#### Cost

Two stages = 2 LLM calls. But:
- Stage 1 can use a small model (normalization is syntactic and does not need strong reasoning)
- Stage 2 must use a strong model (extracting atomic facts requires semantic understanding)
- If the input is short (<500 characters), the two can be merged into a single call, with the prompt outputting two sections

#### Comparison with Cognee

Cognee does have a similar framing step—before extracting the graph it first performs a data → structured representation transformation. The two-stage pipeline aligns with existing practice. For a detailed technical analysis of Cognee and an item-by-item comparison with this document, see [Memory Architecture](agent-memory-en.md).

### Proactive Graph Memory: What Is Worth Remembering

The two stages constitute full extraction (building the complete graph without filtering for "worth remembering", relying on later weights to filter). A parallel mechanism is **proactive graph-building**—when the LLM judges during a conversation/task that "this is worth long-term memory", it proactively generates atomic facts and graphs them. The output is likewise an atomic fact graph (distinct from the proactive `memory_store` in agent-memory, which is a flat record).

Three mechanisms compared:

| Mechanism | Trigger | Filtering | Output | Storage |
|:--|:--|:--|:--|:--|
| **Passive two-stage** | System (at compression time) | No filtering, full extraction | Complete graph | Atomic facts (graph) |
| **Proactive flat** (agent-memory) | LLM judges "worth remembering" | Yes (LLM judgment) | flat | `memories` table |
| **Proactive graph-building** (this document) | LLM judges "worth remembering" | Yes + explicit criteria | Graph (persistent memory graph + task DAG) | Atomic facts (graph) |

The criteria for proactive graph-building ("what is worth remembering") reduce to **two orthogonal dimensions: specificity × generality**.

**Dimension one: specificity (strength of situational coupling)—the main axis for what to record.** The strength of coupling between knowledge and a particular situation (a project, a person, a time and place). Stronger specificity = more constrained, something the LLM does not know each time and must make up for via memory:
- **Person-related** (who said what, preferences, habits)
- **Time-and-place-related** (decisions and context at a certain time and place)
- **Project-related** (what choices a project made earlier and why)
- **Counter-mainstream derivation skeleton**: the conclusion itself may not be new, but the attached **logic chain is constrained**. What is stored is not the conclusion but the chain of "why it was derived this way"—this step uses "mind-map it → organize into a logic DAG skeleton → add node by node as atomic facts", using the logic chain as the skeleton.

**Dimension two: generality (breadth of coverage of the knowledge itself)—high does not necessarily mean record.** The breadth of coverage of the knowledge itself. The criterion is **"whether it will always be known reliably"**, not "whether the LLM currently knows it":
- **High generality + reliably reproducible = universal law** (e.g. HTTP status code semantics) → do not record; the LLM regenerates it every time, so recording it is waste.
- **High generality + constrained logic chain = counter-mainstream derivation skeleton** (see dimension one) → record, but what is remembered is the derivation chain, not the conclusion.
- Generality itself does not determine keep-or-drop—it must be **considered together with reproduction stability and the logic chain**. A law with broader coverage is more important on the abstract evaluation (value layer), but high generality and stable ≠ worth remembering (memory layer); the two layers must not be conflated.

**Coupling of task graphing (DAG) with the criteria**: graphing tasks as a directed DAG is not just task orchestration—it is simultaneously the natural container for the **project dimension** of the "specificity" criterion. Each atomic fact hangs under a task/project node and automatically carries the specificity dimension of "which project it relates to". The criteria and the storage structure complete each other: the graph structure itself is the landing point for "what to record".

### Proactive vs. Full Extraction: Division of Labor by Content Type

First, clear up a common misconception: **proactive extraction is not "narrow seeing"**. Its context window length is adjustable, and it can see the current compressed version (which already contains the complete global picture). So "local vs. global" is not the dividing line between proactive and full—the real difference between the two lies in **the output target and the timing of judgment**, not the breadth of the view. The judgment is "should this be recorded by the standard" (proactive) or "let everything land, filter later for what is worth it" (full).

Accordingly, each is assigned the method suited to it by **content type**:

| Content type | Suitable method | Timing of judgment | Note |
|:--|:--|:--|:--|
| **Emphasis signals** | Proactive (real-time) | On the spot | Preferences, the user's "remember this", a single core decision—can only be identified when they occur; full extraction afterward cannot see "which one was emphasized" |
| **General knowledge graph** | Full | After the fact | **This part the LLM can largely replace**—highly general and stable knowledge the LLM already has, so recording it has low value (corresponding to the criterion "high generality + reliable reproduction → do not record") |
| **Unique logic chains (derivation skeletons)** | Full | **Offline batch** | Requires seeing the complete thing to extract structure; but it is not a high-frequency real-time need, so it can be deferred to the background, in batches, offline, avoiding the peak cost of full extraction |

Key points:

- **Proactive = the real-time emphasis channel**: captures high-confidence emphasis that "happens now and can only be identified on the spot". Fast, exact, clearly targeted.
- **Proactive is also the lowest-latency publication pipeline**: write latency is compressed to the RTT of one tool call—visible to the whole graph the moment it happens. Passive extraction's persistence time is bound to compression; in a single-consumer scenario that does not matter (it is only needed by its own next session), but **in a multi-consumer scenario the latency is a hard flaw**: when multiple agents collaborate over a shared memory graph, a key decision in Agent A's session must wait for compression to be persisted, and during that time what B retrieves is always a lagging graph that excludes the current session; the same holds for cross-constraints between parallel projects ("B is waiting for A's interface to be finalized") and for team-shared workspaces—the longer the session, the longer the results are invisible to other readers. The identification argument (can only be identified on the spot) and the visibility argument (latency is a hard flaw) are two independent arguments, and the latter carries more weight in multi-consumer scenarios.
- **Full = a structural product**: its value is not in the general graph (which the LLM can replace), but in the **logic skeleton** (which only the complete view can extract). The skeleton does not challenge real-time requirements, so it should be done **offline in batches**—gaining the structure without paying the online cost. Nor is the skeleton a real-time dependency of collaboration: what consumers need is constraints and decisions, not the derivation process itself.
- **The judgment of "what to record" is not a competition between "proactive vs. full"**, but a matter of finding the right method for each content type: what needs emphasis goes proactive (real-time), what needs a full-view structure goes full (offline). The two are not mutually exclusive; they are a division of labor.

This is self-consistent with the criteria above: proactive falls mainly on the emphasis signals of "specificity"; full mainly serves the derivation skeletons of "constrained logic chains". The division holds in both the single-consumer scenario (write latency irrelevant) and the multi-consumer scenario (latency is a hard flaw, and the weight of the proactive channel is raised further), only with different weights.

### Three Clustering Strategies

#### 1. Entity-Anchored

All edges mentioning entity X naturally hang on the X node. At retrieval time, find the entity → return its neighborhood subgraph → the subgraph contains coherent context.

```
Entity node "NixOS module ownership"
├── edge: split by function, not by implementation language
├── edge: lsp-bridge goes in system/units/
├── edge: pop-launcher merged into cosmic.nix
```

Applicable scenario: querying "all facts related to X". Analogous to a graph neighborhood query.

#### 2. Temporal-Clustering

All edges of the same event/decision, grouped by time window. Solves queries like "what was considered at the time of a certain decision".

```
Event node "2026-07 NixOS Overlay architecture refactor"
├── edge: previously scattered directories → high maintenance cost
├── edge: introduce developMode gating
├── edge: vivaldi.nix as the standard template
└── edge: write ADR-011 to record the decision
```

At retrieval time the complete causal chain is pulled out by event ID, rather than pieced together from twenty loose edges.

#### 3. Causal-Chain Clustering

Grouping by the "why" reasoning chain. Not by entity (who), not by time (when), but by causal logic (why).

```
Judgment node "permission reachability selection"
├── edge: permission = graph reachability pruning
├── edge: property graph declared with PGQ (CREATE PROPERTY GRAPH)
├── edge: one graph, many relations → same trust domain merged into one graph
├── edge: GRAPH_TABLE hybrid traversal → along membership+hierarchy to skill
└── edge: strict edges locked by table permissions → user-organization relations manually written only
```

Each edge of this chain has limited meaning when retrieved alone, but together as a causal chain it is a complete technical judgment.

### Strategy Selection

| Strategy | Retrieval mode | Analogy |
|:--|:--|:--|
| Entity-anchored | "all facts related to X" | Graph neighborhood query |
| Temporal | "the decisions around that event" | Temporal index |
| Causal chain | "the complete reasoning of this judgment" | Materialized view |

In practice, an edge may belong to several clusters at once (the same fact both hangs on an entity and lies in a causal chain). This is not redundancy—it is the projection of the same atom in different views.

**Core insight**: atomized storage guarantees no redundancy; clustered retrieval guarantees context. A cluster is a view, not a copy, so it does not violate the atomization principle.

### Fusion of Atomic Facts: Redundancy Control When Entering the Graph

The above solved **extraction** (two-stage/proactive criteria) and **retrieval** (two-layer/clustering). Here we add the third link: **when new atomic facts enter the graph, how to merge with existing ones, deduplicate, and avoid redundancy dissipation**.

**Problem**: atomic facts have limited granularity, and newly extracted nodes may not be atomic—the same entity splits into multiple nodes across time and across sessions (entity splitting), or the same fact is extracted repeatedly with different wording. This is redundancy across writes, which stage 1 deduplication (which only covers the current input) cannot reach.

**First, split entity classes: not all entities need search-based alignment.** Before fusion, first determine which class the fact's `subject`/`object` belongs to; the two classes are handled completely differently:

| Entity class | Source of identity | Alignment method | LLM involvement |
|:--|:--|:--|:--|
| **In-domain entity** (organization member, project, task, **store product**) | Stable database id | **Deterministic, direct id lookup, no search** | None |
| **Out-of-domain entity** (free knowledge object, external concept) | Newly created by AI at extraction time | Normalized key + embedding fallback | Fact novelty only |

**In-domain entities pose no search problem**—they have a stable id, so judging "is this Zhang San the Zhang San" or "is this product that product" is not a collision problem at all; it is simply a direct hit by id: recognize that the fact's `subject`/`object` points to an in-domain entity, look up its id, and attach it—zero embedding, zero retrieval. Entities like organization members and products are **directly exempted from the search step** during fusion. Only loose knowledge objects with **no stable identity in-domain** need the search-based alignment below.

**Approach: the layered-cost principle—cheap retrieval is responsible for finding the needle in the haystack, expensive LLM decisions face only a small candidate set (for out-of-domain entities only).**

```
Out-of-domain entity facts after extraction (in-domain entities were routed by direct id lookup in the previous section and do not enter this flow)
    ↓
Embedding retrieval (cheap, large volume OK): semantically search the graph by {subject, predicate, object}
    → return candidate facts + the entity nodes they anchor
    ↓
LLM decision (expensive, only for single-digit candidates): judge one of three cases
    ├── reuse the existing subject node, add only a new edge (entity already exists, this is a new fact)
    ├── full merge/discard (fact already exists, merely different wording)
    └── create a new node + edge (genuinely new knowledge)
```

**Why this layering is right**:

- **Embedding retrieval is cheap**, so it can return many candidates—large volume is not a concern; it handles "finding the needle in the haystack", pulling out all possibly relevant entities/facts.
- **LLM decisions are expensive**, so the huge graph must never be thrown at it—hence retrieval first converges it to single-digit candidates, and the LLM judges only those.
- The key: **the retrieval must return the entity nodes that the candidate facts anchor, not just similar facts**—otherwise the LLM cannot see that "Zhang San already exists" and misjudges a "new fact" as a "new node", thereby creating entity splitting. Retrieval converges by entity anchor, and the LLM decides by entity alignment.

**Cost accounting**: one extra embedding call (extremely cheap) + one LLM judgment (only for a few candidates, small tokens). Relative to a one-off "pure append" that ignores fusion and lets entities split without bound, this is an extremely cheap insurance—what it prevents is the long-accumulating explosion of node count and retrieval dilution.

**Relation to the two-layer model**: fusion is **write-side** structural control (ensuring atomized storage is realized); clustering is **retrieval-side** view organization (ensuring retrieval has context). The storage side does not rely on clustering for deduplication—it relies on the fusion here; clustering only does retrieval projection, and the two do not conflict.

### The Weight System

Each edge carries system-level metadata:

```
Business attributes (relation_props):  since:2011, scope:global, reason:xxx
System attributes (weight_meta):       read_count, write_count, last_read, last_write
```

The two layers do not mix. Business attributes describe "what this piece of knowledge says"; system attributes describe "how important this piece of knowledge is".

#### The Read/Write Ratio Matrix

read_count and write_count form a two-dimensional coordinate that defines four classes of knowledge:

```
             write-heavy (popular)
                       │
            Noise zone │     Core zone
  low read, high write │ high read, high write
 popular but unqueried │  repeatedly confirmed, repeatedly used
 → consider downweight │    → highest weight
                       │
   ────────────────────┼────────────────────
                       │
             Cold zone │      Deep zone
   low read, low write │ high read, low write
             low value │   queried but high barrier
 → demote to cold data │  → high weight, rarity bonus
                       │
              write-light (depth)
```

| Quadrant | read | write | Meaning | Handling |
|:--|:--|:--|:--|:--|
| **Core zone** | high | high | Knowledge used and confirmed repeatedly | Highest weight, returned first |
| **Deep zone** | high | low | Queried but with a high write barrier (expert knowledge, deep-water traps) | High weight + rarity bonus; does not penalize low writing |
| **Noise zone** | low | high | Written frequently but queried by no one (stale decisions, invalid records) | Downweighted, marked as noise candidates |
| **Cold zone** | low | low | Low-value knowledge | Demoted to cold data, spills to S3 |

**Key insight**: low writing does not mean unimportant. Deep-zone knowledge (e.g. deep-water traps of a particular tech stack) has a high write barrier—only those who have been burned will write it, but every query is a high-value hit. Traditional weight formulas penalize such knowledge with write_count; the read/write ratio matrix corrects this bias.

**Guarantee of read/write independence**: atomized facts naturally decouple reads from writes. In document mode, read and modify are the same operation chain (read++ and write++ bound together), making reads and writes positively correlated and collapsing the 2D matrix into 1D. In atomic-graph mode, read = a traversal query over existing edges (read_count++), write = adding a new node/edge (write_count++). The two operations act on different physical objects, and existing edges are unaffected by new operations. This is the architectural advantage of atomization—the four-quadrant classification is stable at the physical layer.

#### Weight Calculation

```
weight = read_score + write_score + rarity_bonus - time_decay

read_score  = log(1 + read_count)        # logarithmic decay, prevents head domination
write_score = log(1 + write_count) × 0.5  # write weight lower than read
rarity_bonus = 1.0 if read_count > threshold AND write_count < threshold else 0  # deep-zone bonus
time_decay  = days_since_last_access × 0.01  # slow decay
```

High-weight edges are returned first at query time. Low-weight edges are not deleted but demoted to cold data.

### The Write-Read Loop and the Extraction Mechanism

The write-read loop, the three designs of the extraction mechanism (parallel extraction → pure append instructions → Prefix Checkpoint), the details of checkpoint compression, the fact serialization format, the extraction prompt, and the isomorphism of encoding scenarios—these belong to implementation mechanisms and have been moved to [Krystallizer](krystallizer-en.md), where the original content is preserved in full.

## Read-Time Conflict Resolution

Traditional memory designs (LangChain, Mem0, etc.) treat conflict as a system fault: the write path goes through read-modify-write, maintaining a single truth by rewriting or overwriting, and conflicts are forcibly eliminated at write time. The atomic fact graph takes the opposite stance: **conflict is not a fault but a read-time problem awaiting resolution**.

### Writes Have No Conflicts

Facts are an append-first record of objective events. Two opposite facts (different budget amounts, opposing architecture decisions) are two independent appends in physical storage; there is no write conflict—appends are naturally lock-free (see [KV Storage Engine](kv-storage-engine-en.md) §read-modify-write: delta appends eliminate the RMW race). Logical conflict exists only at the moment the reader assembles the view; the write layer never separates truth from falsehood.

### Resolution Relies on Weights and Routing, Not on Write-Time Adjudication

- **Weight accumulation**: each fact carries a weight, and retrieval sorts by weight. Facts that recur and are corroborated by multiple sources gain weight, and isolated facts naturally sink—the authority to adjudicate conflicts is delegated to the accumulation of evidence, not to the order of a single write.
- **Actor/domain attribution**: each fact node carries the subject to which it belongs. If opposite statements of the same question belong to different subjects, they are not a conflict but two positions—the retriever adopts according to its own perspective, and the two coexist in the graph.
- **Derived views at retrieval time**: retrieval (vector landing + the community the landing falls into) returns a set of facts ranked by the current query context, and the view is assembled at read time. The same graph derives different views for different queries—this is a direct corollary of "read-time computing" in conflict resolution.

This stance is consistent with the computational-timing spectrum: conflict resolution is a read-time operation. Moving it to write time (separating truth, overwriting, RMW rewriting) means giving up all the benefits of append-first and returning to the old path of traditional designs.

## Graph Skills

### Graph and Network: Two Lenses on the Same Structure

A graph and a network are mathematically isomorphic—both are nodes + edges; the divide is neither in storage nor in data structure, but in **operating mode**. The two are not two kinds of memory or two storage substrates, but two sides of the same coin:

- **Graph = static presentation**: what is stored, what the structure looks like, for traversal / matching / reasoning. Serves deterministic queries.
- **Network = dynamical process**: what flows between nodes, how it transmits, how it grows, how it is adopted. Serves propagation / diffusion / growth.

**The lens of extension is the network**. The essence of "extension" is diffusion / adoption / popularization—a thing propagating from a few nodes to more nodes until widely accepted, a typical propagation, connectivity, centrality problem that belongs to network science, not the territory of graph theory. Whenever the goal is "to spread memory / skills to more people, more organizations", the network way of thinking is right.

The growth and propagation mechanisms of this memory design are, from the root, network dynamics, not graph theory—all the following mechanisms use network methods:

| Mechanism | Network science method |
|:---|:---|
| Emergent skill boundaries | Community detection (Louvain) partitions natural boundaries |
| Cold start | PageRank-style dangling-link initial weight (propagation / centrality idea) |
| Organization-level sharing | membership edge O(1) = propagation channel |
| Dream consolidation | Global review of high-frequency subgraphs = reconstruction of the whole graph's diffusion structure |

"Emergence" is itself a network science concept (structure emerging in dynamics), not graph theory.

**Precise statement**: the memory's **presentation substrate is the graph** (what is stored, traversal and reasoning), and the memory's **growth / propagation / adoption is network dynamics** (how it grows, how it spreads, how it is adopted). One graph, two operating modes—use topology to answer "what is stored", use network science to answer "how it diffuses".

Down to the last layer of "extension": it is not about switching to another storage, but about designing the very act of extension with network methods—finding which skills are propagation hubs (centrality), which skills naturally form a group worth extending together (community detection), and which propagation channel the membership edge is (contagion path). The storage substrate does not need to change from graph to network because of this—the graph is the static presentation of memory, the network is the propagation mechanism of memory, two sides of the same coin.

### The Scale Criterion: Why the Graph Inevitably Wins at Scale

The difference between traditional designs and the graph design at scale is not "a difference in degree" but **a difference in complexity**—one is quadratic degradation, the other sublinear payoff. This is the structural root of why the graph gets better with use while the traditional gets harder, not an implementation issue.

**The true premise of permissions**: traditional ACL is a `user ↔ skill` pairwise relation table, with maintenance cost `O(users × skills)`, both dimensions growing → combinatorial explosion, quadratic degradation. This is not a failure of some team's maintenance—it is that a pairwise relation table physically cannot hold up under two-dimensional expansion. The graph turns permissions into **graph connectivity**: add a person = one membership edge (O(1)) inheriting all of the organization's capabilities; add a skill = connect to the organization to be shared with (O(1)) spreading it to everyone. Cost is linear in entities, benefit is superlinear in connection density—**structured sharing amortizes, and the larger the scale the more worthwhile**.

**The true premise of loading**: traditional full loading loads all SKILL summaries, tokens = `O(N) × per round`, and the larger N the worse the degradation. The apparent mitigation—ACL filtering—does not cure the root: it is a "reduced view", not "on-demand production", and the more say and the more skills a user has, the more is left after filtering and the heavier the pressure. The only correct solution is **dynamic retrieval** (look it up on the spot, Top-K injection): tokens drop from O(N) to O(K), and K does not grow with the total scale N.

**The true premise of creation and updating**: SKILL functionality keeps expanding and boundaries inevitably overlap, and full maintenance is likewise quadratic. The fallback "global scan and rewrite" is feasible only for purely document-type skills; **an enterprise SKILL with scripts is real executable code (with interface contracts, side effects, state), which cannot be synthesized/rewritten and can only evolve incrementally**—relying on structured graph-boundary management (emergence + dreaming + human review), not full rewriting.

> One-sentence criterion: **ACL = pairwise combinatorial explosion (harder with use at scale, physically hitting a wall); graph = structured sharing amortization (better with use at scale, cheaper the denser the connections).** Any design should first answer "at scale, does it get better with use or harder with use".

### The Control-Target Criterion: ACL Has No Clear Control Target

The scale criterion discusses "how cost degrades"; the control-target criterion discusses "whether there is anything that can be authorized". The two run in parallel, and neither can be missing.

An ACL authorization row is essentially `(subject, object)`—"who can do what to which resource". The subject (user/organization) is still well-defined in an emergent architecture; but **the object (a skill) is precisely what is not well-defined**. ACL implicitly requires the object to be a resource unit with a fixed boundary, enumerable and addressable, onto which "an access right can be nailed". An emergent skill, however, is a subgraph of a mass of usage data, whose boundary evolves with real use through merge/split: at authorization time "Alice may use skill_cart", and after evolution skill_cart merges with other skills into a new subgraph, so that object no longer exists in reality—the ACL row is nailed to a vanished object. **ACL has no clear control target; it is not that the implementation is poor, but that the "resource unit with a well-defined boundary" it presupposes does not exist in an emergent architecture**; it freezes a flowing emergent structure into a single static snapshot as the basis for authorization, and every time the snapshot goes stale it becomes the seed of the next configuration error/permission hole. The graph changes the control target from "a resource unit" to **"the subgraph reachable from the subject along membership/hierarchy (topological relation)"**—topology does not require a skill to have a fixed boundary; reachable today, and still reachable after tomorrow's evolution upon re-traversal, never nailed down.

### The Problem: The Human Quandary of Skill Boundaries

When Hermes autonomously evolves and writes skills, it easily produces a lot of duplication—skill scopes easily overlap, and large-scale scanning to re-partition boundaries is too costly and too slow (the token cost of an LLM scanning all skill descriptions is unacceptable).

Hermes's current skill management is **manual boundary-drawing**: humans judge boundaries → write SKILL.md → static loading (all skill summaries injected every round). The problems are:

- **Boundary overlap**: when the LLM writes a new skill it does not fully know the boundaries of existing skills, causing duplication
- **High scan cost**: re-partitioning boundaries requires the LLM to read all skill descriptions, token cost O(N) × frequency
- **Token bloat**: injecting all skill summaries every round bloats the context

### Tool Index

Core idea: give each edge a **tool index**—an edge loaded when the agent invokes a tool gets `tool_invoke_count++`.

This is essentially a **subclass** of graph-memory's existing `read_count`: not all reads count, only **reads triggered by a tool invocation** count. A tool invocation shows that this piece of knowledge has **actual execution value**, not just "a glance".

```
System attributes (weight_meta): read_count, write_count, tool_invoke_count, last_read, last_write
```

`tool_invoke_count` is independent of `read_count`, and the weight formula gives `tool_invoke_count` a higher coefficient:

```
weight = read_score + write_score + tool_score + rarity_bonus - time_decay

tool_score  = log(1 + tool_invoke_count) × 1.5  # tool invocation weight highest
read_score  = log(1 + read_count)               # passive read
write_score = log(1 + write_count) × 0.5        # write weight lower than read
```

### Emergent Skill Boundaries

Edges with a high tool index naturally cluster into a "frequently used operation chain"—this is the **implicit definition** of a skill. A skill is not a manually drawn boundary but a subgraph **emerging** from usage data.

Compared with Hermes's current skill management:

| Dimension | Manual boundary (current) | Emergent boundary (new design) |
|:---|:---|:---|
| Boundary drawing | Manual judgment → SKILL.md | Automatic clustering of usage data → subgraph |
| Loading method | Full injection every round | Vector search → high weights surface |
| Token cost | O(N) × per round | O(K) × per round (Top-K) |
| Update | Manual rewrite | Usage data automatically updates weights |

**Potential problems**:

1. **Cold start**: a new skill has no usage data yet, its tool index is 0, and it will not be retrieved. A bootstrap mechanism is needed—newly written edges get an initial weight (similar to PageRank's handling of dangling links).
2. **Skill fusion**: two high-frequency subgraphs may have overlapping nodes—merge or keep separate? A **community detection** algorithm (e.g. Louvain) is needed to partition natural boundaries, rather than relying on a threshold alone.
3. **Multi-membership**: the same tool may be invoked by several different skills—a high `tool_invoke_count` on an edge does not mean it belongs to only one skill. An edge can belong to multiple subgraphs (the projection of the same atom in different views, consistent with graph-memory's clustering design).

### Dream Consolidation: Offline Global Reconstruction + Human Review

Emergent boundaries are produced at runtime, but the **long-term evolution of boundaries cannot rely on online increments**—increments only handle the local and cannot see global drift. Set up an offline phase every night ("dreaming"): run **global community detection**, review the whole graph's high-frequency subgraphs, reconstruct the SKILL body boundaries, and produce an **operation set** (merge / relink / split) sent to the manager for diff review and application.

**Body granularity**: the SKILL body = a **first-class superset node** (a special type) whose content is a **coherent procedure/description** (not scattered into atoms)—invocation gets a self-consistent block of content; but as a node its form can be reconstructed offline and can be associated with other nodes (experience/rule/precondition neighborhoods). Coherent content preserves integrity, the graph node preserves evolvability—avoiding the document trap of "complete means stable, and stable means hard to evolve" (see §"Document vs. Atomic Fact").

**Closed loop**:

```
Runtime: emergent boundaries + tool_invoke_count accumulation
   ↓ every night
Dreaming: global community detection + usage weighting → propose an operation set (mostly merge/relink, high-confidence small batches)
   ↓ send to the manager
Review: inspect operations one by one in diff form (reversible commands, not final text) → approve / reject
   ↓
Apply: approved operations change the graph boundaries (write-time heavy, offline amortized); rejected operations are written back to the dreaming input → not proposed again next time
```

**Compared with manual boundary-drawing**: boundaries are not presupposed but grow out of usage data + offline global reconstruction; evolution does not rely on a full scan every round but on the amortized offline compute each night; accuracy is backstopped by "human review of operations"—filling in the verification layer of the synthesis closed loop (see [Memory Architecture](agent-memory-en.md) §7).

**Five constraints**:
1. **Write-time heavy → inherently one notch behind**: boundary reconstruction only lands at "night + after review", so runtime boundaries have a one-day drift window; acceptable for slow-evolving things like SKILL—this is an eventually-consistent boundary, not real-time.
2. **Precision bias, small batches**: each night propose only high-confidence, low-risk operations, prioritizing merge/relink, to avoid aggressive splits flooding the review.
3. **Usage-weighted, not purely topic similarity**: use `tool_invoke_count` + read/write ratio as signals, and act only on boundaries that are "proven hot" or "proven cold"; deep-zone (high read, low write) stable objects are by default left untouched unless clearly better.
4. **Rejected operations flow back**: operations rejected as merge/relink are written back to the dreaming input, otherwise they recur every night—rejection is half of the closed loop.
5. **Single canonical, no files**: the graph node is the logical entity (the single source), with no file materialization export; when a "document view" is needed, a function call queries the graph and **renders and returns** it—no export artifact, no dual-representation drift (the engine does not mix reads).

### Progressive Loading

Instead of loading summaries of all skills, search directly (vector/text), and in the results, items with a high tool index automatically count as part of the skill.

This solves Hermes's current problem: injecting all memory + all skill summaries every round → token bloat. Change to vector search → rank by weight → inject only the Top-K high-weight edges.

Prerequisite: the skill description itself must be atomized into the store. Currently SKILL.md is a markdown document and must be split into atomic facts before it can participate in vector search.

### Script Management: Two Designs

#### Design A: Script as a Node, the SKILL Description Points to It

Each script is a node (including how to invoke it), and part of the skill description points to it. This is the **reference pattern**—the skill is metadata (description + pointer) and the script is the entity (code + invocation method).

```
[Skill node] --references--> [Script node]
  ├── edge: description (natural language)
  ├── edge: trigger condition
  └── edge: points to the script node (lang, entry_point, params schema)
```

Compatible with aura's `set()` + `interface_schema()` design—scripts are stored in the Fjall `booth_defs` partition, skill descriptions are stored in graph nodes, pointing to the script node's `booth_type_name`.

#### Design B: Scripts Split into Atomized Booths, AI Wires Them Together

Split the script itself into atomized booths, and the AI wires them together, adapting to the Aura architecture. Each execution generates a field. Aura's design may need to be revisited—no longer event listening, but call-by-reference.

> ⚠️ **Pending**: this design involves adjustments to the Aura architecture and needs further discussion. The core question is: Aura's `emit`/`on` is event-driven (pub-sub), while "call-by-reference" is imperative (RPC); how do the two coexist.

### SKILL Topologization: Four-Part Nodes + Deterministic Materialization

A SKILL goes from "a piece of text" (SKILL.md) to being structured into a set of **four node types**; storing the four parts is the structuring:

| Part | Node type | Belongs to |
|------|----------|------|
| Trigger condition | Condition node set (boolean-combinable) | SKILL graph node |
| Numbered steps | Ordered step nodes | SKILL graph node |
| Invocation contract | Interface/parameters/entry | **Booth node** |
| Trap experience | Bound to specific steps/contract layer | SKILL graph node |

**Booth**: an executable primitive, implementable in any language such as python / lisp, stored **inside the graph (logical entity)** as a first-class node—not a code pointer, but an atomic unit that can be summoned and retrieved together with triggers/steps/experience; the abstraction granularity is the booth (see [Aura Architecture](aura-architecture-en.md)). Here "code as node" refers to the **booth primitive** (executable unit), which is a different matter from CodeGraph recording **code structural semantics** (what/why); the two are not conflated.

**Invocation flow = the LLM makes only two judgments, everything else is deterministic**:

```
1 Dispatch: search only within the "trigger condition" type → pass similar top results to the LLM for selection
        (type-scoped—trigger search does not enter the type space of steps/contracts/traps)
2 Materialize (not LLM): once selected → step nodes deterministically expanded into a task checklist
3 Execute (LLM): verify completion item by item; completion judgment belongs to the DB; each step calls its corresponding booth contract
4 Traps: each step injects the trap experience bound to that step; contract/booth-level traps are injected on their invocation side
```

**Deterministic boundary**: the LLM is responsible for "choosing the trigger + executing item by item"; materialization is template instantiation and completion is a DB judgment—the LLM does not judge process completeness.

**Three constraints**:
- Trigger search is **type-constrained** (searches only trigger condition nodes); the trigger condition itself is structured (boolean-combinable, decidable) to reduce LLM selection ambiguity; cross-SKILL trigger overlap is resolved at the boundary layer by "dreaming/community detection".
- Step nodes carry two states: call a certain contract of a certain booth, or a pure reasoning/information step—materialization determines whom to call.
- Traps are divided into two levels: step level (attached to the step node) + contract/booth level (attached to the invocation side).

**Extensible parts (orthogonal, added as needed)**: the four parts are the backbone of "dispatch + execution"; there are also three orthogonal items, each with its own dedicated step, to which corresponding node types are added when needed:
- **Preconditions / dependencies**: participate in dispatch (excluded if unmet) and materialization (merged into the task check—the gate's material).
- **Boundary / negative space**: participate in dispatch (negative trigger: scenarios that should be rejected / handed off, i.e. not done).
- **Delivery acceptance**: participate in the semantic layer of completion judgment (the task's DB judgment is "mechanically complete", acceptance is "deliverable up to standard").
- **Examples (few-shot)**: as needed, participate in injection.

**Criterion for adding a type**: whether a part is worth becoming a node type depends on whether it serves a dedicated step among **dispatch / materialization / injection / completion judgment**; those that cannot answer this (name, version, category, etc.) are metadata, belonging to node attributes and system weights, not node types.

---

## Dynamic Summoning: Retrieval-Augmented Skill Context

When executing a SKILL, it does not use only its static definition but retrieves the **experience and rules** most relevant to the task from the graph and injects them as context—this is the invocation-side use of "vector search first, then read the neighborhood by graph (community detection)":

- **Steps**: query (task description) → vector hits candidate edges/nodes → expand the neighborhood by graph (entity/causal-chain clustering) → rank by weight and take Top-K → inject into this execution's context.
- **Deterministic materialization**: procedural content is stored structurally as four node types (see §"SKILL Topologization"); after triggering, it is expanded into a task checklist via template instantiation (not LLM), and what is injected is only peripheral situational knowledge (related experience, preconditions, reference cases).
- **Ranking signals**: which to inject, in what order → `tool_invoke_count` + read/write ratio (the weight system): hot (core zone/deep zone) is given first; cold/noise is not injected.
- **Compared with static loading**: static = full injection of summaries every round (a lot of the irrelevant, token bloat, fixed boundaries); dynamic = bring in only what is relevant this time (precise, small context, boundaries evolving with use). The cost is **the retrieval cost of each invocation**—so it pays off only for the "core zone/deep zone"; a cold skill is neither dynamically summoned nor resident.

## Task Networking (Task Forest)

The **instance state** of tasks does not enter the write-time-heavy graph, but tasks can have a **network structure**, with state and type placed in two places:

- **Type (KG, slow variable)**: task templates, patterns, preconditions, references—task nodes hold references to shared context.
- **State (task table, fast variable)**: current instance, items, `claimed_at` (written on claim), `completed_at`—high-frequency writes, short lifecycle, kept in dynamic storage; the high-churn instance state is not poured into the write-time-heavy graph (see §"The Computational-Timing Spectrum").

**Structural evolution**: linear checklist → directed DAG.

- `parent_task_id`: subtask decomposition (tree-shaped).
- `task_links`: cross-task dependency edges ("A first, then B").
- Completion derivation is unchanged: task completion is still = no incomplete item; networking only adds structure, not a state machine.

**gate: pre-execution validation for script execution (one of the consumption scenarios of a task).** Besides carrying task state, `tasks/task_items` is also used for **pre-execution validation of script execution**—before executing a SKILL script, the agent first checks whether the task has an incomplete item; if not satisfied, it refuses to execute and attaches the checklist, and the agent stops upon reading it and runs again after filling it in. Completion judgment belongs to the DB (satisfied when there is no incomplete item), the gate is lazy, and it triggers only at the script execution entry. This is one scenario of the gate—the same task structure can also carry other pre-execution validations (parameter completeness, cross-task dependency readiness), extended as needed. On the engineering side (skillforge) there is additionally an explicit `task_id` (primary) + `skill_name` fallback, and the interception semantics of "returning an error object means natural stop"; these implementation details are not expanded here.

**Why not message passing: loss is the root problem.** The failure root cause of graph coordination's opposite—message passing between agents—is **information transfer loss**: information is distorted each time it is passed. This is not a defect unique to LLMs; distortion in person-to-person relayed messages and in hierarchical level-by-level reporting is a general law of communication media; the LLM merely magnifies the loss rate sharply (natural language is the only interface, no acknowledgment read-back, Context truncation silently drops information). The quality upper bound of a multi-agent system is determined by "hop count × per-hop loss rate", independent of the number of agents—graph coordination compresses the hop count to zero: each agent reads runtime facts directly, without any intermediary relay.

**The essence of multi-agent is a parallelism problem.** Stripping away the "multiple AIs in a meeting" wrapping, the real demand of multi-agent is to increase parallelism, and parallelism does not need inter-agent communication—its requirements are that the task be decomposable and the results mergeable. Message passing couples "parallel execution" and "coordination communication" rigidly into the same mechanism (natural language dialogue), and the coordination loss eats the parallelism gains. The solution is to decouple: execution in parallel, coordination through a shared structure—which is exactly the role the task graph plays. For an in-depth critical analysis see [Multi-Agent Critique](multi-agent-critique-en.md).

**Coordination medium: the task graph replaces message passing (multi-agent).** The task graph is the **shared view** of multi-agent division of labor—each agent reads the graph to claim tasks, change state, and leave items; if information is insufficient it checks the graph itself rather than passing messages. Message passing looks direct and efficient, but the number of channels grows combinatorially with the number of decision-makers (N agents pairwise = N(N-1)/2 channels), and the record drifts; routing everything through the graph converges to N-to-1, and the state on the graph is a runtime fact. This is the application of "observation surface replaces declaration" at the organizational level. Two boundary conditions:

- What the graph replaces is **state synchronization**, not **decision consensus**. "What it is, where it is stuck, which item is missing" the graph can carry; "why do it this way" relies on references in the type layer (templates, preconditions, experience edges). When the type layer is left empty, graph coordination degrades into replacing meetings with collective silence—information disconnection exists in another form.
- The coordination medium requires **strong consistency** of state transitions: when multiple agents concurrently claim/update the same task, without conditional updates (CAS or a single-writer reducer) the coordination surface drifts first. The task table is the natural candidate for "control-plane single writer".

**Parallelism comes from topology, not from the scheduler.** Two task nodes with no dependency are naturally concurrent on the graph: Gravity starts from the current event and performs one topological pruning along `task_links` (nodes without a strong `DEPENDS_ON` dependency branch off and execute independently); the unit of parallelism is the branch structure of the graph, not the message orchestration between agents. This is isomorphic to the compression parallel bypass—the same prefix branches into multiple independent execution branches, each appending its own writes without overwriting others. The precondition for concurrency safety falls on append-first writes: facts are appended into the graph as deltas (see §"Read-Time Conflict Resolution"), with no write conflict at the physical layer; concurrent updates of task instance state are still bound by the strong consistency of the coordination surface (the CAS condition above).

**Organizational ledger (audit surface).** When most of the work in a project goes through the task graph, who / when / which task / what become structured facts—the task table is the organization's event log, and artifacts like weekly reports and standups are its lossy compression. Tracking after agents take on work is a zero-cost byproduct, provided each record carries attribution fields (actor identity + timestamp), otherwise the ledger is not trustworthy. The same data serves two purposes: for the agent it is a coordination medium, for the organization it is project management.

---

## Relation to LLM Wiki

LLM Wiki is the extreme of write-time computing—manually organized, fixed structure, deterministic reads. It is the "manually pre-warmed version" of the property graph.

Once a Wiki's content is extracted into atomic facts and stored, it migrates from write-time computing to the middle zone:

- Write time: Wiki source text → atomic fact extraction → storage (one-time cost)
- Read time: graph traversal + weight ranking (continuous payoff)

**Payoff of migration**: the Wiki's knowledge gains dynamic weights and can automatically adjust importance by usage frequency. It is no longer necessary to manually judge "which piece of knowledge is more important"—the usage data will tell you which.

**Cost of migration**: the precision loss of atomic fact extraction. Implicit relations in prose may be lost during extraction. This is the inherent price of migrating from write-time computing to the middle zone.

## Storage Implementation

The complete implementation design of the KV route (property graph encoding patterns, delta appends eliminating read-modify-write, secondary index updates, WriteBatch transactions, vector and full-text retrieval supplementation, selection criteria) has been moved to [Krystallizer §Storage Implementation](krystallizer-en.md#storage-implementation).

## Cross-References

- **[Property Graph Queries](property-graph-queries-en.md)**: the definition and query implementation of the graph in a relational database—with SQL/PGQ as the standard, one graph with many relations, criteria for multiple graphs, crossing graphs via underlying tables, strictness by edge, and dynamization of relation types.
- **[KV Storage Engine](kv-storage-engine-en.md)**: the underlying carrier of this design's KV route—property graph encoding patterns, read-modify-write elimination (delta append), secondary index update strategy, WriteBatch transactions, vector hibernation/loading lifecycle.
- **[Memory Architecture](agent-memory-en.md)**: analysis of existing designs (agentmemory vs cognee, the four-quadrant panorama, an item-by-item comparison with this document).
- **[Stateless Agent Architecture](stateless-agent-architecture-en.md)**: the general outline of the component architecture—the turn model, the compression parallel bypass, Surface architecture semantics, and the place of skill emergence in the architecture.
- **[Agent Compound Interest](agent-compound-interest-en.md)**: the concrete implementation path of "memory accumulation" in the property graph within the compound-interest mechanism.
- **Knowledge atomic fact extraction skill** (Hermes skill `knowledge-triplets`, local `~/.hermes/skills/`): the extraction format definition, the write-layer specification of this document.