# Wasm Unified Runtime: From Compile Target to General-Purpose Backend

> **Languages:** [English](wasm-unified-runtime-en.md) (primary) · [中文](wasm-unified-runtime.md)

**Core thesis**: Wasm has long been regarded as a "third-party plugin sandbox" — Rust/C++ compiled artifacts running in an isolated environment. But Wasm is evolving from a compile target into a general-purpose language backend. This process comes in two steps: non-GC languages (Rust/Go) can already compile to Wasm and run; once wasm-gc lands, languages that need GC (Python/Kotlin/Swift, etc.) can also run smoothly, and only then does a unified runtime truly hold.

## Dual Identity: Compile Target vs. Runtime

Wasm currently has two uses, often conflated:

1. **Compile target**: Rust/C/C++/Go compile to `.wasm`, with Wasm serving as a portable binary format. The host language is unchanged; Wasm is merely the final artifact.
2. **Runtime**: Wasm as a general-purpose execution environment, where multiple languages compile to Wasm and then run inside the same runtime. Wasm is no longer the artifact of some particular language, but the common backend of all languages.

Currently Wasm's role is dominated by (1). The maturity of (2) comes in two steps: non-GC languages are usable now (the Rust toolchain is the most mature); languages that need GC await wasm-gc landing.

## Two Language Tiers

Wasm's support for languages is not a single point in time, but two tiers:

### First Tier: Usable Now (No wasm-gc Required)

- **Rust**: The most complete language for writing Wasm, bar none. Mature toolchain, high-quality artifacts, stable `wasm32-wasi` target. The barrier is the learning curve and compilation cycle, which make it unsuitable for rapid iteration.
- **C/C++**: Theoretically viable, but the language itself (manual memory management, ABI complexity, undefined behavior) holds it back, and the engineering experience falls far short of Rust.
- **Go**: Already compiles to Wasm (`GOOS=wasip1`), but its own GC runtime is packed into the module, so the size is on the large side. Usable, but inelegant — Go's GC runs in Wasm linear memory and does not use the runtime's native GC.
- **Kotlin/Native**: The AOT compilation path can emit Wasm, bypassing the JVM and not requiring wasm-gc. But the ecosystem is limited.

The first-tier languages can already cover most "embedded script language" scenarios today — Rust → Wasm for async, Go → Wasm for tool invocation, both already run. Rune's async scenario is already covered by Rust → Wasm now, without waiting for wasm-gc.

### Second Tier: Requires wasm-gc

- **Python**: A dynamic language; even with wasm-gc it still needs an interpreter (except that the interpreter can use wasm-gc instead of carrying its own GC). The slowest transition.
- **Kotlin/JVM**: Needs GC; once wasm-gc lands it can compile directly, with no AOT and no bundling of the JVM.
- **Swift**: ARC plus partial GC semantics; after wasm-gc it can run detached from the Apple platform.
- **Java**: GraalVM already has a Wasm path, and wasm-gc makes it more natural. But the language is verbose, and the JAR deployment model does not match Wasm modules.
- **C#**: .NET already has Blazor (Wasm); after wasm-gc efficiency improves (no longer bundling the .NET runtime's GC).

The second tier is the true beneficiary of wasm-gc — when running on Wasm they no longer need to carry a complete runtime, and the performance penalty is eliminated.

## wasm-gc: The Turning Point

wasm-gc (Garbage Collection proposal) gives the Wasm runtime native GC support, rather than relying on linear memory to manage it itself.

- **Before it lands**: First-tier languages (Rust/Go/C++) can already compile to Wasm and run. Languages that need GC can only carry a complete runtime themselves (e.g. Pyodide bundling CPython) or go the AOT route and lose eval capability — running a complete interpreter inside Wasm, with the performance penalty unresolved.
- **After it lands**: Languages that need GC can compile to Wasm and share the GC provided by the runtime, without carrying their own runtime. Wasm becomes truly usable for second-tier languages.

The significance of wasm-gc is not "letting more languages compile to Wasm" — the first tier achieved that long ago. The significance is **letting languages that need GC run on Wasm without carrying their own runtime**, thereby eliminating the fragmented landscape of "one embedded interpreter per language".

## WASI: Standardizing the System Interface

