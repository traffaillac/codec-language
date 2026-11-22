# Control Flow

Control flow follows **C** semantics on a **Python-inspired** surface, and every construct is
shaped by the codec-development context: predictable cost, hot loops, SIMD, and tight C interop.
The blended `for assignment() while condition():` loop, the branch-table `switch`, the `alt:`
alternate blocks, and the `@assert`/`@assume` contract directives all exist to make the cost of a
loop or a dispatch explicit rather than hidden. Settled decisions come first; the unresolved
decisions, edge cases, and likely confusions are gathered in the "Open Questions" section at the
end. Overflow and comparison semantics referenced here live in `operators-and-expressions.md` and
`type-system.md`; the ephemeral `_`-identifier scope rule referenced throughout is settled in
`lexical-structure.md`; the formal validator that backs `@assert`/`@assume` and `alt:` equivalence
checking lives in `safety-and-validation.md`.

## Statements, not expressions

- **[Decided]** All control-flow blocks are **statements, not expressions** — an `if`, a loop, a
  `switch`, or an `alt:` block does not evaluate to a value. **(differs from C)** — C's `?:` is an
  expression and C lets assignments and comma expressions carry values through control flow;
  codec-lang keeps control flow value-less so that the only way to move a value is an explicit
  assignment (see `operators-and-expressions.md` for the non-short-circuit `if(cond, t, f)` ternary,
  which *is* an expression).

## Conditionals

- **[Decided]** An `if` block may fit on one line, `if condition: statement(); statement()`, but a
  second block such as `else:` must start on a new line. The conditional chain is `if` / `elif` /
  `else`.
- **[Decided]** `if` has **no assignment slot** (unlike `for`). To test the result of a computation,
  use an ephemeral assignment first: `_ = expression()` on its own line, then `if _ < threshold:`.
  The ephemeral `_` remains available to the `elif` conditions of the same chain (resolution #5; the
  full carry-over rule is settled in `lexical-structure.md`). **(differs from C)** — C allows a
  declaration or assignment inside the `if` controlling expression (`if ((n = f()) < t)`), which
  codec-lang forbids because assignments are statements, not expressions.

## Loops

- **[Decided]** The loop shape is `for assignment() while condition():`. Both the `for` clause and
  the `while` clause are **optional, but they cannot swap** — the assignment clause always precedes
  the condition clause. **(differs from C)** — C's `for (init; cond; step)` has three clauses with a
  distinct step slot; codec-lang folds the loop into an assignment clause plus a condition clause
  and has no separate step clause.
