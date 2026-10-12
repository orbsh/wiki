# Why You Shouldn't Write Comments

> **Languages:** [English](why-not-write-comments-en.md) (primary) · [中文](why-not-write-comments.md)
> Comments lie, because keeping them synchronized costs too much and no mechanism enforces synchronization. Eliminating the need for synchronization is more effective than maintaining comments.

---

## I. Comments Lie

Code changes; comments do not necessarily change with it. This is the most fatal defect of a comment — it is not verified by the compiler, so it is **untrustworthy**.

```python
# Compute the user's balance (excluding frozen funds)
def get_balance(user_id):
    return db.query("SELECT balance FROM accounts WHERE id = %s", user_id)
```

The comment says "excluding frozen funds," but the code queries the `balance` field — where are the frozen funds? This comment was written three years ago; there really was frozen-funds logic back then, it was deleted during a later refactor, and nobody deleted the comment.

**A lying comment is more dangerous than no comment**: with no comment, the reader goes and reads the code; with a comment, the reader trusts the comment and then skips the code. An outdated comment is a carefully disguised bug.

---

## II. Why Synchronization Is So Hard

The root cause of comment rot is not "programmers are lazy," but that **synchronization carries an extra cost, and that cost pays nothing back**.

```
Code changes → comment needs syncing → developer judges "is this comment still correct?" → manual edit → no mechanism verifies the edit was right
```

Every step has friction, and the last step has no safety net. The result is that comments inevitably rot.

**Every other code artifact has a safety net — only comments have none**:

| Item | Compiler check | CI check | Developer maintains proactively |
|------|-----------|---------|--------------|
| Code | Yes | Yes | Yes (it won't compile, so it gets fixed) |
| Type signature | Yes | Yes | Yes |
| Tests | No | Yes | Yes (it breaks, so it gets fixed) |
| Comments | No | No | **No** (nobody forces you to fix them) |

---

## III. Eliminating the Need for Synchronization

The solution is not "maintain comments harder," but to **eliminate the need for synchronization**. There are two paths to doing so:

### Path One: Code Describes Itself

Type systems, tests, naming, structure — these tools are **naturally synchronized with the code**, because they are themselves part of the code. Change the code, and if you don't change these too, it won't compile or the tests break.

### Path Two: Comments Describe Not Behavior, but What Code Cannot Express

Describing behavior (the what) is the job of code + types + tests. Comments carry only information that code **structurally cannot express** — but even for that information, the best carrier is not an inline comment, but a git commit message / ADR / wiki.

The following sections expand each of these two paths.

---

## IV. Code Describes Itself

### 4.1 Type System > Comments

The most common use of comments is to "explain what this function returns" or "what type this parameter is." But the type system does the same job, and it is **enforced by the compiler, so it never goes stale**.

| Tier | Language/Tool | Ability to replace comments | Notes |
|------|----------|------------|------|
| Strong | **Rust** | Near-total replacement | Ownership, lifetimes, and algebraic data types make it impossible to misuse parameter semantics |
| Next strongest | Swift / Kotlin / TypeScript | Largely replacement | Null safety, sealed classes, and generic constraints cover 80% of comment demand |
| Usable | Python with type hints | Partial replacement | `mypy`/`pyright` static checks; not enforced at runtime, but effective at development time |
| Weak | Go / PHP | Almost no replacement | Go lacks algebraic types and PHP's type system is weak — **these two languages have the highest demand for comments, and they are precisely the languages with the worst comment culture** |

**Causality**: it is not that "code is clear because comments were written," but that "comments are forced because the type system is weak." Pick the right language, and the need for comments disappears on its own.

### 4.2 Tests > Comments

The type system constrains **the shape of data**; tests constrain **the contract of behavior**. Both share the same idea: let the machine perform verification automatically, rather than relying on a human reading comments.

**TDD** (test-driven development): the same philosophy as the type system — hand the constraints to the machine. A test case is an "executable comment" — it not only describes behavior, it verifies behavior.

```python
# Comment: raises an error when the user's balance is insufficient (may go stale)
# Test: never goes stale
def test_raises_error_when_balance_insufficient():
    account = Account(balance=0)
    with pytest.raises(InsufficientBalanceError):
        account.transfer(to="bob", amount=100)
```

**BDD** (behavior-driven development): describing things in `Given/When/Then` natural language is clearer than TDD function names. But the price is **a thicker intermediate layer** — Cucumber/Gherkin needs glue code to map natural language onto test logic.

| Dimension | TDD | BDD |
|------|-----|-----|
| Clarity of description | Medium (function names) | High (natural-language scenarios) |
| Intermediate-layer thickness | Thin (write the test directly) | Thick (Gherkin + glue code) |
| Maintenance cost | Low | Medium-high (two layers to keep in sync) |
| Applicable scenarios | Unit/integration tests | Acceptance tests for non-technical stakeholders |

**The pragmatic choice**: TDD is sufficient in the vast majority of scenarios. BDD is only worth it when product managers or business stakeholders need to read the tests directly.

### 4.3 Naming and Structure > Explanatory Comments

```python
# Bad style (requires a comment)
# Iterate over the user list, filter out active users, take the first 10
result = []
for u in users:
    if u.active:
        result.append(u)
        if len(result) >= 10:
            break

# Good style (self-explanatory code)
active_users = [u for u in users if u.active][:10]
```

The variable name `active_users` is itself the comment. The structure of the list comprehension is itself the logic. If you need to write a comment to explain what a piece of code is doing, then that code has a problem with its naming or its structure.

**Principles of self-explanatory code**:
- Function name = action + object (`calculate_total_price`, not `calc`)
- Variable name = what it is (`active_user_count`, not `count`)
- Boolean variable = a question (`is_valid`, not `flag`)
- Extracting a subfunction = a substitute for a comment

### 4.4 Programmers Are Not Apprentices

The most common form of comment is **explaining what the code is doing**:

```python
# Iterate over the user list
for user in users:
    # Check whether the user is active
    if user.is_active:
        # Add to the result list
        result.append(user)
```

This is **an insult to the code and to the reader**. The reader is a programmer, not an apprentice. They can read what `for user in users` is doing.

**Two modern reasons make "explanatory comments" even more superfluous**:

1. **Code should be self-explanatory**: good naming + clear structure = no comments needed. If you can't write it that way, the code needs refactoring, not comments.
2. **AI-assisted comprehension**: a developer in 2026 can select any code snippet and have AI explain what it is doing. The "translation" function of comments has already been replaced by AI.

### 4.5 Don't Write Regex, Use a Parser

Some say "regex is too hard to read, comments are the only way out" — but the correct answer is **don't write regex**.

Use a parser instead:
- **Generators**: `tree-sitter`, `ANTLR`, `lalrpop`
- **Combinators**: `nom` (Rust), `pest` (Rust), `combine` (Rust), `parsimonious` (Python)

A parser produces typed, debuggable, testable structured data; a regex produces fragile string matching. If a regex is complex enough to need a comment, it is already complex enough to deserve replacement by a parser.

---

## V. Comments Describe Only What Code Cannot Express

Type systems, tests, and naming solve the problem of "describing behavior." But some information **structurally cannot be expressed by code** — this is the legitimate use of comments.

### 5.1 Contracts (Preconditions / Invariants)

```rust
/// Precondition: the caller must guarantee `amount > 0` and `from != to`
/// Invariant: after the call, the change in `self.balance` = -amount
fn transfer(&mut self, to: &Account, amount: u64) { ... }
```

The type system can express "this parameter is a u64," but it cannot express "this parameter must be greater than 0." What a contract comment adds is a **value-domain constraint**.

### 5.2 Design Intent

```rust
// Use a B-tree rather than a HashMap: above 100K entries, the B-tree's cache locality
// is markedly better than the HashMap's random memory access (measured 3x faster)
```

This is not explaining what the code is doing (B-tree code is itself a B-tree), but explaining **why this data structure was chosen**.

### 5.3 Decision Exclusions (Why Not)

```rust
// Chose Raft over Paxos: Raft's leader election is simpler,
// and our scenario does not need Paxos's flexible leader switching.
// Why not use etcd's Raft directly? Because we only need log replication,
// not the KV storage layer.
```

### 5.4 Friction Markers (HACK / FIXME)

```rust
// HACK: delete this workaround once upstream fixes #1234
// FIXME: this should use atomic, but the current crate version has a bug
// XXX: temporary solution, replace during the Q3 refactor
```

The value of this kind of comment lies in **traceability** — paired with an issue tracker, it forms a closed loop.

### 5.5 But Even for This Information, the Best Carrier Is Not an Inline Comment

| Information type | Inline comment (suboptimal) | Best carrier |
|---------|----------------|---------|
| Contract | Comment at the function head | Rust doc comment / type constraint (`where T: Positive`) |
| Design intent | Next to the code | ADR / wiki (full context available) |
| Decision exclusions | Next to the code | git commit message + ADR |
| Friction markers | `// HACK` | Issue tracker (has tracking, an owner, and an expiry mechanism) |

**Principle**: an inline comment is a **degraded carrier** for this information. If it can go in a commit message / ADR / issue tracker, it should not go in the code — because inline comments have no tracking mechanism, while those external tools do.

---

## VI. Conclusion

```
Comments lie → synchronization is hard → eliminate the need for synchronization
                  │
  ┌───────────────┴───────────────┐
  ↓                               ↓
  Code describes itself           Comments describe what code cannot express
  · Type system                   · Contracts (preconditions / invariants)
  · Tests (TDD/BDD)               · Design intent
  · Naming and structure          · Decision exclusions (Why Not)
  · Parsers instead of regex      · Friction markers (HACK)
  · AI-assisted comprehension     ↓
                                  Best carrier: commit message / ADR / Issue
                                  Inline comments are a degraded carrier
```

> **Default stance: don't write comments, unless you can prove a comment is irreplaceable.**