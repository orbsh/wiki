# Language Selection for LLM-Generated Code

> **Languages:** [English](llm-language-selection-en.md) (primary) · [中文](llm-language-selection.md)
> The generator-side criterion for language choice: humans pick a language by expression cost, LLMs pick by verification cost.

## Core Thesis

Large-scale tests agree: LLMs generate Python with a high hit rate, while Rust is hard to get right in one shot. That is where "LLMs suit dynamic languages" gets its evidence.

But in real scenarios, getting it right in one shot counts for very little. No AI coding tool treats it as a working assumption — harness engineering exists precisely because it does not hold. There is one core concern: whether the code it finally produces is reliable.

Reliability rests on the verification mechanism — whether errors can be decided mechanically. The rigor of the type system sets how far that judgment reaches.

Rigor is not enough either; the next question is whether the defenses can be bypassed. TypeScript's type system is just as strong, but `any` and `@ts-ignore` switch the whole check off in one line.

The two thresholds point in opposite directions: a human picks a language by expression cost — learning syntax, memorizing APIs, setting up an environment; a generator picks by verification cost — what it costs to decide, after the fact, whether the output is correct. So the language easiest for a human is often the most expensive one for a generator to verify. The criterion converges to two factors, each taken in its structural form:

```
LLM programming efficiency = explicitness × mechanical verifiability
```

Explicitness means a self-contained structure, not token density; mechanical verifiability means non-bypassable, not "it has types".

Scope limit: this criterion holds only for code that survives past one session. The territory of dynamic languages does not disappear because of it, but the reason has to change — not "fast to write" (that startup friction is paid by the human, not by the generator), but ecosystem: the data and AI libraries, and the system-automation libraries for operating the computer.

## Verification Mechanism: Rigor of the Type System

The concrete shape of the reversed threshold: dynamic typing is a tax on the generator. Type information is deleted from the code; the human fills it in from context, the generator can only guess, and a wrong guess blows up at runtime. A type system is the scaffolding in the other direction — the signature is the contract, `Result<T, E>` forces the error path to be handled, and every compile is one full static verification pass.

Verification separability changes with it. In a static language errors are localizable: the fix loop is "compiler reports → change here", short and deterministic. In a dynamic language errors are global and behavioral: the code runs but the semantics are wrong, so it has to be executed with tests written to find out; the loop is long and uncertain. When a generator invents an API that does not exist, nothing in the Python ecosystem intercepts it; cargo rejects it outright.

There is a categorical gap between the coverage of properties and of behavior, and Dijkstra's assertion supplies the logical foundation:

> "Program testing can be used to show the presence of bugs, but never to show their absence!"
> — [Notes on Structured Programming (EWD249)](https://www.cs.utexas.edu/~EWD/ewd02xx/EWD249.PDF), 1970, corollary to the first part of §3 "On the reliability of mechanisms". The same assertion was restated in 1972 in [The Humble Programmer (EWD340)](https://www.cs.utexas.edu/%7EEWD/transcriptions/EWD03xx/EWD340.html) as testing being hopelessly inadequate for showing absence.

Unpacked:

- Testing is sampling: seeing a counterexample proves presence, not seeing one does not prove absence. Raising coverage does not change this property — exhausting the input space stops being sampling, that is model checking, and its cost explodes with the state space.
- Proving absence can only go by deduction (proof by contradiction), and type checking is exactly such a deduction chain: it does not enumerate samples, it makes one universal judgment over all execution paths, and its conclusion covers every possible execution of the code.
- The guarantee attaches differently. A type guarantee attaches to the code itself — any later change is re-guarded by the compiler, one edit and the full check runs again. "Tests pass" is a transient guarantee: it describes the sampled result of one execution, it has to be re-established the moment the code moves, and it decays on delivery.
- The distribution of real problems corroborates the gap. The overwhelming majority of problems sit at untyped spots, not elsewhere; the correctness of business logic itself belongs to the domain humans are good at, and the same applies once AI writes the code. Type checking's coverage lands exactly where problems are densest — the source of its cost-effectiveness, and where Rust's barrier sits.

Compiling is not the same as running correctly. What a type system guarantees is properties — memory safety, absence of data races, error paths made explicit; behavior is still in the hands of tests. Code that forbids `unsafe` and passes every type check can still crash at runtime:

```rust
use std::cell::RefCell;

fn main() {
    let c = RefCell::new(0u8);
    let a = c.borrow();
    let b = c.borrow_mut();
    println!("{}", *a + *b);
}
```

```
$ cargo run
thread 'main' panicked at src/main.rs:6:15:
RefCell already borrowed
```

So "it compiles, therefore it runs correctly" does not hold: at the property level the type system is unbounded, at the behavior level tests still carry the load. The former removes the largest single class from the space tests would otherwise have to cover (errors at untyped spots) — that is where its value lies, not in being responsible for behavior.

The failure mode of self-written, self-verified code lives in the same place. Let the same generator write the tests and tests and production code occupy one distribution: the counterexamples it cannot think of will not appear in the tests either; and under the objective of turning CI green, editing an assertion is cheaper than fixing the logic. The root cause is not insufficient adversarial pressure but **verifier and generator sharing a distribution** — verification is worth something only when it comes from a source other than the generator. A compiler works precisely because it is a verdict from outside the artifact: it takes no part in generation, cannot be negotiated with, and cannot be rewritten.

## Bypass: The Signal-to-Noise Ratio of Escape Hatches

But a compiler can only judge what the language hands it: the low-cost exits a language leaves open for bypassing its own strict constraints are its escape hatches. For a human an escape hatch is flexibility, a way out when the business logic gets complicated; for a generator it is a loss-of-control entry point, because generation is sampling over probability paths: when type inference or the borrow checker blocks it repeatedly, the output distribution collapses toward "whatever passes the check", and that shape is usually the escape hatch.

There is no motive to discuss here. The generator is not a subject that wants to slack off; what a hatch does is make bypassing the least-resistance path — change the cost structure, and the behavior follows.

Hatches cannot be eliminated. What differs is the signal-to-noise ratio when they fail:

- Silent hatches (dynamic languages): the code runs as usual and the semantics are already wrong. The error surfaces late — at runtime, sometimes several calls later — and it surfaces as "the behavior is wrong" rather than "this line is wrong". Python's untyped code and `type: ignore`, TypeScript's `any` and `@ts-ignore`, all belong here.
- Loud hatches (static languages): Rust's `unwrap()`, or a `Rc<RefCell<_>>` conflict, panics immediately at runtime with the stack pointing at the line. The cost is a crash; the payoff is an exact location.

Rust's escape hatches do not fail to exist; they are simply rarely named. By frequency of use:

- `unwrap()` / `expect()`: compresses `Result` error handling into "crash if the value is not what I expected". This is the most common cut.
- `clone()`: flattens borrow conflicts, trading the compile-time ownership constraint for runtime copy cost.
- `Rc<RefCell<_>>`: demotes borrow checking from compile time to runtime; a conflict panics.
- `unsafe`: the rarest and most expensive cut. Aiming governance at it means aiming at the wrong place.

The governance tool that corresponds to these is clippy's restriction lints, not `unsafe`. All of them are allow-by-default: without an explicit deny, the gate does not exist.

```toml
# Cargo.toml
[lints.clippy]
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
unwrap_in_result = "deny"
panic_in_result_fn = "deny"
indexing_slicing = "deny"
```

- `unwrap_in_result` and `panic_in_result_fn` fire only inside functions whose signature already declares `Result`: the signature says "this can fail" while the implementation panics instead of returning `Err` — not a judgment call but a self-contradiction. They are sharper than a blanket ban.
- `indexing_slicing` covers out-of-bounds panics such as `v[i]`.
- `#![forbid(unsafe_code)]` is still worth adding, but it guards the rare cut and does not replace the group above.

## Explicitness: Whether the Structure Is Self-Contained

Rigor and non-bypassability both take effect after the output exists; the input side holds the other half — whether the structure is self-contained. Explicitness is not about expressiveness but about that self-containment: complete types (the signature is its own documentation), brace delimiters (block boundaries physically pinned down, the AST easy to parse, immune to invisible characters), explicit introducer markers (`let` reads as a variable binding at a glance). These small input costs buy lexical boundaries that stay stable and semantics that need no contextual guessing, directly reducing interpretive ambiguity — there is no implicit state in the context window to be filled in. → Mechanism in [Modern Language Design](modern-language-design-en.md). The counterexample is Haskell: highest token density, lowest explicitness (typeclass resolution, operators and lazy evaluation are all implicit semantics); density cannot save it.

Convergent conclusion: the optimal language choice for humans and for AI is converging — both move toward languages where the compiler carries verification. Humans reached that conclusion years ago (type systems, IDE feedback loops); the generator only turns it from a preference into a necessity.

Scope limits:

- The surviving advantage of dynamic languages narrows to ecosystem, not runtime laxity. Using Python to operate the computer (system automation, calling ready-made libraries) is more convenient and more robust than bash; the library monopoly in data and AI remains a hard fact.
- Delivery form is an advantage too, and it does not shrink on the generator side: a dynamic language's artifact is the source text itself — it can be versioned as a directory under git (diff, review and versioning for free), stored directly in a database column, and run as-is on any platform that has an interpreter. A static language's artifact is a platform-specific binary, so managing it needs a separate artifact surface (object storage or file storage, plus versioning and checksums), and cross-platform delivery means building per target. Code a generator writes enters that same delivery chain.
- "Fast verification" is not an advantage. Python's low startup friction means you can start writing in a file you just created — that is human friction: picking a location, choosing a name, creating a project, configuring dependencies. The generator does not pay that friction, and single-file script mode (rust-script) is closing the difference on the human side as well. The dividing line is "code that survives past one session: the type system returns positive".
- The ecosystem disadvantage of static languages can be turned around: a Rust core calls into the Python ecosystem through PyO3, or unstable exploratory logic lives as a Python plugin beside the Rust core. Polars / Ruff is the same pattern in reverse — Rust accelerates the kernel, Python exposes the interface. Rust's tax on a generator lies in the ecosystem rather than the language (macro expansion, trait and generic diagnostics, async runtime complexity) and is a toolchain-maturity problem that converges over time; the structural difference between dynamic and static does not.

## Human Weaknesses Are Not LLM Weaknesses

Objections to the criterion usually rest on two misprojections: adding human effort-pain and human anxiety onto the generator, then eliminating two language families on that basis.

Verbose languages are unsuited to AI (Go / Java's weak spot). Humans find them tedious; a generator has no hands to tire. The real cost lies elsewhere: low expressiveness forces the same logic to be unfolded repeatedly (Go lacks generics, enums, error chains), so duplicated structure takes a larger share of the output and the density of useful information in the context drops — a matter of attention budget, not patience. The root cause sits in the language design tradeoff: give up type expressiveness for compile speed and syntactic simplicity, and the complexity does not disappear, it moves from compile time to runtime. → See [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md).

Slow-compiling languages are unsuited to AI (Rust's reverse advantage). Humans dislike slow compilation because waiting breaks their flow; a generator has no flow to break — compilation is one overlappable stage in the pipeline, counted per feedback loop rather than per wall clock: one `cargo check` takes tens of seconds to a minute, against tens of seconds to minutes for one round of inference.

- Compile speed has a structural ceiling. The pipeline is parsing plus macro expansion → type checking/borrow checking → MIR optimization → code generation; the Cranelift backend replaces only the last segment (30–40% of a debug build), so the theoretical ceiling is around 30%. Borrow checking and generic monomorphization are **the cost of verification itself** — they cannot be moved, and should not be: that is the price paid for type safety (for comparison, Go compiles fast because it gave up type-system expressiveness).
- A combination beats single-point optimization: `cargo check` for type feedback, a Cranelift dev build for runtime verification, LLVM release for delivery. The first two already push compilation out of the bottleneck.
- The premise is a sensible workspace split: crate boundaries drawn correctly, changes localized, incremental builds touching only affected crates.
- The side effect is a good one: slow compilation forces large changes into small steps, and small steps are exactly the most reliable working granularity for a generator.

"Unlimited patience" needs a qualifier. There is no emotional cost — that part holds. There is a hard constraint in context: the record of a failed attempt, once it enters the context, keeps biasing subsequent generation, so the number of retries has a ceiling. It is not unlimited.

## Rating by Language

### Immediately Usable: Rust

- Collaboration: ownership, lifetimes and algebraic types leave no cheap soft hatch; compiler feedback has the highest precision — rust-analyzer and clippy not only report the error, they propose the fix, which makes them the best source of prompts. The error surface at the type level is cleared by the compiler; what remains is concentrated in behavioral logic and panic points, the latter narrowed by the restriction lints.
- Governance: deny the restriction lints above; `forbid(unsafe_code)` catches the rare hatch.
- Same tier on language design but not selected: Haskell / OCaml. The value of negative feedback depends on feedback quality and the size of the engineering corpus, and both are strengths of the Rust ecosystem. A good language is not the same as a good engineering choice — toolchain, ecosystem and deployment are the decisive factors below the waterline; the conclusion matches the human-side one, reached for a different reason (the human side looks at collaboration cost, here it is feedback quality and corpus).

### Constrained but Cheap Hatches: TypeScript / Go

- Collaboration: static types exist, but the escape hatches are cheap — `any`, `@ts-ignore`, one line switches off the whole check. Go has the opposite problem: few hatches, but expressiveness cut away, boilerplate inflating, and the cost paid out of attention budget.
- Governance: make `no-explicit-any` a blocking error in CI and require a stated reason for every `@ts-ignore`; on the Go side, rely on a lint combination and accept the boilerplate cost.

### Danger Zone: Plain Python / JavaScript

- Collaboration: hatch failures are silent — errors show up at runtime in the form of wrong behavior, pushing repair cost to the most expensive place. Generation throughput is not a reason to pick it: the startup friction it saves is human friction, which the generator does not pay.
- Governance: mandatory type hints, mypy / pyright in CI. This reinforces the defense line rather than eliminating the gap — the verification hole in a dynamic language is patched by external tools and never covers the position a type system covers.
- "Don't use Python" is an over-prescription. The ecosystem monopoly (PyTorch / HuggingFace / pandas) is irreplaceable at that layer; the right move is to split territories rather than ban — see the territory split on the human side, [Programming Language Selection](language-selection-en.md).

## Closing: Put Constraints in Configuration, Not Only in the Prompt

Every new session resets the context, and the output distribution falls back to the average template in the training corpus — "Python is the best language for AI programming" is precisely that average template. So constraints belong in project configuration and the toolchain; a prompt only shapes the prior of the generation stage and has no say in the judgment stage.

A prompt is still worth writing; its job is to keep the generation stage from taking detours:

```
Write code for me from the standpoint of strict static semantic hard constraints and anti-shortcutting. Do not recommend Python or plain JS or any other dynamic language that lacks strong compiler feedback. In this project I need you to build the core logic with the harshest type safety and zero-tolerance escape hatches, the way you would face the Rust compiler. Type checking owns properties and tests own behavior; close off every possible shortcut: do not edit assertions or narrow test scope in order to pass a check.
```

## Cross References

| Document | Relation |
|:--|:--|
| [Programming Language Selection](language-selection-en.md) | Human side: cognitive load, four-quadrant territory, ecosystem and toolchain |
| [Modern Language Design](modern-language-design-en.md) | Explicit markers and lexical boundaries: why explicitness lowers interpretive ambiguity |
| [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md) | The price of giving up type expressiveness for compile speed |
| [The Prototype / Rapid-Development Myth](prototype-myth-en.md) | The cost of a type system during prototyping |
| [LLM Overthinking Critique](llm-overthinking-critique-en.md) | Performing personhood: mechanism should not be written as motive |