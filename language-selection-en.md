# Programming Language Selection

> **Languages:** [English](language-selection-en.md) (primary) · [中文](language-selection.md)
> The essence of tool selection is cognitive load management. Choosing a language is not about picking the "best" one, but the one "best suited to your current mode of thinking."

---

## 1. Selection Logic

Two dimensions cross, producing four quadrants, which four primary languages happen to fill exactly:

```
                      Compiled                         Scripted            
         ┌────────────────────────────────┬────────────────────────────────┐
   Static│              Rust              │            Nushell             │
     Type│    Systems / Infrastructure    │ Structured Scripts / Pipelines │
         ├────────────────────────────────┼────────────────────────────────┤
  Dynamic│              Lisp              │             Python             │
     Type│         Embedded / DSL         │        Data / AI / Glue        │
         └────────────────────────────────┴────────────────────────────────┘
```

Static typing + a scripted language is a rare combination—Nushell and [Gluon](https://gluon-lang.github.io/) are among the few languages that occupy this quadrant at the same time. Most statically typed languages choose compilation (Rust/C++/Go), and most scripted languages choose dynamic typing (Python/Ruby/JS). What makes Nushell unique is this: **it uses static typing to guarantee the correctness of pipeline data while preserving the interactive experience of a scripted language**.

The four languages form a perfect symmetry: each language exclusively occupies one quadrant, with no overlap.

Core principle: **each language has a clearly defined territory and does not cross its boundary**. Rust does not write glue, Python does not touch the low level, Lisp does not do I/O-intensive work, Nushell does not do complex logic.

### The Distribution Model Is Counterintuitive to Engineers at the User Level

Engineers often assert that "advanced users like Rust, while ordinary users are suited to Python"—but this describes **the engineers' own aesthetic** (Rust's typing/performance/modernity), not the users' actual orientation of needs. Seen at the user level, this orientation is **reversed**:

- **Rust's "runtime-free distribution" is meaningful only to ordinary users**: a single static binary with no runtime to install is an entry point for "grab and use".
- **For advanced users it is instead more troublesome**: advanced users have usually already installed various runtimes and are used to `git` / flake source management (especially on NixOS)—what they want is not "no runtime to install" but "source that can be modified directly".
- **Python's "runtime already installed + source editable and immediately usable" satisfies advanced users precisely**: dynamic scripting, no compile wait, zero friction when modifying core interactions.

So: **Rust's "runtime-free" appeal to ordinary users and its "modifiability" appeal to advanced users are two conflicting sets of requirements**. For a project whose innovation lies in **UX and relies on advanced users modifying and extending it themselves**, the mode of propagation is the **source repository** (`git`/flake, editable and immediately usable in the development state), not distribution to unfamiliar users—in this case **Python's modifiability outweighs Rust's distribution convenience**. Engineers easily project their own Rust preference into "advanced users should love Rust too"; in reality what advanced users want is modifiability, and Python fits better instead.

This also explains why projects of the "UX innovation × advanced users modifying it themselves" kind are better served by a scripted core (this likewise supports, in selection terms, **not reversing the verdict on a scripted core on account of "distribution to unfamiliar users"**—the runtime-free distribution advantage of static compilation has nowhere to cash in under this propagation model).


---

## 2. Rust: Statically Compiled

### Positioning

A systems language. Scenarios needing determinism, performance, and memory safety—infrastructure, runtimes, CLI tools, embedded.

### Why Rust

- **Zero-cost abstractions**: generics, traits, pattern matching—the expressiveness of a high-level language with the performance of C
- **Ownership system**: eliminates data races and memory errors at compile time, with no need for a GC
- **Ecosystem**: Cargo + crates.io is currently the most active systems-language ecosystem
- **Toolchain**: rust-analyzer, clippy, rustfmt—an industrial-grade toolchain out of the box

### Boundaries

The conventional narrative holds that Rust's cost is "cognitive load"—the borrow checker, lifetime annotations, and trait bounds impose a significant mental burden—and then rationalizes this cost with "the compiler shoulders the runtime risk for you".

This narrative inverts cause and effect. **Rigor is non-negotiable, and the type system is the lowest-cost carrier of rigor**.

For medium-to-large long-lived projects, rigor is what prevents rapid decay. Except for prototypes, rigor is an inevitable requirement. The question is: is rigor without a type system feasible? Technically feasible—but at a higher cost:

