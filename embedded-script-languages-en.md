# Embedded Script Language Selection: Rune vs Steel Technical Deep Dive

> **Languages:** [English](embedded-script-languages-en.md) (primary) · [中文](embedded-script-languages.md)
> **Upper Framework**: This document is the language-level deepening of the "program (embedded)" extension paradigm—it discusses only the embedded-script lineage (general-purpose scripting / restricted DSL / compiled-artifact WASM), and does not touch the "address (RPC) vs. program (embedded)" methodology, which is covered in [Aura Architecture](aura-architecture-en.md) §0 "Extension Unit: Address vs. Program".


When developing a general-purpose business or AI-agent orchestration system in Rust, the core motivation for introducing an embedded dynamic scripting language is to achieve policy-layer decoupling, hot reloading, and flexible team extension. Rune and Steel represent two diametrically opposed technical aesthetics and design architectures.

## 1. Core Technical Comparison

### 1.1 Detailed Dimension-by-Dimension Technical Comparison Sheet

| Dimension | Rune (rune-rs/rune) | Steel (mattwparas/steel) | Wasm (Wasmtime/Wasmer) | PyO3 (Python embedding) |
|---------|---------------------|--------------------------|------------------------|-------------------|
| **Language family** | Rust-like syntax (C-style) | A modern Lisp / Scheme dialect | Compilation target (Rust/C/C++/Go, etc.) | Python 3 interpreter |
| **Underlying VM** | Stack-based VM | Bytecode VM | Cranelift JIT / hardware sandbox | CPython interpreter (GIL) |
| **Memory / garbage collection** | Reference counting (RC) + stack isolation | GC + immutable persistent data structures | Managed by the host language / Wasm linear memory | Python GC + Rust zero-copy |
| **Concurrency / async support** | First-class in the kernel (async/await) | Bridged support (Promise-driven) | No native async (requires host bridging) | GIL-limited, needs multithreading / multiprocessing |
| **Metaprogramming & macros** | Rust-like procedural macros | Lisp syntax-rules hygienic macros | None (fixed at compile time) | Python decorators / metaclasses |
| **Toolchain (LSP)** | Mature (official LSP) | Immature (basic highlighting) | Depends on the source-language toolchain | Mature Python ecosystem |
| **Core positioning** | Async scripting, web crawling | Policy engine, DSL, permission routing | Third-party plugin sandbox, compute-intensive (once wasm-gc matures it can become a unified runtime; see [WASM Unified Runtime](wasm-unified-runtime-en.md)) | AI/Data ecosystem reuse |
| **Startup latency** | Milliseconds | Milliseconds | Milliseconds (JIT compilation) | Tens of milliseconds (interpreter init) |
| **Memory footprint** | ~5-10MB | ~3-5MB | ~2-8MB (depends on the module) | ~20-50MB (Python runtime) |
| **Security isolation** | In-process, relies on host restrictions | In-process, no I/O by default | Hardware-level sandbox (cosmic-grade isolation) | In-process, GIL-protected |
| **Ecosystem advantage** | Rust-native async | Metaprogramming, DSL expressiveness | Cross-language compilation, near-native performance | The entire AI/ML/Data ecosystem |

### 1.2 Polyglot Embedded Harness Architecture: User Selection Guide

In the ultimate geek full-stack architecture, the framework provides four embedded runtime capabilities, but **which language to use is entirely up to the user (developer) based on business needs**; the framework does no intelligent dispatch.

| Scenario requirement | Recommended solution | Rationale |
|---------|---------|------|
| **High-risk policy engine** | Steel (Lisp) | 100% character-boundary determinism, naturally sandboxed isolation, no whitespace/indentation ambiguity |
| **Heavy async web crawling** | Rune | Native async/await, fits perfectly into Tokio Futures |
| **Third-party plugin hosting** | Wasm | Hardware-level sandbox isolation, supports Rust/C/C++/Go compilation artifacts. Once wasm-gc + WASI mature it can be extended into a unified runtime (see [WASM Unified Runtime](wasm-unified-runtime-en.md)) |
| **AI/ML model inference** | PyO3 | Zero-overhead reuse of the HuggingFace/PyTorch/LangChain ecosystem |
| **Custom DSL** | Steel | The Lisp macro system is a decisive advantage; syntax is the AST |
| **Compute-intensive tasks** | Wasm | JIT compilation approaches native performance, linear memory model. Once wasm-gc lands it can carry more languages (see [WASM Unified Runtime](wasm-unified-runtime-en.md)) |
| **Legacy Python code integration** | PyO3 | Call .py files directly, zero migration cost |
| **Permission routing / access control** | Steel | The topological geometry of S-expressions can never be ambiguous |

