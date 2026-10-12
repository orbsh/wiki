# Zig and Anti-Intellectualism

> **Languages:** [English](go-zig-anti-intellectualism-en.md) (primary) · [中文](go-zig-anti-intellectualism.md)

## 1. The Necessity and Rationality of Rust's Complexity

### The Impossibility Trilemma and the Unique Mathematical Solution

Systems programming faces an impossibility trilemma: **absolute performance** (no GC / no runtime), **memory safety** (guarding against UAF / data races), and **developer cognitive burden** (language learnability). The three cannot all be attained at zero cost: C sacrifices safety, Java/Go sacrifice performance; Rust compromises on neither safety nor performance, but the learning curve is the price.

To guarantee safety without sacrificing performance, one must track memory ownership and lifetimes at compile time through strict mathematical constraints. At the language level, such constraints necessarily manifest as a complex, advanced type system (Rust's Traits + borrow checker).

**Corollary**: The complexity of Rust's type system is not over-design; it is the only mathematical solution to the "impossibility trilemma" above.

### Compile-Time Complexity vs. Run-Time Catastrophe

Macros and advanced generics do bring the complexity of AST manipulation and compile-time inference, but their essence is to move the state explosion and logical holes that could appear at run time forward into compile time.

**Conclusion**: Rust's complexity is a necessary price, not a redundant burden. It trades the "high barrier" of compile time for "zero overhead and high reliability" at run time.

**Necessary complexity ≠ cognitive burden.** Rust's type system is indeed heavier at the point of declaration than Go's `interface{}` — but that is a one-time investment. The key difference lies in the **location** of the complexity: declarative constraints are written at the definition site, written once; imperative validation is scattered across every call site, repeated in every project. The compiler consolidates the scattered validation into a single declaration, and the developer's daily cognitive load actually goes down. This is not "trading complexity for simplicity"; it is "moving complexity from the high-frequency path to the low-frequency path."

---

## 2. The Essence of a Type System Is Simplification

### Dispelling the "Black Magic" Myth

The starting point of every argument: does a type system (such as Rust's Traits) actually add complexity, or simplify it?

The answer is the latter. A type system is essentially compile-time logic programming, grounded in the **Curry-Howard correspondence**. Its core purpose is to simplify static semantics — to converge the hundreds or thousands of possible run-time state combinations into a handful of provable logical paths through type constraints.

Many people complain that Rust is "black magic." Here we must distinguish three levels:

1. **Declarative macros** (`macro_rules!`): essentially templates, doing pattern matching and text substitution. This has nothing to do with black magic — it is as plain as an HTML template engine.
2. **Procedural macros**: essentially AST transformations. It is like writing for the Web — HTML describes structure, JS handles logic; a procedural macro is that "JS," performing programmatic transformations on the syntax tree produced at compile time. Taking Rust itself as the baseline, the relative difficulty between mastering Rust and further mastering procedural macros is no higher than that between a front-end developer mastering HTML and JS.
3. **The type system and trait inference**: this is the real "magic." It is essentially declarative logic programming (similar to Prolog). When you write `T: Clone + Send`, you are not telling the compiler "go execute these few steps of checking," but declaring a logical constraint, which the compiler's internal solver derives automatically.

People who feel it is "black magic" conflate the three. Declarative macros are templates, procedural macros are AST transformations — neither is mysterious. What truly intimidates is the logic-programming nature of the type system, and that is precisely Rust's most powerful simplification tool.

Mistakenly attributing "I don't understand it" to "it is unreasonable" is exactly the starting point of anti-intellectualism.

### Rust's Complexity vs. Zig/Go "Simplicity"

| Dimension | Rust | Zig | Go |
|------|------|-----|-----|
| Type constraints | Trait declarative constraints, compiler derives automatically | comptime hand-written if assertions to simulate | None (interface{} runtime assertions) |
| Error handling | Result<T, E> algebraic data types | Error codes + hand-written checks | if err != nil boilerplate |
| Code volume (same functionality) | Type declarations heavier, but logic code concise | comptime assertions bloat generic functions | Large amounts of boilerplate to fill |

---

## 3. Type Systems and the Illusion of "Simplicity" (Zig as the Example)

### Paradigm Mismatch: Doing Logic Programming with an Imperative Language

Zig's comptime is at bottom an undertaking aimed in precisely the wrong direction.

A type system rests on the Curry-Howard correspondence and is a declarative logic-programming language (similar to Prolog). When you write `T: Clone + Send`, you are not telling the compiler "go execute these few steps of checking," but declaring a logical constraint. Through its internal solver, the compiler automatically derives and matches it across the global code context.

Zig's comptime, by contrast, is imperative. It merely takes ordinary Zig code and executes it at compile time. Using imperative if/for to do work that ought to be done by logical deduction is like hand-writing a regex engine in C — theoretically possible thanks to Turing completeness, but a paradigm regression.

### Why Zig's comptime Is Extremely Complex

Lacking declarative logical deduction, Zig developers can only hand-write huge numbers of comptime if assertions to simulate type constraints. For example:

```zig
// Zig: imperatively simulating type constraints
fn foo(comptime T: type) void {
    if (!@hasField(T, "foo")) @compileError("missing field 'foo'");
    if (!@hasDecl(T, "bar")) @compileError("missing decl 'bar'");
    // every added constraint means one more if
}

// Rust: declarative constraints, the compiler derives automatically
fn foo<T: HasFoo + HasBar>(t: &T) { ... }
```

Every generic function is stuffed full of defensive instruction code. The compiler is no longer an "intelligent solver" but has degenerated into a "mindless executor." Developers are forced to take on the duty of the type-inference algorithm.

### Even If comptime Achieved "Simplicity," It Would Still Be Meaningless

Even if Zig optimized comptime syntax to the extreme, it would remain meaningless. First, the background: comptime is essentially the merging of several specialized compile-time mechanisms in Rust (trait bounds, const generics, proc macros, const fn, type reflection) into one general compile-time execution environment. This unification has architectural value — Rust's proc macros + build.rs + const fn + trait bounds are each independent, dispersing the learning cost; comptime covers all compile-time needs with a single syntax, and is indeed more concise. But the problem lies in the price of that unification:

1. **Loss of intent**: A declarative type system lets code express "what I need"; comptime can only express "what I did." To infer what type you are actually constraining, other people reading Zig code must run the comptime in their heads. Error reporting is the same — the compiler has no semantic context and can only point at the syntactic location of the `@compileError` line, or emit incomprehensible errors from a low-level syntax crash; it can never provide the "what you're missing" diagnostic a real type system does.
2. **One-directional inference**: Inference in logic programming (Prolog, Datalog) is global and not direction-bound — given a conclusion you can infer back to the premises, given premises you can derive the conclusion, and the constraint solver automatically finds a solution satisfying all constraints. Imperative programming's data flow is one-directional, pushing only from input to output. comptime, as an extension of the imperative, is likewise locked by that directionality and cannot do reverse inference or global constraint satisfaction; it can only pile up a forward chain of if checks.
3. **Missing composability of code generation**: Rust's derive macros can stack multiple generators in one line (`#[derive(Debug, Clone, Serialize)]`), each compiled independently, tested independently, published independently. Zig's comptime has no macro system and no ecosystem of independent compile-time libraries — every comptime block is a logical island, uncomposable and unreusable.
4. **Structural limits of reflection**: `@typeInfo` only exposes structural information (fields, method signatures) and cannot query semantic constraints. Rust's trait bounds encode semantics at the type level and the compiler derives globally; Zig can only inspect structural features one by one and cannot make global constraint judgments.

The cost of unification is clear: type constraints → loss of intent, type inference → one-directionality, code generation → non-composability, reflection → structural blind spot. The root cause is that a type system belongs to the logic-programming paradigm — constraint satisfaction, bidirectional inference, global reduction — capabilities beyond the expressive range of an imperative language. However concisely comptime is written, it is imperative at bottom; it can only simulate the appearance of logic programming, never acquire its intrinsic power.

**Conclusion**: The "simplicity" Zig advertises is an illusion. It has not eliminated complexity; it has downgraded work that Rust's compiler and specialized mechanisms do automatically into manual labor for the user. It evades the complexity of language design while leaving the engineering complexity to the user.

### Allocator Philosophy: Core Infrastructure with an Extremely Low Ceiling

Zig's other core selling point is "explicitly passing an Allocator." Does this design philosophy hold up?

**The allocator field has an extremely low ceiling and is highly converged.** In real high-performance systems development, only a handful of allocators matter: jemalloc, mimalloc, tcmalloc, or a slab tailored to a specific structure. This field is extremely latency-sensitive; any fancy algorithm gets eaten by branch mispredictions and cache misses. Developers usually just take an off-the-shelf allocator and fine-tune its parameters, or write a minimal Bump Allocator (arena). Expecting language features to spawn a blooming ecosystem of allocators is unrealistic.

**Rust handles arena allocation just as elegantly, with compile-time safety as a backstop.** Zig often boasts that explicitly passing an Allocator is more flexible than Rust's default global allocator. But Rust now has `allocator_api`, and with the lifetime system, arena allocation is extremely natural — every object allocated by the `bumpalo` library has its lifetime bound to the arena reference, and when the arena is destroyed the compiler guarantees at compile time that there are no dangling pointers. In Zig the programmer must guarantee this themselves, and Zig can hardly even detect dangling pointers. For specialized-purpose allocators, Rust is not only flexible, it has a compile-time safety backstop.

**As a core language facility, it is over-design.** Zig forces almost all standard-library containers to take an Allocator parameter explicitly, meaning every function that manipulates a dynamic array must thread the Allocator down layer by layer. C itself does not do this — it calls `malloc`/`free` directly, or accepts an already-allocated pointer, and never threads an allocator through as a parameter. And the scenarios that truly "squeeze memory" (kernels, embedded) do not go through the heap at all — the strategy is stack allocation and static buffers, and many bare-metal environments have no heap whatsoever. When bulk allocation is needed you use an arena, but that is a single allocator at the subsystem level, not something to thread through every function. Zig's approach neither inherits C's pragmatism nor matches the real needs of embedded work; it is boilerplate pollution manufactured purely for the slogan "explicit over implicit."

### The Conservation Law of Complexity

The need to verify static semantics does not vanish into thin air because a language lacks advanced features. Rust uses Traits to derive and intercept errors automatically at compile time; this is the "co-optation" of complexity.

The core logic of Zig and Go is not that they technically solved the hard problems of systems programming, but that they compromised with the user's cognitive laziness. They convert the "learning cost" arising from a lack of logic-programming understanding into endless "boilerplate" in business code.

### The Cultural Root of Zig's Anti-Intellectualism: Why comptime Rejects Algebra

The preceding sections analyzed the technical defects of Zig's comptime — simulating type constraints with imperative assertions. But the deeper question is: **why does Zig's designer believe this is right?**

This has a different root from Go's anti-intellectualism. Go's simplicity serves management needs (replaceable engineers); Zig's simplicity comes from the **cultural bias of systems programmers**.

#### The Belief That "Abstraction = Hiding = Bad"

Zig's core philosophy is "no hidden control flow, no hidden allocations." Andrew Kelley (Zig's author) treats every "opaque" abstraction as an enemy. The sources of this belief:

**The legacy of the C tradition**: Zig's spiritual ancestor is C, not Haskell/ML. C's philosophy is "the code you write is the code the machine executes" — a single level of abstraction, no implicit behavior. This philosophy was reasonable in C's era (hardware resources were limited, the runtime cost of abstraction was high), but Zig **dogmatized** it: not "some of C's restrictions made sense at the time," but "all hidden abstraction is bad."

**An overreaction to C++ templates**: C++ template metaprogramming really was a disaster (hundreds of lines of compile errors, SFINAE black magic, exploding compile times). Zig's comptime is a direct reaction to that disaster — "we don't want magic, just ordinary code executed at compile time." But this is a **straw-man fallacy**: the problem with C++ templates is not "abstraction itself" but **bad abstraction design** (SFINAE, two-phase name lookup, specialization rules). Rust's trait system is abstraction too, yet it does not have C++ templates' problems — because it is better designed, not because it "hides nothing." Zig's reaction is equivalent to: seeing a shoddily built tower collapse and concluding not "we should design buildings better" but "we should stop building towers and live in bungalows."

#### The Cognitive Bias of Systems Programmers

Systems programmers have a deep-seated intuition: "I must know what every line of code does on the hardware." This intuition is correct when writing kernels, drivers, and embedded code, but it has been wrongly generalized to every programming scenario:

- "Generics are black magic" → in fact generics are monomorphized at compile time, with zero runtime overhead
- "trait bounds are hidden complexity" → in fact trait bounds are the compiler's verification logic, not runtime overhead
- "comptime is more transparent than a type system" → in fact comptime's if assertions are harder to read and more error-prone than trait bounds

**The systems programmer's preference for "transparency" is at bottom a distrust of "what I myself don't understand."** When they see `T: Clone + Send`, their reaction is not "this is a declarative constraint" but "the compiler is doing something behind my back that I don't know about." This distrust drives Zig's design.

#### Andrew Kelley's Mathematical Vision

Andrew Kelley is a self-taught programmer without a CS degree. His background is game-engine development (C/C++), not PLT. His mathematical intuition resembles Pike's — **computational mathematics (numerical computation, performance optimization) dominant, structural mathematics (set theory, logic, category theory) subordinate**.

But Zig and Go have one key difference:

| | Go | Zig |
|--|-----|-----|
| **Target users** | Ordinary engineers (management's perspective) | Systems programmers (experts' perspective) |
| **Motive for anti-intellectualism** | "Engineers are too dumb, they need a simple language" | "Abstraction is too black, we need a transparent language" |
| **Attitude toward mathematics** | Irrelevant (not needed) | Hostile (it hides the truth) |
| **Designer's self-image** | Engineer (pragmatic) | Systems programmer (low-level control) |

Go says: "Mathematics doesn't matter."
Zig says: "Mathematics is hiding the truth."

Both reject algebraic abstraction, but for different reasons. Go is **economically driven** (management needs); Zig is **culturally driven** (the systems programmer's bias).

#### Zig's comptime: Rejecting Algebra in the Name of "Transparency"

Zig's comptime is the archetype of "using arithmetic where algebra is required":

```rust
// Rust: algebra — declare constraints, the compiler verifies
fn process<T: Clone + Serialize>(item: T) { ... }
// the developer says "what I need"; the compiler does the checking
```

```zig
// Zig: arithmetic — manually check every constraint
fn process(comptime T: type) void {
    if (!@hasDecl(T, "clone")) @compileError("...");
    if (!@hasDecl(T, "serialize")) @compileError("...");
}
// the developer says "what I checked"; the compiler merely executes
```

Zig considers this more "transparent" — you can see every if and know what the compiler is checking. But that is exactly the problem: **you turn the compiler's duty (verifying type constraints automatically) into the developer's duty (writing if assertions by hand).** It is like rejecting a calculator because "I must do the arithmetic with my own hands to understand it." Yes, you understand every step, but you spend your time on repetitive labor instead of creative work.

---

## 4. The Anti-Pattern of Enforced Simplification (Go as the Example)

Go adopted a philosophy similar to Zig's: it long refused generics, erased sum types, and downgraded error handling into a flood of `if err != nil`.

Complex business logic and state machines do not disappear because a language lacks advanced abstractions. A lack of type expressiveness forces the logic to be handled implicitly at run time through loose `interface{}` values and strings.

**Conclusion**: Go tries to paper over systems complexity by castrating expressive power, and the result is the opposite of what was intended. Its "simplicity" stops at the syntactic level while creating a greater disaster at the semantic and engineering-maintenance levels.

---

## 5. Anti-Intellectualism in Concrete Practice

### Go's Six Manifestations of Anti-Intellectualism

**1. The readability strategy backfires**

Go's design goal was team readability — letting teams of uneven skill all read the code. The goal itself is reasonable, but the means (removing advanced abstractions, mandating boilerplate) produce the opposite effect: a complex state machine with 20 fields can, in Rust, use algebraic data types and macros to lock down the legality of state transitions completely at compile time (5 lines of code). In Go you must split it into 200 lines of switch-case and if conditions. Not only is 200 lines of hand-written validation harder to read than 5 lines of declaration, it is also easier to miss something — the price of the readability strategy is a drop in correctness, and in the end maintainability is sacrificed along with it.

**2. if err != nil: the systematic erasure of state-space modeling**

"Error" and "normal value" are two branches of the same state space and ought to be bound together at the type level through algebraic data types (such as Rust's `Result<T, E>`).

Algebraic data types have two levels of capability: **whether they exist at all**, and **whether they can be user-defined**. Go lacks both — it provides no built-in sum types (`Result`, `Option`) and does not let users define their own algebraic data types either. By comparison, even languages without language-level sum types (such as C) can at least simulate them manually with tagged unions. Go blocks even that space — all you have is multiple return values + if checks, with no type-level expressiveness whatsoever.

Go considers type-system inference too brain-burning, and forcibly strips errors out into ordinary return values. This not only produces a flood of boilerplate; more fatally, it destroys the compiler's ability to enforce error handling — you can easily swallow an error with `_, _ = doSomething()`.

**3. Refusing generics and higher-kinded types: the pathological psychology of guarding against "cleverness"**

Go refused generics for a full decade, and even when pressure finally forced them in, it deliberately wrote them extremely restricted (no HKT, no specialization). Rob Pike and other designers have said outright that Go is designed for "those junior employees at Google who aren't good at programming." This premise is itself anti-intellectual: instead of trying to raise engineers' ability through tools, it presumes engineers are stupid and therefore all tools requiring an intellectual threshold must be confiscated.

**4. Manufacturing a "false sense of fulfillment" through manual labor**

Go makes you repeatedly write getters/setters, repeatedly write err checks, repeatedly write interface implementations. This flood of boilerplate gives developers a false sense of fulfillment — "I wrote a lot of code, I'm very pragmatic." It robs programmers of the dignity of being a "logical architect" and pins them to the assembly line as typists.

**5. Time formatting: using arithmetic where algebra is required, i.e. anti-mathematization**

Go's time formatting uses `2006-01-02 15:04:05` as a reference-time template, replacing the industry-consensus strftime-style format specifiers (`%Y-%m-%d %H:%M:%S`). Go's justification: creating yet another syntax for formatting time is not worth it.

The absurdity of this decision is that the direction is reversed:

| Dimension | strftime (`%Y-%m-%d`) | Go (`2006-01-02`) |
|------|------------------------|-------------------|
| **Paradigm** | Algebra — declare "what I want" | Arithmetic — use a concrete value as a template and let the system infer the positions backward |
| **Cognitive load** | Remember what the symbols mean (Y=year, m=month) | Remember what position each digit of the reference time corresponds to |
| **Ambiguity** | Low — symbols map one-to-one onto semantics | High — is `01` month or day? Is `15` hour or minute? |
| **Composability** | Strong — symbols can be concatenated freely | Weak — positionally bound; changing one field can break the whole template |
| **Error diagnosis** | Precise — a mistake in `%q` does not affect `%Y` | Vague — write `2007` instead of `2006` and the format silently goes wrong |

**In essence: Go rejected algebra (declarative symbols) and chose arithmetic (concrete-value inference).**

strftime's `%Y-%m-%d` is algebra — symbols declare intent, and the compiler/interpreter does the mapping to concrete values. Go's `2006-01-02` is arithmetic — you must understand the reference time's "positional encoding" and perform in your head an inverse operation from concrete values to semantics. This is using arithmetic where algebra is required; the direction is reversed.

A deeper problem: **the format string is an already-consensus solution.** Python, C, Java, Ruby, and PHP all use `%Y-%m-%d` or a similar symbol system. What Go rejected is not "a new syntax" but "an existing consensus." Go believed it was not worth inventing a syntax for formatting time, and therefore used the Go language itself as the "syntax" — turning a simple symbol-lookup problem into a positional-encoding problem that requires understanding a time reference frame.

This shares the same root as Go's other anti-intellectual designs: **rejecting abstraction (symbolization) in exchange for "simplicity" (concrete values), which ends up being more complex.** Just as rejecting generics in exchange for "simplicity" produces a flood of boilerplate; rejecting the Result type in exchange for "simplicity" produces `if err != nil` spaghetti. Go's time formatting is one more instance of the same pattern: it appears not to require learning a new syntax, but in fact requires learning a harder mental model.

**6. gofmt's braces: elevating taste into a mandate of the language**

"The brace must follow the end of the line, it cannot be on a new line" — if this were merely a formatter setting, there would be no problem at all; it is common practice and a legitimate duty of a consistency tool. The controversy is not about the formatter.

The problem is that Go's handling is not at the formatting layer but welded into the language semantics: Go's lexer inserts semicolons automatically based on newline position, and a brace moved to a new line is a syntax error. A style preference that could have been negotiable was elevated into a language mandate, then draped in the cloak of "the great way is simplicity."

What is exposed here is the author's befuddlement, not a principle. Both are formatting rules: newline position was elevated into a syntax error, while indentation was left as a mere convention (a gofmt convention that does not constitute a syntax error). If "consistently simple" is the goal, why not likewise mandate that indentation must be "no tabs, exactly 4 spaces"? This selective enforcement is itself inconsistent — it does not come from a principle of consistency or simplicity, but from an old man who treats his own taste as language dogma and has lost the ability to examine his own preferences.

---

## 6. The Absence of Algebra: The Mathematical Root of Anti-Intellectual Design

The previous five sections criticized specific defects of Go — generics, error handling, time formatting, and so on. These are not isolated "missing features" but manifestations of a single root cause: **everywhere an algebraic abstraction is needed, Go degenerates into an arithmetic/procedural equivalent.**

### Sum Types = Logical OR; Go Rejected the "Or" in Its Type System

The core operations of algebraic data types:

- **Product Type** = logical AND = `struct { A, B }` → the value is A and B at the same time
- **Sum Type** = logical OR = `enum { A(T), B(E) }` → the value is A or B

Go has product types (struct) and **no sum types (enum)**. Go's multiple return value `(T, error)` is a product — two values existing simultaneously, not "either T or error."

```rust
// Rust: OR — the compiler forces you to handle both branches
enum Result<T, E> {
    Ok(T),    // branch A
    Err(E),   // branch B
}
// the compiler requires match to cover all branches, or it errors
```

```go
// Go: AND — both values exist at once, and the compiler does not force you to check error
func doSomething() (Result, error) { ... }
// the caller can throw the error away with result, _ := doSomething(); the compiler says nothing
```

**Go's multiple return values are not an "error-handling solution" but a "compromise that rejects logical OR."** It forces you to check line by line with if, because the type system has no "or" operation.

### The Complete Pattern: Go Degenerates to Arithmetic Everywhere Algebra Is Needed

| Algebraic structure | Meaning | What Go rejected | What Go used (arithmetic equivalent) |
|---------|------|-------------|----------------------|
| **Logical OR** | a type is A or B | Sum Types (`enum Result<T,E>`) | multiple return values `(T, error)` + if checks |
| **Logical AND constraint** | T must satisfy conditions P ∧ Q | Trait Bounds (`T: Clone + Send`) | `interface{}` + runtime type assertions |
| **Symbol mapping** | symbol → semantics (declarative) | strftime `%Y-%m-%d` | reference time `2006-01-02` (positional encoding) |
| **Pattern matching** | structured dispatch (compile-time exhaustion) | `match` / `switch` on sum types | `switch` on integers/strings (no exhaustiveness checking) |
| **Algebraic simplification** | identity transformations at the type level | HKT, type families | not supported |

Every row is the same judgment: **Go believes algebraic abstraction is not worth it and substitutes concrete values / procedural code.**

### The Designer's Mathematical Worldview

Rob Pike's original words:

> "Go was designed for engineers who are not computer scientists... The needs of software engineering are different from the needs of computer science."

Translation: **programming is engineering, not mathematics.** They treat PLT as "computer science" and engineering as "writing code," and believe the two have a clear boundary.

But a type system is not "a frill of computer science" — it is a **direct application of logic.** Sum types are the OR of propositional logic, product types are AND, pattern matching is the exhaustion of a truth table. In rejecting sum types, Go is essentially saying: "the logical OR operation is not needed in programming." That is not an engineering trade-off; it is a deficit of mathematical understanding.

Mapping this onto stages of mathematics education:

| Stage | Mathematical content | Corresponding programming concept | Go/Zig's position |
|------|---------|--------------|-------------|
| **Primary school** | The four operations | Concrete value manipulation, if/else | ✅ Go's comfort zone |
| **Middle school** | Basic algebra (equations, functions) | Function abstraction, basic types | ✅ Go just barely covers it |
| **High school** | Set theory, logic (AND/OR/NOT) | Sum Types, pattern matching, Trait Bounds | ❌ Go refuses |
| **University** | Category theory, type theory | HKT, GADT, algebraic effects | ❌ Neither Go nor Zig touches it |

Go's design stops at "middle-school algebra" — it has variables (function parameters) and equations (expression evaluation), but no set theory (sum types) and no logical operations (the exhaustiveness of pattern matching). This is not a "good enough is enough" trade-off; it is **welding the ceiling shut**.

### Zig's comptime Is the Same Disease

Zig's comptime also uses arithmetic wherever algebra is needed:

```rust
// Rust: algebraic constraints — declaring "what I need"
fn process<T: Clone + Serialize>(item: T) { ... }
// the compiler uses logical deduction to verify whether T satisfies Clone ∧ Serialize
```

```zig
// Zig: arithmetic assertions — describing "what I checked"
fn process(comptime T: type) void {
    if (!@hasDecl(T, "clone")) @compileError("missing clone");
    if (!@hasDecl(T, "serialize")) @compileError("missing serialize");
    // every added constraint means one more if
}
```

Rust's `T: Clone + Send` is **declarative logic programming** (similar to Prolog) — you declare constraints and the compiler derives automatically. Zig's `if (!@hasDecl(...))` is an **imperative assertion** — you hand-write the checking logic. Algebra is "declare the goal, the system solves it"; arithmetic is "execute the steps by hand." Zig chose the latter.

### One-Sentence Summary

Go's rejection of sum types is like its rejection of strftime and its rejection of generics — **using arithmetic where algebra is needed, procedure where logic is needed, imperative where declarative is needed.** The designer's mathematical horizon determines the ceiling of a language's expressive power. Go's "simplicity" is not making the complex simple; it is refusing to acknowledge that complexity exists.

---

## 7. A Panorama of the "Anti-Intellectual Language" Family

Anti-intellectualism is not a monolithic block; it has at least three schools:

- **Zig = the elitist school**: It has fresh ideas and creativity (comptime, compile-time execution, explicit control flow), but at bottom it is elitism — believing that only those who "understand the low level" deserve to write systems software, and rejecting for that reason every abstraction that lowers the barrier. It is not untalented; it uses its talent to "prove that ordinary developers don't need these things."
- **Go = the old-guard school**: A few outsiders in the PLT field who, on the strength of seniority and industry standing, PUA the entire community. Rob Pike, Ken Thompson, Robert Griesemer — all legendary figures, but not experts in type systems and language-design theory. Their strategy is not to persuade you with technology but to suppress doubt with "we built Unix/UTF-8 — who the hell do you think you are." Greasy, arrogant, refusing to evolve. **That reputation is itself the biggest platform: without the authors' prestige in the Unix world (they built Unix/UTF-8/Plan 9) as an endorsement, a language of this caliber would have a hard time making the top 100. Go was pushed high not by the value of the language itself but by the afterglow of its creators.**
- **PHP = the innocent school**: When Rasmus Lerdorf wrote CGI scripts back then he was cutting corners and did not think that far. Its anti-intellectualism is not active hatred of abstraction, nor suppression by seniority, but pure ignorance — not knowing what a type system is good for, not knowing what the consequences of weak typing would be, and just running naked all the way for thirty years. "Innocent" to the point of being pitiable, but none the less damaging for it.

### Comparison Table

| Language | What was castrated | Target users | Dimensionality-reduction means | Positioning |
|------|-----------|---------|---------|------|
| **Go** | Generics (a decade), sum types, macros, operator overloading | Junior engineers of uneven skill inside Google | if err != nil flattens state machines into spaghetti | The benchmark |
| **Zig** | Trait system, AST macros, hidden control flow | Old C programmers used to purely imperative thinking | comptime assertions hand-writing type constraints, Allocator threaded layer by layer | Isomorphic philosophy, with an extra layer of "low-level fanaticism" |
| **VB/VB.NET** | No new syntax extension, consumption only, no creation | Non-CS Windows desktop developers | On Error Resume Next silently swallowing errors, drag-and-drop controls as programming | Openly acknowledged by Microsoft; the most naked anti-intellectual manifesto |
| **PHP** | Long without a type system, weak typing with implicit conversion | Web outsourcing shops and individual webmasters shipping fast | "5"+10==15, global variable registration | Not deliberately anti-intellectual, but "a pragmatic embrace of chaos," isomorphic in effect |
| **Bash** | No type system at all, everything is a string | Ops people, glue-script writers | Unquoted $var blowing up via word splitting, empty variables crashing the syntax | A domain language that should never have been used this way |
| **Lua** | No type system, no object system | Game designers writing config, embedded extension | OOP hand-rolled with metatables | **Exception**: honest minimalism (see below) |

### Individual Commentary on Each Language

**Visual Basic: the family's spiritual ancestor**

Microsoft's language strategy states it in black and white: "Visual Basic will remain a simple, approachable language... when C# introduces new features, VB generally takes a consumption-only approach and avoids new syntax... it will not expand to new workloads." This is the official version of Go's "guard against cleverness" philosophy, and it is more thorough — it will not even let the language itself evolve, permitting users only to consume wheels others have built. VB's `On Error Resume Next` is more anti-intellectual than Go's `if err != nil`: the latter at least forces you to write a line of checking, while the former swallows the error outright and pretends nothing happened.

**PHP: anti-intellectualism achieved by accident**

PHP's weak typing, implicit conversions, and set pieces like `0 == "abc"` were not originally meant to limit users' intelligence, but were shortcuts Rasmus Lerdorf took when writing CGI scripts back then. Yet over thirty years these shortcuts have sedimented into a culture — PHPers generally regard a type system as "over-design," and only with the forced patching in PHP 7/8 did it moderate somewhat. Its difference from Go: Go actively hates abstraction, PHP passively gives up on abstraction, but the end results are highly consistent.

**Bash: the extreme form of anti-intellectualism**

It does not even have the concept of "types"; everything is a string. A space in a filename can make your `rm $file` delete the wrong file; an empty variable can crash `if [ $name = "admin" ]` syntactically. It downgrades all handling of structured data into awk/sed/pipe string concatenation, and downgrades error handling into after-the-fact patches like `set -euo pipefail`. Bash is not included in discussions of "anti-intellectual languages" only because everyone tacitly assumes it is not a general-purpose language — but in ops it has in effect played the role of a general-purpose language.

### Lua's Special Case: Honest Minimalism

Lua must be pulled out of this family. It too looks like it "has nothing," but its essence is completely different:

- Go's "no generics" **forbids** you to abstract
- Lua's "no OOP" **hands the right to abstract back to you** — through metatables' `__index`/`__add`/`__call` you can hand-roll any paradigm you want, including full prototype-based OOP

Lua's philosophy is "mechanism, not policy" — provide mechanisms, do not make decisions for you. This is precisely the opposite of anti-intellectualism: it assumes the user is smart, so it gives only the most orthogonal primitives and you build the rest yourself. Go assumes the user is stupid, so it does not even give primitives. On the surface both are "minimal," but at bottom they are opposites.

"Mechanism, not policy" is one of the most fundamental Unix ideas. Lua took orthogonal primitives from this idea; yet Go's authors have deep ties to Unix — Ken Thompson is a co-author of Unix, Rob Pike worked on Plan 9 — and they should have been the people who understood this idea best, yet they put it behind them and elevated style preferences into language doctrine. The same idea: Lua learned and applied it, while Go's ancestors went the opposite way. This is not a difference of position but a degeneration of cognition — an epic regression for Go in the field of programming languages.

---

## 8. A Shared Fate and Final Endgame

This family shares one terminal characteristic: their "simplicity" can survive only on the dividend of an ecological window.

- **VB** survived on the Windows desktop monopoly and was marginalized by C# as soon as the Web era arrived
- **PHP** survived on having no rivals in the early Web and was eroded once front-end engineering and typed backends appeared
- **Go** survives on the cloud-native window and began to be squeezed once Rust's async ecosystem matured
- **Zig** survives on "the C toolchain sucks" and will lose its footing once Rust eats into C

None of them lose because they are "too simple to beat complexity"; they lose because evading the cost of confronting complexity head-on gets settled in one lump sum once the ecosystem matures. Lua is the exception, because it never claimed to solve the complexity of general-purpose programming — it honestly stays in the embedded-scripting slot, and this restraint of "knowing who it is" is precisely why it has not been eliminated.

### Zig's Existential Crisis

Zig's only currently irreplaceable value in the industry has retreated to a good LLVM-based C/C++ cross-compilation toolchain (`zig cc`) and seamless C interoperability. A toolchain's value depends on what it processes — if Rust progresses smoothly and substantially replaces C code (as in the Linux kernel's evolution), the demand for C interoperability and cross-compilation will shrink sharply. The underlying logic of Zig as C's "life-support system" will be dismantled.

---

## Ultimate Conclusion

The complexity of systems programming is an objectively existing physical law. Rust chooses to confront it head-on, encapsulating complexity inside the compiler by introducing a complex but rigorous type system and macros; Zig and Go try to manufacture the illusion of "simplicity" by restricting the language's power, and ultimately hand complexity back to users in the form of uglier, more error-prone boilerplate. Historical patterns suggest that languages that evade complexity will settle their debts in one lump sum once the ecosystem matures — but the endgame has not yet arrived, and this pattern may still be falsified.

---

## Cross-References

- **[Modern Language Design](modern-language-design-en.md)**: the source of this article's counter-arguments — the positive case for type systems, abstraction facilities, and the conservation of complexity.
- **[Unified Data Layer](unified-data-layer-en.md)**: SurrealQL's "pan-Rust lineage" — sharing design genes with modern systems languages.