WASI (WebAssembly System Interface) is the other piece of the puzzle for Wasm as a runtime. Without WASI, Wasm modules cannot access system resources such as files, network, and clock, and can only rely on Host bridging — each Host implements its own API, so portability is fake.

The goal of WASI standardization: a Wasm module compiles once and runs in any WASI-compliant runtime, without needing to be aware of the underlying operating system. This aligns with the direction of a "unified backend" — a language does not care whether it runs on Linux or Windows, only about the WASI interface.

## Projecting Language Popularity After wasm-gc

Once wasm-gc lands, second-tier languages surge into the Wasm ecosystem. Which will become popular? Criteria: wasm-gc investment, language expressiveness, ecosystem migration cost.

| Language | wasm-gc investment | Language expressiveness | Ecosystem migration cost | Popularity projection |
|------|-------------|-------------|-------------|-----------|
| **Kotlin** | Earliest (Kotlin/Wasm aims directly at wasm-gc) | Coroutines, concise syntax, type safety | Clear path for porting the JVM ecosystem | High — JetBrains' first-mover advantage |
| **C#** | Accumulated via .NET NativeAOT + Blazor | Mature type system, async/await | Low enterprise migration cost | High — driven by Microsoft |
| **Swift** | Apple participates in Wasm GC discussions | Value semantics, ARC, async/await | Thin ecosystem once detached from the Apple platform | Medium — strong language but limited ecosystem |
| **Python** | Not applicable | Too dynamic; compiling to Wasm is extremely hard or even infeasible | The largest ecosystem | Does not take the Wasm path — the interaction model differs; in-process embedding via PyO3 is the correct form |
| **Java** | GraalVM path | Verbose; the JAR deployment model does not match Wasm modules | Heavy historical baggage | Low — neither the language nor the deployment model fits |
| **Wasm-native languages** | MoonBit and others, starting from scratch | Unconstrained by historical baggage | Ecosystem built from zero | Undetermined — depends on an explosion of language design after wasm-gc |

**Core judgment**: Kotlin and C# are the most likely to become the main Wasm languages after wasm-gc — both have big-company investment, modern syntax, and a clear migration path. Python does not take the Wasm path — it is too dynamic, compiling to Wasm is extremely hard or even infeasible, but it does not need to: PyO3 in-process embedding has low overhead, and the interaction model (immediate execution, no compilation required) is itself Python's value, not a degradation of Wasm. Java's language design and deployment model both mismatch the Wasm module paradigm.

Go is a special case: it is already in the first tier (carrying its own GC packed into Wasm), and wasm-gc merely makes it more efficient (no need to carry its own GC). But Go's advantage of simplicity is diluted in the Wasm ecosystem by Kotlin's expressiveness and C#'s maturity — the latter two are better suited to the wasm-gc native path.

## Wasm-Native Languages

Besides mainstream languages compiling to Wasm, there are languages designed specifically for Wasm:

- **MoonBit**: A systems language oriented toward Wasm, with concise syntax and fast compilation. The initial design was sharp, but it is evolving in a "want everything" direction (WASI compatibility in the standard Wasm environment, GC support issues, and stacked language features diluting the sharpness of the design). See [Programming Language Selection](language-selection-en.md).
- **More may appear later**: Once wasm-gc lands, the barrier to designing Wasm-native languages drops — no need to build your own GC and runtime, and language designers can focus on syntax and semantics.

The value of Wasm-native languages lies in this: unconstrained by historical baggage, they are optimized for the Wasm execution model from the start. But it is still early, and MoonBit is one of the few cases with actual output.

## Impact on the Embedded Language Landscape

### Rune: Async Is Already Covered; Value Narrows to Rapid Iteration

Rune (a Rust-like dynamic language) has as its core selling point "native async/await + in-process embedding". But Rust → Wasm is already completely fine today, and Go → Wasm is also available — Rune's async scenario does not need to wait for wasm-gc, it is covered now.

The only reason Rune still has value: dynamic language, immediate execution, no compilation wait. Rust's barrier is the learning curve and compilation cycle, which make it unsuitable for rapid iteration. Once wasm-gc lands, languages like Kotlin/C# can also run smoothly on Wasm, further compressing Rune's space — Kotlin's coroutines cover async, its syntax is easier to pick up than Rust, and compilation is also faster than Rust.

