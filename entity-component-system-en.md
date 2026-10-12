# ECS: Entity-Component-System (with Bevy as the Example)

> **Languages:** [English](entity-component-system-en.md) (primary) · [中文](entity-component-system.md)
> 2026-08-12 Created

## Paradigm Difference: ECS Relative to OO Is Labels, Not a Tree

The dividing line between ECS and OO is not "whether there are objects," but **how data and logic are organized**. OO uses class definitions to encapsulate data into objects and hangs logic (methods) on the objects, with identity determined by the class; ECS splits data into flat components, separates logic (systems) from data, and determines identity by the combination of components. Structurally, this is a migration **from tree to labels** (the full argument is in [Modern Language Design](modern-language-design-en.md)):

- **OO (class inheritance) = tree**: the structural characteristics of the class hierarchy lie in the **edges**—connecting only across levels (parent→child), never within the same level; a class can sit on only one inheritance chain, converging level by level upward to a single root class (Java's Object). A class is a **name** (identifying the subject's "who it is"), single and unique, and identity comes from its named position in this tree.
- **ECS = labels**: an entity is a pure identifier (a single handle), a component is a set of labels—**one object, many labels**—each label describing one property/aspect, able to coexist (`Player`, `Health`, `Transform`) and to combine freely. The number of combinations is the **product** of the label kinds (one object with many labels, i.e., multidimensional).

**ECS is Rust's answer to OO.** Rust has no class (see [Modern Language Design](modern-language-design-en.md)); when class-free + label-based organization had to land in the domain where "a large number of heterogeneous entities continuously evolve and interact with one another," ECS grew out of it—more precisely, ECS is the answer of "class-free/label-based" to **simulation/game** scenarios. Conceptually it stands on the same level as OO, a general data+logic organizational paradigm (agent simulation, ecology, physics, epidemiology and the like can all use it); but in practice it is highly concentrated in games and high-performance real-time simulation. This is not accidental, but decided by the **benefit profile**—every advantage of ECS serves a specific condition:

- **Cache locality**: components of the same type are stored contiguously (SoA), and when systems walk in batches they traverse memory in sequential order. Only when the number of entities is large and systems scan batches of same-type components does cache friendliness translate into perceptible performance.
- **Stateless synchronization**: state lies in components and systems read it directly via `Query`, with no need for getters/setters, notifications, or value passing over an event bus. The moment data changes it becomes visible across the whole network, naturally consistent.
- **Changing identity at runtime**: adding or removing components on the same handle changes its type (alive→corpse), without swapping objects.
- **Systems can run in parallel**: the scheduler runs systems with no shared state in parallel, according to the read/write dependencies the systems declare.

These four benefits pay off only under **one and the same profile**: **a large number of entities** (volume gives cache and parallelism meaning), **performance-sensitive/real-time** (the frame budget forces you to optimize cache and parallelism), **entity types mutable at runtime** (changing identity has real use). This is exactly the game/simulation profile.

Leave this profile and the benefits disappear while the costs become visible: with few entities, cache locality does not matter; with stable types, changing identity at runtime is useless; when not real-time, there is no reason to pay scheduling overhead for parallelism. And the cost of ECS is constant—

- **Indirection**: an entity is an anonymous row key, and to know "what it is" you must look up its components; an OO class tells you directly.
- **Loss of encapsulation**: behavior is scattered across systems that act on component sets, the locality of "methods belong to the object" is gone, and the logic coalesced within an object becomes harder to maintain.
- **Query overhead**: every access goes through component storage, unlike OO's direct field access.

Business CRUD, low entity counts, unchanging types—OO or ordinary data structures are simpler, and ECS is not a universal replacement.

The essence of this distinction is **label = inverted index**: decoupling "entity" from "category"—the entity is located by a unique identifier, the category is expressed by a set of labels, and a query goes through the inverted index to compute an intersection. ECS's `component` and a trait/tag/label are the same kind of thing: both are "labels hung on an identifier"; the only difference is that ECS's component carries data in addition to being a pure label (`Health(3)`, `Transform`), being "label + data."

| Role in ECS | Position in the structure |
|:--|:--|
| entity | pure identifier (no content, just a handle) |
| component | label + data (`Player` is a pure label, `Health(3)` carries data as well) |

