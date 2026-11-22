# Safety and Validation

This chapter covers how codec-lang draws the line between checked behavior and undefined
behavior, and how its formal validator proves that undefined behavior cannot actually occur.
The semantics follow **C** — undefined behavior is real and exploitable by the optimizer —
while the surface syntax stays **Python-inspired**. Every choice is weighed against the
codec-development context: hot loops must not pay for checks they did not ask for, yet a codec
that is validated once should be provably free of memory violations. Settled decisions come
first; the unresolved questions are gathered in the "Open Questions" section at the end. The
overflow-specifier *mechanism* referenced here is defined in `operators-and-expressions.md` and
`type-system.md`; the undefined-behavior facts about pointers and the address space live in
`memory-model.md`; the `alt:` construct itself is settled in `control-flow.md`.

## Checked vs. undefined behavior

codec-lang does not force one global safety mode. Instead, at each risky operation the author
makes an explicit, local choice between "fast and undefined if I am wrong" and "checked with a
defined fallback", and that choice is spelled with the same overflow specifiers used elsewhere
in the language.

- **[Decided]** Bounds checking on array and vector access is an explicit choice expressed
  through the overflow specifier (resolution #4). With **no specifier**, an out-of-bounds access
  is undefined behavior in release mode and is checked by the formal validator in validation
  mode; with the wrapping specifier `%` (e.g. `v%[i]`), the index is wrapped to the array
  bounds; with the saturation specifier `|` (e.g. `v|[i]`), an out-of-bounds access returns `0`.
  This reconciles "no runtime bounds checking — that is the validator's task" with "the author
  makes an explicit checked-or-UB choice": the choice is the specifier. The specifier mechanism
  and where each form is legal are settled in `operators-and-expressions.md` and
  `type-system.md`.
- **[Decided]** Integer overflow is an explicit checked-vs-UB choice for **both signed and
  unsigned** integers, made with the same `|` (saturation) and `%` (wrapping) specifiers; without
  a specifier, overflow is undefined behavior. See `operators-and-expressions.md` for the operator
  syntax. **(differs from C)** — C defines unsigned overflow as modular wraparound and leaves only
  *signed* overflow undefined; codec-lang treats unsigned overflow as undefined too unless the
  author opts into wrapping or saturation, so that a wrap in a hot loop is always visible in the
  source rather than assumed.

## Undefined behavior catalog

- **[Decided]** The baseline rule is "**same as C unless otherwise specified**": constructs that
  are undefined in C stay undefined here unless a decision elsewhere gives them defined behavior
  (for example, out-of-bounds *pointer* creation and comparison are well-defined under the
  circular address space — see `memory-model.md` — even though out-of-bounds *access* is not).
- **[Decided]** The following are undefined behavior: null-pointer dereference; out-of-bounds
  array access (absent a `%`/`|` specifier); use-after-free; double free; reading uninitialized
  memory or an uninitialized variable; reinterpreting a value at a different type / type punning
  through incompatible pointer casts; and a data race. **(differs from C)** — the list itself
  overlaps heavily with C, but codec-lang narrows some cases (unsigned overflow is UB by default,
  see above) and widens others (cross-object pointer *comparison* is defined here, see
  `memory-model.md`), so the catalog is stated explicitly rather than inherited wholesale.
- **[Decided]** Padding bytes are **not** covered by the uninitialized-read rule: their values are
  undefined at initialization and when accessed, but reading them is not undefined behavior
  because it cannot crash (settled in `memory-model.md`).

## Formal validation

The escape hatch from all of the above is that undefined behavior can be *proven absent* rather
than merely avoided by convention. Validation is a distinct build mode alongside release and
debug (the same mode that turns bounds checking and `@assert` into real checks).

- **[Decided]** Formal validation is available to prove that no undefined behavior happens. The
  unit of validation is usually a function: the author defines **preconditions** on the inputs
  that the validator may assume hold, the validator then proves that **no undefined behavior can
  occur** under those preconditions, it checks the **equivalence of all `alt:` paths** (see
  `control-flow.md`), and it checks **postconditions** on the outputs.
- **[Decided]** Contracts between functions are specified manually: because callers and callees
  are validated as separate units, the preconditions and postconditions are the interface the
  validator relies on across a call boundary. Directives that participate in validation include
  `@assert{...}` (checked in validation mode), `@assume{...}` (a validation-mode hint that prunes
  search paths), and `@unreachable` (checked in validation mode); all are settled in
  `control-flow.md` and `functions.md`.

## Contrast with Rust

- **[Decided]** codec-lang takes a **different approach from Rust**. It gives the programmer more
  freedom — for example, two mutable pointers to the same object are allowed, and there is no
  ownership, borrowing, or lifetime tracking (see `memory-model.md`) — but in exchange the
  programmer must **manually specify the inter-function contracts** the validator needs. Rust
  proves memory safety structurally from ownership rules that hold by default; codec-lang proves
  it from contracts the author writes down, trading Rust's automatic guarantees for the freedom
  to alias and the ability to express codec data-flow patterns that a borrow checker would reject.
  (contrast with Rust)

## Open Questions

- **[Open]** The `alt:` equivalence check hinges on "same effects outside their scope" (its home
  is `control-flow.md`), but the precise definition and its limits are open for codec work: does
  "external effect" cover only memory writes and returned values, or also observable timing,
  volatile accesses, and trapping/UB reachability? Two `alt:` blocks that differ only in a
  scratch buffer or in which SIMD lanes are touched transiently should count as equivalent — the
  rule that makes that so, and the point at which two hand-optimized kernels are declared
  *inequivalent*, needs to be pinned down. Cross-reference `control-flow.md`.
- **[Open]** The contrast with Rust needs a concrete safety-guarantee ledger: exactly what is
  *given up* by dropping ownership/borrowing (no automatic use-after-free or aliasing protection;
  every such property now rides on author-written contracts) versus what is *gained* (two mutable
  pointers to one object, cross-object pointer comparison, freedom from lifetime annotations).
  For a codec author the open question is which bugs the validator still catches without contracts
  and which silently become the author's responsibility. Cross-reference `memory-model.md`.
- **[Open]** UB catalog completeness is explicitly flagged in the decision log as needing a
  "comprehensive UB catalog", and the current list may not be complete. Codec-relevant cases that
  are not yet classified include unaligned access via `a~[i]` when it fragments, division by zero
  in vector lanes, reads of padding bytes versus genuinely uninitialized fields, shift amounts at
  or beyond the type width without a specifier, and type punning through unions versus through
  pointer casts. Whether each is UB, defined, or checked needs to be enumerated. Cross-reference
  the "Don't forget" reminders in `open-questions.md` and the UB facts in `memory-model.md`.
- **[Open]** How to **formally validate that the stack and heap do not run out of space** is an
  open problem the validator does not yet address: register values may spill silently to stack or
  heap (see `memory-model.md`), heap allocation via `@alloc` can fail and return `null`, and a
  codec's working-set size is data-dependent. What preconditions describe available space, and how
  the validator reasons about spill and allocation failure, is unresolved. This is tracked as a
  cross-cutting reminder in `open-questions.md`.
- **[Open]** The **contract specification language** the validator consumes is underspecified.
  Preconditions and postconditions are written today with `@assert`/`@assume`, but it is unclear
  how to state the richer facts a codec needs — pointer non-aliasing over a region, buffer length
  and stride relationships, "this index stays in bounds for the whole loop", value ranges that
  justify omitting an overflow specifier — and whether that vocabulary is just Boolean assertions
  over existing expressions or a dedicated contract syntax. Cross-reference `control-flow.md`.
- **[Open]** The interaction of validation mode with `@assert` and `@assume` (both settled in
  `control-flow.md`) is a likely confusion point. `@assume` is *trusted* — an unsound assumption
  lets the validator "prove" a false program safe — whereas `@assert` is *verified*. The rules for
  which one the validator may lean on, whether an `@assume` must itself be discharged somewhere, and
  how a failed proof reports back to the author (counterexample input? offending path?) are open,
  and they matter because a wrong `@assume` reintroduces exactly the UB the mode is meant to rule
  out. Cross-reference `control-flow.md`.