In the Aura architecture, Rune is currently retained. As the Rust → Wasm ecosystem matures (already usable now) and Kotlin/C# surge in after wasm-gc, Rune is downgraded to a fallback. Rapid iteration scenarios are covered by Python (PyO3) or Steel.

### Steel (Lisp): Lasting Value

Steel's core value is not in runtime performance, but in the deterministic boundaries of S-expressions — parentheses physically lock down scope, with no whitespace ambiguity. This value is unrelated to Wasm: even if all languages could compile to Wasm, policy engines and DSLs would still need a language where "syntax is the AST". Lisp's homoiconicity is something Wasm cannot replace.

### PyO3 (Python): No Need to Migrate to Wasm

Python's core value is ecosystem monopoly (HuggingFace/PyTorch/Numpy) and the interaction model — simple and fast, no compilation, immediate execution. There will always be scenarios that need such a scripting language, and Python is the most suitable choice.

Python compiling to Wasm is not a matter of "slow transition", but of a **fundamentally different interaction model**. Python is too dynamic (modifying types at runtime, monkey-patching, eval/exec), and static compilation to Wasm is extremely hard or even impossible. But this is not a defect — Python does not need to compile to Wasm, because PyO3's in-process call method has inherently low overhead: zero-copy pointers passed straight through, no serialization overhead, no module boundary. The Wasm runtime's sandbox isolation is an extra wall for Python, and instead increases call cost.

Therefore PyO3 is unaffected by wasm-gc and does not need to transition — in-process embedding of CPython is the correct form for Python in the Aura architecture, not a degraded substitute for Wasm.

### The Vision of a Unified Runtime

Once wasm-gc + WASI are both mature, the ideal state:

- **One runtime** (Wasm) replacing multiple embedded interpreters (Rune VM, Steel VM) — but CPython is not in this category; Python's interaction model and Wasm's compilation model are complementary rather than substitutes
- **Multiple languages** compiling to the same runtime, sharing GC, memory safety, and sandbox isolation
- **The Host only needs to interface with Wasm**, without integrating a different runtime for each language

This is not the elimination of language diversity, but the elimination of runtime fragmentation. Differences between languages return to the level of syntax and semantics, while the runtime level is unified.

## Timeline Assessment

| Stage | wasm-gc | WASI | Wasm's role in embedded languages |
|------|---------|------|-------------------------|
| **Current** | Proposal advancing, partially implemented | Basic interfaces stable, async/network immature | First tier (Rust/Go) already usable; Rune's async scenario already covered, rapid iteration still needs a dynamic language |
| **Near term** (1-2 years) | Supported by mainstream runtimes | WASI async I/O matures | Rust → Wasm ecosystem matures, Go → Wasm usable; second-tier languages still need to carry their own runtime |
| **Medium term** (wasm-gc lands) | Kotlin/C#/Swift can compile natively | System interfaces complete | Second-tier languages surge in, Rune's space is further compressed; Python is unaffected (does not take the Wasm path) |
| **Long term** | Wasm-native languages mature | Ecosystem stable | The unified runtime covers compiled languages; Python/Steel are retained as interaction-model languages |

The timeline is uncertain — how fast wasm-gc lands depends on the implementation progress of each runtime (Wasmtime/Wasmer/browsers), and WASI's async interfaces are still evolving. But the direction is clear: Wasm's evolution from "compile target" to "general-purpose backend" is a grand unification trend.

---

## Cross-References

- **[Embedded Script Language Selection](embedded-script-languages-en.md)**: Wasm currently serves as one of four embedded options (plugin sandbox); this document analyzes its path toward evolving into a unified runtime.
- **[Modern Language Design](modern-language-design-en.md)**: Wasm as a new backend is yet another example of complexity conservation — backend complexity sinks from each language down to a unified runtime.
- **[Aura Architecture](aura-architecture-en.md)**: Aura §6's multi-language runtime. Rust → Wasm already covers Rune's async scenario now; after wasm-gc, Kotlin/C# surge in and further compress Rune's space. Steel and PyO3 are retained (each has irreplaceable value).
- **[Programming Language Selection](language-selection-en.md)**: MoonBit as a case study of a Wasm-native language and the selection judgment.