Systems filter entities by component (`Query`), which in essence is computing an intersection over the inverted index—one entity can simultaneously carry multiple sets of labels such as `Player`/`Health`/`Transform`, which is exactly "one object, many labels," not a tree. The way labels combine also corresponds to **parametric polymorphism** (implementing interfaces, like Rust's `T: A + B`), not OO's **inclusion polymorphism** (subtyping/Liskov substitution)—yet another paradigm-level divergence from OO.


## Scene-Tree Camp vs ECS Camp: Organizational Models of Mainstream Engines

The "tree vs labels" of the previous section is not an abstract opposition; it maps precisely onto the organizational-model choices of mainstream open-source engines. Four well-known engines happen to split into two groups (data verified against official sites/READMEs in 2026-08):

```
    Tree (inheritance/scene)             Labels (ECS)                
 Godot ── fyrox ───────────────────── │ ── Ambient ─── Bevy          
Composite scene tree Inheritance tree │  ECS + real-time DB  Pure ECS
```

### fyrox — Scene Graph Node Inheritance Tree (tree/inheritance camp)

- The core is a `Node` trait + a scene-graph inheritance hierarchy: `Node` is the root, deriving node types such as `RigidBody`, `Sprite`, `MeshNode`, `Light`, `Camera`, registered by `TypeId`, and restored by type when the scene is deserialized.
- Behavior relies on a `Script` trait (similar to a component, but hung on nodes, with `on_update` every frame), and lifetime is tied to the node tree.
- **Not ECS**: it chose the combined model of "inheritance tree + script," rather than a data-driven ECS.
- It ships with a scene editor, being a "complete engine with its own toolchain."
- Of the four, it is the one closest to the **OO tree**: node types are the hierarchy itself, and identity is jointly determined by "which node type on the inheritance tree + which scripts are attached."

### Godot — Composite Scene Tree (tree/composition camp)

- The core is a **Scene + Node tree**: a game object is not a single node but a subtree of nodes (`Sprite2D` + `CollisionShape2D` + `AudioStreamPlayer`… combined and attached together).
- Scripts (GDScript's `extends Node2D`) are **real class inheritance**, but node composition itself is of the **"one object, many labels"** kind (attaching various functional child nodes to an object)—even though it is carried by a tree, at the composition level it already carries a flavor of labels.
- Communication relies on **Signal**.
- **No ECS**—it explicitly does not use ECS, relying on "shallow inheritance + node composition + signals," which is "flatter" than fyrox's deep, resource-style inheritance.
- The core engine is C++, and scripting can be GDScript/C#/Rust (gdext).

### Ambient — ECS as a "Real-Time Database" (label camp, suspended)

- **Pure ECS architecture**: entity = handle, component = data label, exactly the "labels (one object, many labels)" scheme from the start of this chapter.
- **Key status: development is indefinitely suspended** (a warning sits at the top of the README; the team moved on to the Braindump project). The historical architecture still has reference value, but as a technology choice it is no longer meaningful.
- Distinctive point: it treats ECS as the **authoritative real-time database**—the server holds the full set of entity/component, clients synchronize automatically, and the network layer simply synchronizes ECS changes outward. The "state synchronization" problem of multiplayer games is absorbed by "synchronizing the entire ECS." This precisely echoes the later positioning that "the ECS mental model is like a database"—Ambient turned this metaphor into a literal architecture.
- Game logic is written in Rust/C/C#, compiled to WASM, and isolated outside the runtime. Rust + WASM + WebGPU, targeting web/desktop multiplayer games.

### Bevy — Pure ECS (label camp, the purest)

- `bevy_ecs` is a homegrown ECS; systems + components are the lowest-level mental model, with no built-in editor (the Bevy Editor is still under construction), code-driven + instant preview.
- The rest of the Bevy API in this article (`Query`/`Commands`/`Message`) is all this system.

### One-Line Comparison

| Engine | Language | Organizational model | Editor | Status |
|---|---|---|---|---|
| **Godot** | core C++, scripts in GDScript/C#/Rust | scene tree (composition) + signals | mature | active |
| **fyrox** | Rust | scene graph inheritance tree + scripts | yes | maintained |
| **Ambient** | Rust (+WASM logic) | ECS as real-time database | yes (Studio) | **suspended** |
| **Bevy** | Rust | pure ECS | under construction | most active |

**Architecture in a phrase**: Bevy is the purest form of the label camp; Ambient is the radical attempt at "label camp + network sync as database"; Godot is a flat composition tree; fyrox is the only one of the four that treats "the inheritance hierarchy as a first-class citizen," the tree camp (and also the closest to OO).

This distribution in turn confirms the previous section: **ECS is not the default answer for game engines**. Bevy/Amethyst in the same era turned ECS into a stereotype of Rust games, but Godot and fyrox prove that "scene tree + script" still holds on the path of a mature product—the benefit profile determines the stance: ECS is strong in high-performance batch simulation, while tree + script is strong in onboarding content authors and the directness of the editor.

**Other well-known Rust engines/frameworks** (divided by complete engine vs library): complete engines ship with scenes/toolchain—Amethyst/Legacy (a specs-lineage ECS framework, largely shelved, historically representative of the early effort to bring ECS into Rust games); 2D game libraries with no scene graph, no editor, leaning toward framework—macroquad, ggez, Tetra, nannou (creative coding); pure ECS libraries that solve only "data + system organization" and can be adopted on their own—`bevy_ecs`, `hecs`, `specs`, `legion`, `shipyard`. For an authoritative index see [Are We Game Yet](https://arewegameyet.rs/) (the Ambient README points there too).

## Definition: A Three-Layer Model (Database Mental Model)

The ECS mental model is not an object tree. There are two kinds of object tree, do not confuse them: one is OO's **class inheritance tree** (single-subject naming)—ECS is precisely its replacement; the other is the engine's **scene graph** (`Parent`/`Child`), an independent tree used for spatial addressing. The ECS data model is a **flat library of labels, like a database**:

- **Entity** = row key: joins multidimensional data into one addressable unit, and itself has no content
- **Component** = a queryable dimension (label/column): stores data or serves as an identity marker
- **System** = a query + an operation uniformly executed over the result set

In this way component and system are **naturally derived**: the world is a flat multidimensional table, component is the dimension you want to query, and system is the batch operation uniformly performed on the "query result set." **What an entity is is entirely determined by the combination of components on it**, and does not come from any class definition.

The same structure, viewed from two directions:

- **Top-down (engine/data)**: system is a query acting on a component result set; entity is a row key
- **Bottom-up (entity/game)**: entity is an existence, component is its property, system is the world applied to it (gravity, AI, damage, rendering)

(The "viewport projection" role of an entity: an entity with `Transform` + rendering components projects its multidimensional properties onto the 2D/3D viewport and becomes what you see. But this is only one manifestation of an entity—not all entities are visible, and purely logical entities such as timers, cameras, and game state have no viewport projection.)

## Component vs Entity: Data vs Composition

Entity and component are two halves:

- **A component is pure data, carrying no behavior**
- **An entity is one combination of components** (one handle + a set of components)

```rust
#[derive(Component)]
struct Health(u32);   // data

#[derive(Component)]
struct Player;        // empty marker, used to mark identity

// an entity = handle + component set
commands.spawn((Player, Health(3), Speed(3.0), Transform::from_xyz(0.0, 0.0, 0.0), Sprite::default()));
```

The essential difference from object orientation: in OOP the object type is fixed by the class definition, whereas in ECS an entity can **change identity at runtime by adding or removing components**—the same handle, by removing `Health` and adding a `Dead` marker, changes from "alive" to "corpse," without swapping objects.

## Why: The Benefits of Separating Data and Logic

**The separation of data and logic** is the structural premise of ECS's benefits, not "data-driven." "Data-driven" is too broad—OO, relative to imperative/procedural, is likewise organized around data (an object is data + methods), so it cannot separate ECS from OO. The real dividing line is in **the relationship between logic and data**:

- **OO**: logic (methods) is bound to data (objects), and data owns its own operations
- **ECS**: logic (systems) is stripped from data (components), systems act on data, and data is merely the state being operated on

This separation brings four structural benefits (not just nice-to-haves):

1. **Stateless synchronization**: state lies in components, and any system that `Query`s it reads the latest value—no getters/setters, no notifications, no value passing over an event bus. "State synchronization" becomes a non-problem here, and this is exactly the root cause
2. **Composition far exceeds inheritance**: to get a "flying, attacking, red enemy," just assemble a few groups of components on the entity, without building a pile of subclasses. The Cartesian product of component kinds is the kinds of entities, and the type explosion is dissolved
3. **Cache locality**: components of the same type are stored contiguously, and systems exploit memory contiguity when traversing in batches
4. **Explicit dependencies = parallelism**: read/write dependencies between systems are explicitly declared to the scheduler, and systems with no shared state can safely run in parallel

## Bevy: Mental Model and Main APIs (0.19)

**Mental model**: Bevy turns the whole game into an **App**—inside the App there is an ECS **World** (all entities/components/resources live in it), plus a **Schedule** that controls "when systems run." What you write is not objects, but **systems**: each system is a function that declares what it wants (which components `Query` reads, whether it needs `Commands`, whether it needs a global `Res`), and is then scheduled into some Schedule to run repeatedly. **Data lives in the World, logic lives in the System, and timing lives in the Schedule**, the three being separate.

The correspondence between core types and ECS roles:

| Bevy type | Role |
|:--|:--|
| `App` | top-level container: the assembly of World + Schedules + Plugins |
| `World` | one ECS world: the storage of all Entity/Component/Resource |
| `Entity` | handle (row key) |
| `Component` | the block of data hung on an entity (`#[derive(Component)]`) |
| `Resource` | global singleton data not hung on an entity (`#[derive(Resource)]`: `Time`, `AssetServer`, `Input`…) |
| `System` | a function whose parameters declare dependencies (`Query`/`Res`/`ResMut`/`Commands`/`MessageWriter`…) |
| `Query` | read/write entities by component type (filters `With`/`Without`/`Changed` can be added) |
| `Commands` | deferred entity operations (spawn/despawn/insert), taking effect uniformly at the end of the frame |
| `Message` | one-shot messages between systems (0.19 replaces the old `Event`) |
| `Schedule` | when systems run (`Startup`/`Update`/`FixedUpdate`/custom) |
| `Plugin` | a reusable module packaging systems+resources (`DefaultPlugins` or a custom `Plugin`) |
| `State` | the application state machine (menu/playing/paused), switching which systems run according to state |

Why call this a "mental model" rather than a memorization exercise: **you never write "object methods," you write systems**. To add movement to the player—write `fn move_player(q: Query<&mut Transform, With<Player>>)`, which acts on all entities carrying these two components. Add gravity, add damage—write a system for each. Each system declares only the components it wants, and the Bevy scheduler connects them to the right timing and runs them in parallel when there is no conflict. This is "treating the game as a database" landing in the engine.

Minimal skeleton:

```rust
use bevy::prelude::*;

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)                     // built-in systems such as rendering/input/time
        .init_resource::<Score>()                        // register a global resource
        .add_systems(Startup, setup)                     // run once at startup
        .add_systems(Update, (move_player, update_score)) // run every frame
        .run();                                           // start the App
}

#[derive(Resource)]
struct Score(u32);

#[derive(Component)]
struct Player;

fn setup(mut cmds: Commands) {
    cmds.spawn((Player, Transform::default())); // spawn an entity = a set of components
}

fn move_player(q: Query<&mut Transform, With<Player>>) {
    for mut t in &q { /* move every frame */ }
}

fn update_score(mut s: ResMut<Score>, input: Res<ButtonInput<KeyCode>>) {
    if input.just_pressed(KeyCode::Space) { s.0 += 1; }
}
```

## Bevy in Practice (0.19)

All of the following APIs were actually compiled and verified against Bevy 0.19.0.

### Lifecycle

Spawning, despawning, and changing type all go through `Commands` (buffered and deferred, to avoid crashing on modification during iteration):

```rust
let e = commands.spawn((Player, Health(3))).id();   // spawn and get the entity handle

commands.entity(e).despawn();               // despawn (deferred)
commands.entity(e).insert(Dead);            // change identity at runtime
commands.entity(e).remove::<Health>();
```

### State Sharing and Messages

- **Local state**: `Query` reads components directly, naturally consistent, no synchronization needed
- **One-shot cross-system messages**: Bevy 0.19 uses `Message` to replace the old `Event`

```rust
#[derive(Message)]
struct HitEvent { target: Entity, damage: u32 }

fn a(mut w: MessageWriter<HitEvent>, q: Query<Entity, With<Player>>) {
    let Ok(e) = q.single() else { return };     // e comes from the query
    w.write(HitEvent { target: e, damage: 1 });
}
fn b(mut r: MessageReader<HitEvent>) { for ev in r.read() { /* consume */ } }
```

### Feedback as Components

Make "flash white" and "floating text" into components, managed by the entity's lifecycle, stackable, and uniformly handled by systems:

```rust
#[derive(Component)]
struct FlashColor { timer: Timer }

// inserted on hit; a system reads it, adjusts brightness, and removes it when done
commands.entity(e).insert(FlashColor { timer: Timer::from_seconds(0.15, TimerMode::Once) });
```

## Cross-References

- [Modern Language Design](modern-language-design-en.md): ECS's `component` and `trait` both belong to the "labels (one object, many labels = inverted index)" structure; component composition corresponds to parametric polymorphism rather than OO's inclusion polymorphism
- [AI Creative Production Workflow](ai-creative-production-workflow-en.md): the positioning of Bevy as the interactive outlet in the fabricario pipeline