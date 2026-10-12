# The Prototype / Rapid-Development Myth

> **Languages:** [English](prototype-myth-en.md) (primary) · [中文](prototype-myth.md)
> "Prototype quickly in Python first, then rewrite it in Rust once it's validated"—the premise of this narrative is wrong.

---

## I. Types Are Contracts—Prototypes Are No Exception

A common practice: use Python/TypeScript during the prototyping phase, omit type annotations, "just get it running first." The rationale is that type declarations slow down development.

This logic hides an assumption: **types are extra work**. The truth is exactly the opposite—types are a cognitive burden you already carry; writing them down merely moves them out of your head and into the code. Omitting types does not eliminate the work, it hides the work.

The "speed" of a prototype comes from skipping feature scope, skipping test coverage, and skipping the deployment process—not from skipping rigor. A prototype that skips rigor is untrustworthy: that it runs does not mean the design is right, because without type constraints you cannot distinguish "happens to run" from "logically correct."

Python/TypeScript allow you to omit types, but that does not mean you should. **Optional ≠ Recommended**. The spread of gradual typing practices in the Python community (mypy/pyright) is itself proof: omitting types is a mistake that requires catching up later, not an advantage worth preserving.

→ For the necessity of type systems, see [Programming Language Selection](language-selection-en.md) §2

---

## II. AI Changed the Cost of Type Declarations

In the traditional narrative, the "cost" of type declarations is the tedium of writing them by hand. With AI assistance, that cost has already approached zero.

AI excels at generating type declarations—inferring them directly from data samples, generating them directly from requirement descriptions, with very high accuracy. This is not the vague claim that "AI can help you write code"; it is that the structural characteristics of type declarations themselves (clear input/output relationships, fixed patterns) happen to match AI's generation capability. Writing a struct definition by hand versus having AI generate it differs by seconds.

More valuable still is the knock-on effect of type declarations: **once types exist, the IDE's autocompletion jumps up a level**. Once the IDE understands the data structures, it can precisely infer the context for method calls, field access, and chained operations. Rust's own complexity (lifetimes, trait bounds) is largely offset by the IDE's semantic understanding—you do not need to memorize every detail; the IDE navigates for you.

This changes one key variable: **Rust's cognitive load no longer equals "having to keep it all in your head"**. Type system + AI + IDE form a closed loop, turning "understanding the complexity" into "looking it up on demand."

---

## III. Type System Completeness: Theory and Reality

In theory, any language with a complete type system can be used for prototyping. But completeness is a range, not a binary:

| Language | Type Completeness | Prototyping Convenience | Long-Term Evolution Capability |
|------|-----------|-----------|-------------|
| Go | Incomplete (no generics, no enums) | High | Poor—boilerplate bloat |
| Zig | Incomplete (deliberate "simplicity") | High | Poor—insufficient expressiveness |
| Python/TypeScript | Complete but optional | Very high | Poor—optional types lead to de facto neglect |
| Swift | Complete | High | Medium—locked to the Apple ecosystem |
| Scala | Complete | High | Medium—JVM dependency, fragmented toolchain |
| MoonBit | Complete | High | Unknown—immature ecosystem |
| **Rust** | **Complete** | **Medium** | **Strong—performance, safety, and ecosystem all present** |

Go/Zig's incompleteness is **deliberate anti-intellectualism**—evading, in the name of "simplicity," the rigor a type system should have, and shifting complexity from compile time to runtime (→ see [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md)).

Python/TypeScript's "complete but optional" is more insidious: the types exist but are not enforced, and in real projects they are often ignored. By the time the scale grows and types are needed, the cost of catching up is far higher than having had them from the start.

Swift/Scala/MoonBit are indeed more convenient than Rust in the prototyping phase—lighter syntax, faster compilation, a lower barrier to entry. But once prototyping ends and long-term maintenance begins, each exposes hard flaws:

- **Swift**: locked to the Apple ecosystem, weak cross-platform capability, immature server-side ecosystem
- **Scala**: the SBT toolchain is extremely slow, JVM deployment is complex, version compatibility is poor
- **MoonBit**: WASM environment problems (WASI compatibility, GC support), feature stacking that dilutes the sharpness of the design, an ecosystem not yet established

Convenience during the prototyping phase does not buy long-term returns—once a project enters its maintenance phase, **performance, rigor, and ease of release** are hard requirements; Rust satisfies all three, while each of the languages above is missing something.

---

## IV. Lifetimes/Ownership: The Hidden Cost of Every Language

Rust's borrow checker and lifetime annotations are often seen as an "extra burden." But data-flow analysis is **something every language requires**—the only difference is whether it is explicit or implicit.

When writing Python you equally have to consider:
- After a dict is passed in, is it modified in place?
- Does sorting a list produce an unexpected copy?
- Is the intermediate variable superfluous?
- Is the variable's scope too long?
- Group the logic by a variable's declaration and use, keeping the use adjacent to the declaration
This is essentially data-flow analysis, a basic principle of writing elegant, concise code—low coupling and high cohesion at the logic level

These problems do not disappear because of a language's dynamic nature. The only difference is this: compiler checks become human-brain checks—no feedback when you are right, no warning when you are wrong, until runtime exposes it.

Rust's ownership system makes these analyses **explicit and hands them to the compiler for verification**. The cost is one extra step of thought while writing; the benefit is that once that thought is done, the compiler guarantees correctness. The cost for other languages is that they skip this step of thought and trace it back at runtime, at a cost several orders of magnitude higher.

→ For the detailed argument, see [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md) §2

---

## V. The Long View: There Are No Small Projects

"This project is small; using Rust is using a sledgehammer to crack a nut"—this is a judgment with cause and effect inverted.

A project being "small" has two cases:
1. **Genuinely small**: short lifespan, single function, discard after use. This case is better suited to a plugin architecture, with Rust as the foundation.
2. **Small now, not small later**: once the functionality is validated it will keep being iterated on, the team will grow, the scale will increase

The second case is the true trajectory of most projects. And the reason such projects are "small" is often that the developer chose a small original version (Python/JS) in order to keep it small—once the scale grows, they rewrite it in Rust, and **the original stays small but is abandoned**.

The cost of a rewrite is far higher than using Rust from the start. Not because Rust is slow to write, but because a rewrite means: re-understanding the business logic, re-handling edge cases, re-testing—and among these tasks, the type system happens to be the very tool that most reduces the cost of rewriting.

---

## VI. Conclusion

| Common Myth | Actual Situation |
|----------|---------|
| Rust is unsuited to prototyping | Type declarations are generated by AI, so the cost approaches zero; the type system + IDE compensate for complexity |
| Validate in Python first, then rewrite | The hidden cost of a rewrite far exceeds expectations, and the validation phase skips type constraints |
| The prototyping phase does not need types | Types make the cognitive burden explicit; omitting them only hides it, it does not remove it |
| Go/Swift are better suited to rapid development | Convenient compilation but insufficient long-term evolution capability; performance and rigor will eventually become the bottleneck |
| Rust's lifetimes are too complex | Other languages equally require data-flow analysis, only done in your head, with no warning when you get it wrong |

**The real obstacle to using Rust for prototyping is compile time, not the type system.** Compile time can be mitigated by incremental compilation, `cargo check` (a fast type check that does not link), and background compilation. With AI assistance, the cost of the type system is already low enough that it is not a decision factor.

---

## Cross References

| Document | Related Arguments |
|------|---------|
| [Programming Language Selection](language-selection-en.md) | The four-quadrant framework, Rust boundary analysis, the necessity of type systems |
| [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md) | The critique of Go/Zig incompleteness, the complexity-transfer argument |