**Key clarification**: the framework only provides runtime capabilities; it does not decide which language is used. The user, according to the concrete business scenario (choose Steel if a DSL is needed, Rune if async is needed, Wasm if a plugin sandbox is needed, PyO3 if the AI ecosystem is needed), explicitly chooses and writes the corresponding code.

## 2. Developer Cognitive Conflict and Psychological Logic

### 2.1 The Doubt: The "Uncanny Valley" of Rust-like Scripting (Rune)

Although Rune gives Rust engineers a zero-learning-cost onboarding experience, for developers who are proficient in Rust it produces severe muscle-memory betrayal and psychological friction when they actually build general-purpose environments:

```
[Rune script coding] ──(Visually near-identical to Rust)──► [Brain auto-activates] ──► Instinctively agonizing over lifetimes and borrow checking ──► Mental burnout
```

- **Phantom Borrow Checker**: A Rust developer's brain constantly keeps lifetimes and the borrow-checking line mounted. When writing Rune, the visual presentation is almost identical to Rust (it supports match, ?, impl Struct), yet underneath it is dynamically typed reference counting (RC). This drives developers to instinctively agonize over compile-time ownership conflicts that do not exist, causing severe cognitive drain.
- **Security expectation unmet**: The developer writes tight type declarations, expecting the rigid enforcement of the Rust compiler, yet Rune throws a Panic at runtime because of an implicit type mismatch. This "looks like one thing, behaves like another" destroys Rust developers' most core, bottom-layer sense of coding safety.

### 2.2 The Argument: The Natural Cognitive Isolation Mechanism of a Lisp (Steel) Background

If the developer's first or fluent language includes Lisp (e.g. Scheme, Clojure, Racket), choosing Steel erects a perfect line of defense at both the physical and psychological levels:

- **Zero Mental Blending**: Lisp's S-expressions `(defn ...)` have an extremely distinctive visual signature. When you press the parenthesis `()`, the brain instantly switches to Lisp's pure-expression and metaprogramming mode; when you press the curly brace `{}`, it seamlessly returns to Rust's hardcore world of memory management and strong typing. The two muscle memories never fight.
- **Metaprogramming and contracts as a decisive advantage**: Using Lisp's peerless syntax-rules macros, one can, on top of Steel, very easily strip parentheses from the project and encapsulate an extremely clean domain-specific language (DSL). Together with its higher-order contract system, one can enforce extremely strong pre-/post-condition type constraints at the dynamic script layer.

## 3. Engineering Obstacles to Team Collaboration

Although Steel performs perfectly in personal sentiment and psychological isolation, entering a commercial general-purpose environment and team collaboration means confronting the following three hard red lines:

1. **Team "Lisp Phobia"**: Most modern mainstream engineers (Rust/Go/TypeScript) have a natural defensiveness and aversion toward Lisp's prefix expressions and dense parenthesis nesting. Forcing it in would turn the script layer into a solo performance by the architect, completely losing the original intent of team-collaborative extension.

2. **Tooling Abyss**:
   - Rune has a mature official Language Server (LSP), which directly provides silky smooth autocompletion, jump-to-definition, and formatting in VS Code/Helix.
   - Steel, as a relatively young Scheme dialect, currently lacks a mature IDE toolchain, deep static analyzers, and a local Debugger. The debugging cost for team members investigating script-layer bugs will rise exponentially.

3. **Async engineering impedance**: If the business involves an extremely overlapping, high-concurrency async state-machine network (e.g. frequent communication among a multi-agent Swarm), Steel's callback- and Promise-based bridged async is not as intuitive and efficient as Rune's native reproduction of `async fn` + tokio integration.

## 4. The Decision Chain: From a Single Script to a Polyglot Harness

### 4.1 The Traditional Binary Choice (Obsolete)