- The relevant constraints are stored in working memory, a greater mental burden; when they cannot be held, comments are needed
- Comments carry more problems (→ see [Why You Shouldn't Write Comments](why-not-write-comments-en.md) for details), and their actual maintenance cost far exceeds that of type annotations
- They are not types, so they cannot rely on compiler verification. For common, scattered, interrelated logic, no unified context can be formed.

Rust's borrow checker and lifetime annotations are often regarded as an "extra burden". But data-flow analysis is **something every language needs to do**—the only difference is whether it is explicit or implicit.

When writing Python, you likewise need to consider:
- Will the dict be modified in place after being passed in?
- Did sorting the list produce an unexpected copy?
- Is the intermediate variable redundant?
- Is the variable's scope too long?
- Group logic by a variable's declaration and use, with the use next to the declaration
This is essentially data-flow analysis, and the basic principle for writing elegant, concise code: low coupling and high cohesion at the logical level

These problems do not disappear because of a language's dynamic nature. The only difference is that compiler checks become human-brain checks—no feedback when correct, no warning when wrong, until runtime exposes it.

Rust's ownership system makes this analysis **explicit and hands it to the compiler for verification**. The cost is one extra step of thinking while writing; the payoff is that once the thinking is done, the compiler guarantees correctness. The cost in other languages is omitting this step of thinking and tracing back at runtime, at a cost several orders of magnitude higher.

During the prototype stage it can be set aside for the moment, but once a project enters its maintenance phase it must eventually be faced—and you then find that the final structure is basically identical to what Rust requires. The difference is that in other languages this analysis is done by the human brain and cannot be verified by machine, whereas Rust requires you to complete it while writing and the compiler verifies it when you are done. A bit stricter, but not excessively so.

Python's gradual-typing practice in recent years (mypy/pyright) also proves the same direction: **the type system is not optional decoration but an engineering necessity**. The only difference is that Rust is a seamless whole by nature, while Python is catching up after the fact.

→ For the detailed argument, see [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md) §2

Scenarios where Rust should not be used:
- Rapid prototyping (compile-time penalty; but the cost of the type system itself has approached zero with AI assistance → see [The Prototype / Rapid-Development Myth](prototype-myth-en.md) for details)
- Pure glue/data processing (using a sledgehammer to crack a nut)
- Scenarios requiring hot reload (a natural disadvantage of compiled languages)
- Limited team cognitive budget (Rust's learning curve is a team cost)

### AI Perspective

The generator-side criteria (verification cost, escape hatches and governance, per-language rating) have been written up separately—see [Language Selection for LLM-Generated Code](llm-language-selection-en.md).

---

## 3. Lisp (Elisp / Steel): Dynamically Compiled

### Positioning

Embedded scripting language / DSL construction. Elisp is the soul of Emacs, Steel is the future of Helix (if it can be merged, that is).

### Why Lisp

- **Code as data** (homoiconicity): the macro system is Lisp's killer feature; metaprogramming in other languages is all imitation of Lisp
- **REPL-driven development**: interactive exploration, immediate feedback, suited to scenarios requiring repeated trial and error
- **Minimal syntax**: S-expressions eliminate syntactic ambiguity—the parser is precise by nature
- **Embedding-friendly**: Steel, as a Rust-embedded Lisp, starts up in milliseconds and interoperates with the host with zero friction

### Boundaries

Lisp's parentheses are not a problem—**minimal syntax is an advantage**. The real barrier is the mental model: you need to get used to the "code is a list" way of thinking.

Elisp's particularity: it is Emacs's "operating-system API", not a general-purpose programming language. Writing Emacs configuration in Elisp and writing business logic in Elisp are two different things.

→ For a detailed analysis of embedded Lisp selection, see [Embedded Script Language Selection](embedded-script-languages-en.md)

---

## 4. Python: Dynamically Scripted

### Positioning

Data processing / AI / rapid prototyping / glue language. It is not meant for writing "proper systems" but for writing code that "glues systems together".

The same territory applies on the generator side: code that is handed to an AI to write and must live beyond a single session should switch to a statically typed language—for the criteria, see [Language Selection for LLM-Generated Code](llm-language-selection-en.md).

### Why Python

- **Ecosystem monopoly**: the de facto standard in AI/ML/Data, with PyTorch/HuggingFace/pandas irreplaceable
- **Lowest cost to get started**: pseudocode-like syntax that even non-programmers can read
- **Lowers the project barrier**: Rust writes the core logic (embedded via PyO3), Python exposes the interface—developers with lower technical skill can use high-performance libraries without knowing Rust. Polars and Ruff both follow this pattern
  → For embedded script/runtime selection in detail, see [Embedded Script Language Selection](embedded-script-languages-en.md)
- **Glue capability**: calling C/Rust libraries, handling JSON, writing automation scripts—absurdly fast
- **NumPy/Pandas**: vectorized operations crush everything else in data-processing scenarios

### Boundaries

Python's problem is not the language itself but **using it for things it shouldn't do**:
- Web backends (Go/Rust are more suitable)
- System tools (neither performance nor reliability is sufficient)
- Large projects (the curse of dynamic typing erupts as scale grows)

The correct use of Python: **short-term, data-intensive, ecosystem-dependent**. For projects with a lifecycle beyond 6 months, seriously consider whether Rust/Go should be used.

---

## 5. Nushell: Statically Scripted

### Positioning

**The strongest shell for day-to-day Linux operations and system administration**. Its core advantage in one sentence: **structured data replaces "everything is text"**. A traditional shell's pipeline carries an untyped stream of text; Nushell's pipeline carries typed structured data (tables, numbers, dates, file sizes). Data pipelines are the natural extension of structured data.

### Why Nushell

- **Strongest for system administration**: the experience of day-to-day operations (file management, process monitoring, log analysis, system inspection) far exceeds that of a traditional shell
- **Static typing**: the data type of each step in the pipeline can be inferred before running, and type errors give immediate feedback in the interactive environment
- **Structured data**: `ls | where size > 1mb | sort-by modified`—every step is a structured table, not text parsing
- **Extremely powerful autocompletion**: extremely convenient to customize, far beyond zsh (bash isn't comparable)
- **Rust-native**: performance far exceeds bash/zsh, fast startup, low memory footprint
- **Ecosystem doesn't matter**: to call another program, just call it directly—does bash have an ecosystem? Did that stop it from being used everywhere?

### Data Analysis Capability

With the Polars plugin, it can replace Python in many data-analysis scenarios—but there are limitations

- **Insufficient engineering surroundings**: database drivers, format handling, algorithm libraries/frameworks, and so on are far less mature than the Python ecosystem.
- **Distribution friction**: when a Nushell script is shared with non-Nushell users, they lack the runtime environment and the knowledge
- **Tutorial resources/AI proficiency inferior to Python**: when you hit a problem, the reference material available and the AI capability are both a notch weaker

### Effector carrier Use Case

[Effector](stateless-agent-architecture-en.md) (the executor of the stateless Agent architecture) is the direct realization of Nushell's territory: among Effector's language carriers, nu handles system-interaction operations—operation code is dispatched from a task frame, and after the wrapper imports it, it feeds the JSON parameters in from stdin (`$in`), calls the entry function declared by the operation, and the result flows back as structured JSON. Execution is forced through nushell with no bash fallback: static typing keeps guarding the correctness of pipeline data across process boundaries, and AI-generated operation code is far more reliable in a structured pipeline than in a text shell. This is exactly the engineering landing point of the "static + scripted" quadrant among the four quadrants—system-interaction operations go through nu, data processing/AI through Python, embedded strategy through Steel, third-party code through Wasm.

### Boundaries

Nushell's combination of static typing + a scripted language is rare (similar to [Gluon](https://gluon-lang.github.io/)). Its territory is **system interaction and day-to-day operations**—this is not "suitable" but "the strongest".

---

## 6. Eliminated Languages

### JavaScript / TypeScript: Proficient but Disliked

Proficiency does not mean it is worth using.

Ecosystem problems are being systematically solved by an "integrate as much as possible" strategy:

- npm dependency hell, version conflicts, supply-chain security: Deno/Bun
- Frontend toolchain: Vite+

More important is **engineering quality**: professional organizations take responsibility for screening high-quality components, or directly improve performance while rewriting at the lower level—for example, Bun rewrote a large number of components in Zig (database drivers, object storage, etc.).

Single-threading is not a problem either—single-threading by default constrains beginners, while experienced users have no shortage of ways around it.

**The real problem is the language itself**: a C-like classical style, tedious and redundant, plus tons of bad parts. No amount of progress in the type system and toolchain can patch the fundamental flaws at the level of language design.


### Lua / Go: Anti-Intellectualism

The shared characteristic of the two languages: **deliberately reducing complexity at the cost of sacrificing expressiveness**.

> "If the programmer is trying to solve a complex problem, and the language forces them to ignore the complexity, the complexity will come out sideways."
> — [Zig: Small Minded Language Design](https://ziglang.org/documentation/master/#Small-Minded-Language-Design)

Zig's critique hits the core problem of both languages precisely: in the name of "simplicity" they avoid the rigor that a type system ought to provide. The result is that complexity is not eliminated but merely shifted from compile time to runtime—debug time, production incidents, team communication cost. **It is simpler to use, but the complexity has shifted onto people.**

- **Lua**: no module system, no namespaces, global variables by default—the cost of "simplicity" is a maintainability disaster for large projects
- **Go**: no generics (until 1.18), no enums, no error chaining—an extreme of "less is more", resulting in a large amount of boilerplate code

→ For the detailed argument, see [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md) §2

### Scala / Haskell: Excellent Languages, Bad Environment

The design quality of the languages themselves is very high:
- Scala: fusion of functional + OOP, a powerful type system, elegant pattern matching
- Haskell: purely functional, monad abstraction, compile-time guarantees

But engineering is not just language design:
- **Toolchain**: SBT (Scala) is outrageously slow, Cabal/Stack (Haskell) is severely fragmented
- **Ecosystem**: poor version compatibility between libraries; upgrading is like defusing a bomb
- **Learning curve**: monad transformers, typeclass derivation—the mental cost is out of proportion to the actual payoff
- **Deployment**: huge compilation artifacts, complex runtime dependencies

Conclusion: **a good language ≠ a good engineering choice**. The language is only the tip of the iceberg; the toolchain/ecosystem/deployment below the waterline are the decisive factors.

### MoonBit: The Best Syntactic Hand, but the Door Is Not Open

MoonBit holds the most suitable syntactic hand in the Wasm race—modern algebraic data types + GC + no ownership burden—landing exactly in the market gap of "wants Wasm, doesn't want to learn ownership, can't stand Go's old syntax". The real landscape of the Wasm race:

| Language | Writing it | Artifact |
|----------|------------|----------|
| C/C++ | Heavy mental burden | Standard, small, but unsafe |
| Rust | Strong ownership burden (without AI assistance) | Standard, extremely small |
| Go | Old syntax | Standard, but runtime 1MB+ |

The gap is clear: **modern syntax, standard artifacts**. MoonBit has seized the first half and done the second half backwards.

**Core criticism: treating a life-or-death line as a detail that can be patched in later.** Its Wasm artifacts depend on a private runtime (non-standard imports) and cannot run on a standard Wasm runtime—the extreme 211B size is a demo format, not a delivery format. The correct path is: default artifacts 100% standard Wasm/WASI (accepting a runtime size at the Go level) → opt-in trimmed runtime for differentiation → then talk about ecosystem bridging. MoonBit reverses the order entirely: it boasts about the extreme first, adds features first, and treats "can it run on the standard" as a detail to patch in later, while the debt grows heavier with each feature.

**Behavioral pattern: narrative-driven, not product-driven.** Its survival does not depend on the market—it is funded by research grants through a government-backed research institute (IDEA Research Institute), and its evaluation criteria are academic influence and policy contribution rather than user satisfaction. This determines where resources flow: a full assault on four backends (Wasm/JS/Native/LLVM) + a full-stack framework + domestic chip adaptation, each front corresponding to a type of narrative (AI-native, full-stack vision, self-controllable), none of which corresponds to "seeing the standard Wasm product through". The very existence of the JS backend is itself a signal—insufficient confidence in the core Wasm route. The gap between the marketing rhetoric ("5-10x efficiency") and the artifact quality (won't run on a standard runtime) points to a hype-driven behavioral pattern. From this a pattern prediction can be made: **"pragmatically filling in standardization" is not on its evolutionary path**—filling in standardization yields no return for its survival structure.

**The Matthew effect is accelerating**: weak ecosystem → little engineering corpus → weaker AI capability → fewer adopters. Its corpus-accumulation strategy is tied to educational partnerships and competitions, feeding in textbook corpora rather than engineering corpora, with limited pull on both training value and practical usability. Educational input cannot offset the Matthew effect; it only delays the endgame.

Conclusion: Wasm requirement + acceptance of runtime overhead → Go or Rust; wait for MoonBit to address standardization first—and by its behavioral pattern, that day most likely will not come.

---

## 7. Core Conclusions

| Language | Quadrant | Territory | Core Value |
|----------|----------|-----------|------------|
| Rust | Static + compiled | Systems/Infrastructure | Determinism + performance + safety |
| Nushell | Static + scripted | Data pipelines/System interaction | Static typing + structured data |
| Lisp | Dynamic + compiled | Embedded/DSL | Metaprogramming + interactive + minimal |
| Python | Dynamic + scripted | Data/AI/Glue | Ecosystem monopoly + lowest cost to start |

**Selection principles**:
1. Each language has a clearly defined territory and does not cross it
2. Territory is decided by the problem, not by personal preference
3. "Proficiency" is not a reason to use it; "suitability" is
4. The language is only the tip of the iceberg—the toolchain/ecosystem/deployment are the decisive factors below the waterline

---

## Cross-References

| Document | Related argument |
|----------|------------------|
| [Embedded Script Language Selection](embedded-script-languages-en.md) | Steel vs Rune technical comparison, Rust embedded scripting ecosystem |
| [Editor Selection Analysis](editor-selection-2026-en.md) | The language bindings of Neovim (Lua) / Emacs (Elisp) / Helix (Steel) |
| [The Prototype / Rapid-Development Myth](prototype-myth-en.md) | Feasibility of Rust prototypes, AI-assisted type generation, the long-term project perspective |
| [Vim Movement Logic Critique](vim-movement-critique-en.md) | The indirect connection between editor operation paradigms and language selection |