- **[Decided]** The supported forms are:
  - `while 1:` — infinite loop (the `for` clause omitted).
  - `for i=0..10:` — range loop over a slice (the `while` clause omitted).
  - `while valid:` — plain while loop.
  - `for i=0.. while a[i]:` — range with an open-ended `range` plus a stop condition.
  - `for _=0.. while a[_]:` — the ephemeral identifier carries from the assignment clause into the
    condition clause (resolution #5).
  - `for i,j=0..10,a..b:` — zip: comma-separated assignment iterates several indices in lock-step.
  - `for i=..vector:` — iterate over the indices of a vector.
  - `for i=..?vector:` — iterate over the indices of non-zero values (equivalently, the set bits of
    an integer).
- **[Decided]** `break [n]` and `continue [n]` break from, or continue, a given **number of
  enclosing loops** (default `1`). **(differs from C)** — C's `break`/`continue` always act on the
  single innermost loop and multi-level exit needs a `goto`.
- **[Decided]** `0..10`, `..vector`, and `..?vector` produce a **read-only internal iterator type**
  that can be read but not assigned. (Slice syntax and its overflow specifiers are settled in
  `type-system.md`.)
- **[Decided]** Loops support iterating **over bitmasks** and the **find-first-set SIMD pattern**
  directly (via `for i=..?vector:` over set bits / non-zero lanes), so common codec scan loops do
  not need hand-written bit tricks.
- **[Decided]** A **do-while** loop is obtained with the `@passonce` attribute on the loop, which
  guarantees the body runs at least once before the condition is tested. **(differs from C)** — C
  has a dedicated `do { } while (cond);` form; codec-lang expresses it as an attribute on the
  ordinary loop.
- **[Decided]** Loop variables have **block scope**, the same as `if`. A variable introduced in the
  `for` assignment clause is visible only within the loop.

## Branch table (`switch`)

- **[Decided]** `switch i:` is a **branch table**: it is at once a jump table on a single integer
  value and a region of code in which `goto` jumps are available — but **only within the table**.
  **(differs from C)** — C's `switch` falls through from one case into the next by default, and C's
  `goto`/labels are function-scoped rather than table-scoped.
  - **[Decided]** The table may contain **only cases and labels**; statements may appear only inside
    a case or a label.
  - **[Decided]** `case 0:` binds a target to a single value; `case 0..10,12:` binds a case to
    multiple values (a slice and/or a comma-separated list). `default:` is the target reached when
    no case matches.
  - **[Decided]** Variables declared in a case are **scoped to that case**; variables shared across
    cases must be declared outside the table.
  - **[Decided]** `goto case 0` jumps to a given case from inside the table. There is **no
    fallthrough** — to reach the next case you must write `goto case ...`. **(differs from C)** — C
    fills through unless a `break` stops it; codec-lang inverts the default so that reaching another
    case is always explicit.
  - **[Decided]** `switch:` with **no value** jumps straight to `default:`. `label:` and
    `goto label` provide custom jump targets, still block-scoped within the table.
  - **[Decided]** `break` does **not** affect a `switch` — it only breaks enclosing loops.
    **(differs from C)** — in C, `break` is exactly how you leave a `switch`; here it passes through
    the table to the nearest loop, so a `switch` inside a loop is left by falling off its cases, not
    by `break`.
  - **[Decided]** `@unreachable` placed in the `default:` statement asserts that **all possible
    cases are covered**; it behaves as an assert (ignored in release, checked at runtime in debug,
    checked in validation mode — see `@assert` below and `functions.md`).

## Returns

- **[Decided]** Functions can have **multiple returns** — a function may return several named or
  unnamed values, captured with comma-separated multiple assignment at the call site (see
  `functions.md` and `variables-and-bindings.md`).

## Contract directives: `@assert` and `@assume`

- **[Decided]** `@assert{condition, message}` (the message is optional) attaches a checked
  condition whose behavior is **mode-dependent**: not checked in release mode; checked at runtime in
  debug mode, displaying the message on failure; and checked in validation mode, displaying the
  message on failure. **(differs from C)** — C's `assert` is compiled in unless `NDEBUG` is defined
  and always aborts at runtime when it fails; codec-lang gives `@assert` three distinct
  release/debug/validation behaviors, and in validation mode it is a proof obligation rather than a
  runtime abort.
- **[Decided]** `@assume{condition}` attaches a condition that is **not checked** in release or
  debug mode and is used only as a hint to prune search paths in validation mode. **(differs from
  C)** — this mirrors a compiler builtin like `__builtin_assume` rather than the C standard
  `assert`; the condition is never verified at runtime, so an incorrect `@assume` is a silent
  correctness hazard outside validation mode.

## Alternate blocks (`alt:`)

- **[Decided]** Alternate blocks `alt: ... else: ... else: ...` express **multiple optimized
  versions** of the same code block. The compiler is free to choose the fastest version for each
  host target; in debug mode the chosen option is compared against the first `alt:` block at
  runtime; and in validation mode the compiler formally checks that **all blocks have the same
  effects outside their scopes** (the equivalence check is described in `safety-and-validation.md`).
  **(differs from C)** — C has no built-in notion of interchangeable optimized variants with a
  compiler-verified equivalence guarantee; the closest C practice is hand-written `#ifdef` variants
  with no equivalence checking.

## Deferred forms

- **[Future]** There is **no `defer`** (incompatible with the no-function-scope value model), **no
  context manager** (Python's `with`), and **no RAII** (C++). RAII or a context manager may be added
  in the future. **(differs from C)** — C likewise has no `defer`/RAII, so this is a deferred
  ergonomic gap rather than a C deviation, noted here because the alternatives will shape how
  cleanup and resource scoping are expressed.

## Open Questions

- **[Open]** The `for assignment() while condition():` loop is a novel blend for both C and Python
  readers, and the set of **valid clause combinations** is not fully pinned down. Which assignment
  clauses may pair with which conditions — e.g. can a zip assignment `for i,j=0..10,a..b:` combine
  with an explicit `while`, can `for i=..?vector:` (set-bit iteration) take a `while` guard, must an
  open-ended `for i=0..` always be paired with a `while` or may it stand alone as an infinite loop,
  and is `for:` with an empty assignment and empty condition legal? A codec author needs a clear
  allowed/disallowed matrix so hot-loop shapes do not accidentally fall into an unspecified corner.
- **[Open]** The interaction between **loop-variable block scope and the ephemeral `_` carry-over**
  (resolution #5) has subtle edges: when `for _=0.. while a[_]:` carries `_` into the condition, does
  `_` remain visible in the loop body, is it reset on each iteration or on the next assignment, and
  how does a loop-scoped `_` interact with an outer `_` established before the loop or with a
  following `if`/`elif` chain inside the body? Cross-reference `lexical-structure.md`, where the
  ephemeral scope rule is authoritative. This is a likely confusion point because C has no analogous
  carry-over identifier.
- **[Open]** `@assert` and `@assume` are likely to be **misread against C's `assert`**: a C
  programmer expects an assert to compile out under `NDEBUG` and otherwise abort, whereas `@assert`
  has three mode-dependent behaviors and `@assume` is *never* checked at runtime. The confusion is
  sharpest for `@assume`, where a wrong condition is undetectable outside validation mode and can
  license miscompilation in hot code. The exact release/debug/validation contract, and what the
  validator treats as provable vs. assumed, needs worked examples; cross-reference
  `safety-and-validation.md`.
- **[Open]** The `alt:` **equivalence / "same external effects" definition** is the central open
  question of this chapter: what counts as an *external effect* (memory writes through escaping
  pointers, observable I/O, effects on `@extern`/FFI state, timing?), how the validator proves two
  alternate blocks equivalent, and what happens when the debug-mode runtime comparison against the
  first `alt:` block disagrees. Because codec `alt:` variants typically differ only in SIMD strategy,
  the effect model must treat lane-order and intermediate-register differences as internal. This
  belongs with the formal validator; cross-reference `safety-and-validation.md`.
- **[Open]** `break`/`continue [n]` and the branch-table `switch` have an **unresolved
  interaction**: since `break` does not affect a `switch` but does count enclosing loops, it is
  ambiguous whether a `switch` region counts as a level for the `[n]` count of a `break`/`continue`
  issued inside a case, and whether `continue [n]` from inside a case targets the loops enclosing the
  `switch` as expected. The counting rule for `[n]` across mixed loop/`switch` nesting needs to be
  stated explicitly.
- **[Open]** **`goto`/label and `goto case` scoping** inside the branch table has edge cases to
  resolve: whether a `goto label`/`goto case` may jump forward *and* backward within the table,
  whether jumping into or across a case that declares case-scoped variables is allowed (and what
  happens to those variables' initialization), whether `goto case` to a multi-value case
  (`case 0..10,12:`) is permitted and which entry point it uses, and how `switch:` (jump straight to
  `default:`) composes with an explicit `goto default`. The table-scoped restriction is settled, but
  the legal jump graph within it is not.
- **[Open]** The **allowed vs. disallowed one-line and nesting syntaxes** need to be enumerated: an
  `if` body may fit on one line but a second block must start on a new line — does the same one-line
  rule extend to loop bodies, to a single-case `switch`, or to an `alt:` arm? Which control-flow
  constructs may legally nest on a shared line via `;`, and where is a newline mandatory? This is an
  allowed/disallowed-syntax gap that affects how terse a codec kernel can be written.
- **[Open]** `@passonce` (do-while) and the read-only loop iterators leave a **first-iteration
  semantics** question: for `@passonce` on a `for ... while ...:` loop, is the assignment clause run
  before the guaranteed first body execution, and does the `while` condition get skipped only on the
  first pass or is the whole `for` clause re-evaluated? For iterator forms (`..vector`, `..?vector`),
  what does `@passonce` mean when the iterator is empty? The at-least-once guarantee needs a precise
  definition against the blended loop shape.
- **[Future]** No `defer`, context manager, or RAII yet — cleanup and resource scoping have no
  dedicated construct, and the future choice among RAII, a `with`-style context manager, or nothing
  will materially change how codec resource lifetimes are expressed.