```
[Development Decision Orientation]
│
┌───────────────────────┴───────────────────────┐
[Business-led / team-wide collaboration]        [Personal geek deep customization]
│                                               │
(Imposing Lisp meets too much friction)         (Leveraging the long-term edge of a first-love language)
│                                               │
[Choose Rune scripting]                         [Firmly adopt Steel]
│                                               │
(Mandate that the script layer use no           (Use its superb metaprogramming
static type annotations, name things            to build a parenthesis-free DSL,
in snake_case, and build a violent              and erect an absolute psychological
visual barrier against the host                 and visual defense wall against
Rust)                                           Rust's curly braces)
```

### 4.2 The Ultimate Solution: Polyglot Embedded Harness

In industrial-grade scenarios, a single embedded language cannot satisfy all needs. The framework provides four runtime capabilities, but **which language to use is decided explicitly by the user according to business needs**; the framework does no intelligent dispatch.

```
[User selects the language based on business needs]
    │
    ├─► Need a DSL / policy engine? ──────────────────► [Steel Lisp]
    │                                                   └─ 100% deterministic boundaries, zero ambiguity
    │
    ├─► Need async network crawling? ─────────────────► [Rune]
    │                                                   └─ Native async/await, fits into Tokio
    │
    ├─► Need a third-party plugin sandbox? ───────────► [Wasm]
    │                                                   └─ Hardware-level isolation, cross-language compilation artifacts
    │
    └─► Need the AI/ML ecosystem? ────────────────────► [PyO3]
                                                        └─ Zero-overhead reuse of the Python ecosystem
```

**Key clarification**: This is not "the framework dynamically dispatching according to task characteristics," but "the user explicitly choosing according to the business scenario." The framework only provides runtime capabilities; the user decides which language to use in which module. For example:

- When the user writes a permission-routing module, they **actively choose** Steel Lisp to write the policy rules
- When the user writes a crawler module, they **actively choose** Rune to exploit async/await
- When the user needs to integrate a third-party compiled plugin, they **actively choose** the Wasm sandbox
- When the user needs to call a PyTorch model, they **actively choose** PyO3

At the same time, the following are maintained:

- **Zero Redis pollution**: all state lives in booth memory, written to PostgreSQL in a single transaction
- **~20ms system startup**: binary startup + Tokio runtime initialization + Fjall open (booth wakeup is microsecond-level; embedded VMs are created instantaneously)
- **Zero-copy data exchange**: the four runtimes share the same memory address space, passing pointers directly

### 4.3 User-Driven Polyglot Routing

In a distributed agent architecture (Fjall + independent metadata + Polyglot Core), the user's will writes directly to this node's metadata instance (control-plane single-writer), without consensus broadcast. The framework is no longer a judge, only a humble provider of power; the dynamic will of the user and the AI is the supreme commander that decides which language takes the polyglot memory stage in this millisecond.

#### The User-Driven State Command Protocol

```rust
// src/store.rs - user-driven polyglot execution proposal
#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum EngineType {
    Steel,  // Lisp policies with absolute visual determinism
    Rune,   // Rust-like coroutines for heavy, high-concurrency async
    PyO3,   // Python that freeloads the world's AI and data-science assets
    Wasm,   // Black-box sandbox plugins at near-bare-metal speed
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum RaftCommand {
    UpdateBoothState {
        agent_id: String,
        serialized_context: Vec<u8>,
    },
    // An arbitrary-language execution proposal dynamically initiated by the user or the AI
    DispatchUserScript {
        agent_id: String,
        engine: EngineType,      // the language type decided by the user
        script: String,          // the dynamic script submitted by the user
        target_function: String, // the target function to call
        input_payload: String,   // input data
    },
}
```

#### The Dynamic Polyglot Dispatcher: User Will Realized in Memory at Full Speed

Once the user-submitted `DispatchUserScript` has been written to this node's metadata instance, the local state machine maps physical memory pointers to the corresponding language VM within seconds, according to the user's choice:

```rust
fn execute_user_chosen_engine(
    &mut self,
    engine: &EngineType,
    script: &str,
    func_name: &str,
    input_data: &str,
) -> Result<String, String> {
    // 1. Flash-fetch this agent's persisted state out of the Fjall LSM-Tree
    let raw_state = self.booth_partition.get(agent_id.as_bytes())
        .unwrap().unwrap_or_default();
    let current_status: String = bincode::deserialize(&raw_state)
        .unwrap_or("idle".to_string());
    
    // 2. Dispatch the corresponding embedded engine according to the decision of the human user (or AI control center)
    match engine {
        // [User chose Steel]: high-risk gating, permission firewall; valuing Lisp's whitespace-free ambiguity and safety
        EngineType::Steel => {
            let mut vm = steel::steel_vm::engine::Engine::new();
            vm.register_value("current-status", /* ... */);
            vm.register_value("input-raw", /* ... */);
            let outputs = vm.run(script).map_err(|e| format!("Lisp Error: {:?}", e))?;
            Ok(format!("{:?}", outputs.last()))
        }
        
        // [User chose Python]: run HuggingFace Tokenizer matrix computation locally on damaged logs
        EngineType::PyO3 => {
            pyo3::prepare_freethreaded_python();
            pyo3::Python::with_gil(|py| {
                let module = pyo3::types::PyModule::from_code_bound(
                    py, script, "user_mod.py", "user_mod"
                )?;
                let func = module.getattr(func_name)?;
                let py_res = func.call1((current_status, input_data))?;
                Ok(py_res.to_string())
            })
        }
        
        // [User chose Rune]: mount an async state machine that concurrently crawls 10 API endpoints
        EngineType::Rune => {
            // Spin up the compact Rune stack VM and throw its compiled Future straight to Tokio
            // rune_vm.run().await...
            Ok("Rune Async Success".to_string())
        }
        
        // [User chose Wasm]: a closed-source decryption module compiled from C++/Go and submitted by a third party
        EngineType::Wasm => {
            // Wasmtime wakes up instantaneously, runs bare-metal compute acceleration, and isolates physically
            Ok("Wasm Execution Success".to_string())
        }
    }
}
```

#### The Engineering Payoffs of the User-Driven Mode

1. **The user holds the absolute "syntax non-conflict right"**:
   If you want to write a core control policy that must interact with a large model frequently and that you collaborate on deeply inside Neovim, you can immediately issue the instruction: "For this step I'll run Steel Lisp." This guarantees your brain is fully immersed in the absolute geometric boundaries of parentheses, completely free from the torment of Koto-style implicit-whitespace semantic landmines.

2. **The AI agent holds the "ecosystem self-selection right"**:
   When your cloud-hosted Hermes brain discovers: "The next task requires reading a complex deep-learning .bin weight file, or analyzing a stretch of legacy PyTorch matrix," the AI will itself write in the distributed proposal: `engine: EngineType::PyO3`. Through pure memory pointers, it seamlessly consumes Python's AI ecosystem directly inside the current Rust process.

3. **A perfect unification of multiple languages on the Fjall disk**:
   No matter whether the user just capriciously chose Lisp or Python, their mutation of the agent state is ultimately deserialized back into the most basic binary memory block (`Vec<u8>`), written to disk in a single pass, and locked in a highly compressed form into the local Fjall LSM-Tree physical drive.

## 5. The Rejected Candidate: Koto

### 5.1 Engineering Arrogance: Ideology Hijacking Ergonomics

Koto's whitespace sensitivity and Redis's early all-in-memory single-threaded design represent the same class of architectural trap: to sustain some nearly obsessive "design aesthetic" or "specific metric," they spare no cost to the developer's baseline sense of safety, engineering predictability, and ergonomics, offloading all cognitive load and error risk onto the human engineer sitting in front of the screen.

**Redis's all-in-memory obsession**: Back then, to sustain an absolute speed metric, early Redis exercised extremely strict restraint over persistence and multithreading. This forced enterprise-grade developers to write extremely complex external cache-eviction logic and to bear the psychological shadow of data loss on power failure. Although it became a speed legend, it left a heavy operational burden for modern complex systems that need to handle persistent state.

**Koto's minimalist visual obsession**: Koto's author wanted too much to replicate, within the Rust ecosystem, the "perfect, visually impurity-free" minimalist picture of CoffeeScript and Python. They obsessively wanted to eliminate all parentheses, braces, and semicolons. But at what cost? To sustain this surface-level "cleanliness," they forcibly promoted "invisible whitespace" into a heavy-industry-grade operator with strong semantic splitting power.

### 5.2 The Ultimate Scrutiny from the Lisp Perspective: Why This Violates First Principles of Engineering

Because the entry-point language is Lisp, this design is philosophically a rebellion against nature. It openly violates computer science's first golden rule: **Explicit is always better than Implicit**.

In the Lisp world, the parenthesis `( )` is not some "visual impurity" at all; it is the abstract syntax tree (AST) directly, nakedly exposed to the human eye, endowed with absolute physical boundaries.

- In Lisp you can stuff 50 spaces into the code, change 10 lines, or even compress an entire project's code into one long line. As long as the geometric topology of the parentheses is unchanged, the semantics stand rock-solid and no one can tamper with them.
- Koto, however, strips away this spatial-geometric sense of safety. It turns whitespace into a strong semantic, which is equivalent to turning "Nothingness" into a variable that may explode at any moment. If your Neovim does not have an extremely powerful formatter configured, or your teammate carelessly copy-pastes, during Git collaboration, code that mixes Tab and Space, they look completely identical to the naked eye, yet the execution logic inside the Koto VM may be poles apart. This is the archetypal "engineering nightmare."

### 5.3 The Veto Conclusion

If whitespace can determine whether a function receives a tuple or an expression, such an implicit bomb in a commercial production environment will sooner or later be detonated by some slip of the hand. Firmly veto Koto.

## 6. The Cultural Schizophrenia of the Rust Open-Source Ecosystem

This series of in-depth conversations reveals an interesting cultural schizophrenia (Paradox) across today's modern Rust open-source community:

1. **The host (the Rust language itself)**: built on extreme pragmatism. The compiler is incomparably strict, strongly typed and rigid, forcing you to declare absolutely explicitly every detail—lifetimes, exceptions, type conversions—when writing code.

2. **The parasite (script layers written in Rust)**: when this group of developers, accustomed to Rust's harsh scrutiny, goes to build embedded scripting or derivative tools, they tend to develop an overcompensation psychology. Looking at Rust's verbosity they think: "We want something completely uninhibited and extremely concise!"
   - **Rune** copied Rust's skin but pulled out the borrow checker's bones, leaving an "uncanny valley."
   - **Koto** wanted to be an elegant Python/CoffeeScript hybrid, but buried whitespace landmines in invisible places.
   - **Helix + Steel** chose a wonderful all-Rust architecture and a Lisp blueprint, yet built, around PR review, a political wall more bureaucratic, conservative, and inefficient than that of C-language projects.

## 7. Cross-References with the Editor Ecosystem

The selection of an embedded scripting language is tightly coupled with the editor plugin system:

- **Helix + Steel**: The plugin language officially chosen by Helix is precisely Steel Scheme, but the merge progress is extremely slow (see [Editor Selection Analysis](editor-selection-2026-en.md)).
- **Neovim + Fennel**: If you choose the Neovim ecosystem, you can, through Fennel (a Lisp-to-Lua compiler) or the conjure plugin, retain Lisp metaprogramming capability while enjoying a mature toolchain (see [Editor Selection Analysis](editor-selection-2026-en.md)).
- **Rune + any editor**: Rune's mature LSP can be integrated directly into VS Code/Helix/Neovim, with no editor-binding restriction.

---

## 8. First-Principles Engineering Decision Loop

This analysis, together with the critiques at the database layer and the editor layer, forms a complete engineering decision loop:

| Domain | Rejected technology | What was sacrificed | What illusion was bought |
|------|-------------|-----------|---------------|
| **Database** | Redis | Transaction safety, strong ACID consistency, multi-core scalability | "Pure in-memory speed" that is meaningless against network RTT |
| **Scripting language** | Koto | Execution determinism, toolchain maturity, human visual safety | The surface-level minimalist look of "no parentheses / whitespace as strong semantics" |
| **Editor** | Helix | Pragmatic evolution, plugin ecosystem maturity | The "aesthetic purity" that stalled the Steel plugin system for three years |

**The unified Occam's razor**: no tech worship, no swallowing the pie the open-source world draws, only the real physical hardware limits and team productivity.

→ **Database-layer critique**: [Redis: the "Network RAM" Trap](redis-critique-en.md) — the data-layer counterpart of this analysis.
→ **Language selection panorama**: [Programming Language Selection](language-selection-en.md) — the Rust/Lisp/Python/Nushell four quadrants and eliminated languages.
→ **WASM Unified Runtime**: [WASM Unified Runtime](wasm-unified-runtime-en.md) — after wasm-gc lands, Wasm evolves from a plugin sandbox into a general language backend, potentially reshaping the embedded-language landscape.
