# Builtins and Directives

This chapter catalogs the `@`-directive/attribute *system* itself and the compiler builtins, rather
than re-deciding each individual directive — every `@`-form is settled in the chapter that owns the
feature it configures, and this chapter cross-references those chapters by file name. As everywhere
in the language, *semantics* follow **C** (builtins map to concrete machine operations, no hidden
cost) and the *surface syntax* is **Python-inspired** (a leading `@` prefix, an indented block where
a directive heads a construct). Every choice below is weighed against the codec-development context:
predictable cost, hot loops, SIMD, and tight C interop. Settled decisions come first; the unresolved
decisions, ambiguities, and likely confusions are gathered in the "Open Questions" section at the
end.

## The directive/attribute prefix

- **[Decided]** Attributes and compiler directives are written with a leading `@` prefix. For
  example, `@constexpr expression` requires that the expression be resolved at compile time. The `@`
  makes directives lexically distinct from identifiers and keywords so they never collide with the
  user's names.
- **[Decided]** The prefix form is `@attr x` — the directive binds to the target that follows it, as
  a prefix operator. In the precedence table (see `operators-and-expressions.md`) `@attr x` sits with
  the other prefixes (`+x`, `-x`, `~x`, `!x`, `..x`, `..?x`), an "attribute before an identifier".
- **[Decided]** A directive may configure a declaration, a definition, a type, a parameter, a call
  site, or a statement, depending on which directive it is; the owning chapter for each `@`-form
  states exactly what that form attaches to.

## Builtin bit-counting functions

- **[Decided]** The language provides builtin bit-counting functions: `popcount` (population count,
  the number of set bits), `ctz` (count trailing zeros), and `clz` (count leading zeros). These are
  compiler builtins rather than library calls so they lower directly to the target's native
  instructions — exactly the predictable, single-instruction cost a codec hot loop wants. They also
  appear in `operators-and-expressions.md` alongside the bitwise operators, but their naming and home
  belong here with the rest of the builtin surface.

## The `@`-vocabulary

- **[Decided]** The following `@`-forms appear across the language. Each one is **Decided in its
  owning chapter** — this chapter only records that they share the `@`-directive system and points to
  where each is specified. Cross-references use the target file name.
  - `@constexpr` — force compile-time evaluation of an expression (this chapter; see the prefix
    section above).
  - `@mut` — make a variable mutable (immutable is the default). See `variables-and-bindings.md`.
  - `@stack` — give a variable stack storage / block scope. See `variables-and-bindings.md`.
  - `@alloc` — used both as a storage-class attribute (`@alloc a`, heap/global scope) and as a
    heap-allocation expression (`p = @alloc T{...}`). See `variables-and-bindings.md` and
    `memory-model.md`; the overloaded meaning is a cross-cutting open question tracked in
    `open-questions.md` (noted again under Open Questions below).
  - `@extern` — export/import with the host C calling convention and C-compatible struct layout;
    prevents field reordering. See `functions.md`, `memory-model.md`, and `modules-and-interop.md`.
  - `@packed` — pack a struct like C (member default alignment becomes 1). See `memory-model.md` and
    `type-system.md`.
  - `@aligned{N}` — force an alignment (including *decreasing* it, contrary to GCC) on a type,
    struct, field, or object storage. See `type-system.md` and `memory-model.md`.
  - `@order{n}` — pin a struct field to a position, overriding automatic reordering. See
    `type-system.md`.
  - `@restrict` — assert a pointer will not alias any other. See `type-system.md`.
  - `@noalias` — assert a pointer aliases no other. See `memory-model.md`.
  - `@overload` — allow same-name functions distinguished by argument type. See `functions.md`.
  - `@ref` — pass a parameter by reference/variable (custom convention only). See `functions.md`.
  - `@function` — reference a function as a first-class pointer value (`f = @function name`). See
    `functions.md`.
  - `@noinline` / `@alwaysinline` — control inlining on a definition or a call site. See
    `functions.md`.
  - `@tailcall` — warn if a call cannot become a tail call. See `functions.md`.
  - `@hot` / `@cold` — hint expected execution frequency. See `functions.md`.
  - `@passonce` — turn a loop into a do-while (the body runs at least once). See `control-flow.md`.
  - `@assert{condition, message}` — mode-dependent runtime/validation check. See `control-flow.md`.
  - `@assume{condition}` — validation-mode hint to prune search paths. See `control-flow.md`.
  - `@unreachable` — assert a branch or function end is never reached (counts as an assert). See
    `control-flow.md` and `functions.md`.

## Inline assembly and borrowed builtins

- **[Decided]** `asm(...)` inserts inline assembly and uses the **same syntax as GCC**. This lets a
  hot loop drop to hand-written SIMD without inventing a new assembly dialect. (borrowed from
  GCC/LLVM)
- **[Decided]** Compiler builtins borrowed from GCC/LLVM are adopted where confirmed and carry
  `[Decided]` once settled. The bit-counting builtins above are the confirmed set today; the broader
  survey of which GCC/LLVM builtins to adopt, rename, or omit is still open (see Open Questions and
  the cross-cutting reminder in `open-questions.md`). (borrowed from GCC/LLVM)

## Deferred directives and features

- **[Future]** `@minaligned` — a minimum-alignment attribute may be added if a need appears (see
  `type-system.md`).
- **[Future]** `@readonly` / `@writeonly` — directives to express read-only/write-only pointed values
  may be added later (today only the variable binding is mutable/immutable, not the pointed value;
  see `type-system.md`).
- **[Future]** An `@inline` **ordering** hint may be added to tell the compiler in which order to
  attempt inlining calls (for example, ordering by decreasing impact when inlined; see
  `functions.md`).
- **[Future]** `@callconv{"..."}` — an attribute to select calling conventions other than the custom
  and C ones may be added later (see `functions.md` and `modules-and-interop.md`).
- **[Future]** Debugging / printing features are not specified yet and may be added later.

## Open Questions

- **[Open]** The `@`-prefix is reused for three conceptually different things — plain directives
  (`@constexpr`, `@tailcall`, `@assert`), storage-class attributes (`@stack`, `@mut`, `@alloc a`), and
  even an allocation *expression* (`p = @alloc T{...}`) — and it is unresolved exactly which category
  each `@`-form falls into and how each parses in the grammar (a prefix statement modifier, a
  declaration qualifier, or an expression head). `@alloc` is the sharp case because it is both a
  storage class and an allocation expression; that overloaded meaning is a cross-cutting question
  tracked in full in `open-questions.md` (this chapter only notes its directive-parsing facet). A
  single, uniform parse rule for `@attr` matters so codec authors can predict where a directive is
  legal and what it binds to.
- **[Open]** Builtin and intrinsic **naming conventions and namespace collisions** are unsettled:
  `popcount`, `ctz`, and `clz` are ordinary lowercase names that live in the same lexical space as
  user identifiers and library functions, so it is unclear whether builtins occupy a reserved
  namespace, whether a user may shadow or redefine one, and how a name clash between a builtin and an
  imported symbol is resolved. A codec codebase leans heavily on these bit primitives, so the rule
  needs to be unambiguous.
- **[Open]** There is a likely **confusion between compiler intrinsics and ordinary library
  functions**: which operations are true builtins that lower to a single instruction (and are always
  available), versus which live in a library or a `import "compiler"` extension (see
  `modules-and-interop.md`), is not spelled out. A programmer expecting `popcount` to be a guaranteed
  single-instruction intrinsic needs to know when it is one and when it silently becomes a call — the
  distinction is exactly the kind of hidden cost the language tries to avoid.
- **[Open]** Which **GCC/LLVM builtins are adopted, renamed, or omitted**, and their exact C-facing
  behavior, is an open survey — only the bit-counting set is confirmed today. The decision log's
  "check all GCC extensions and builtins" reminder (tracked cross-cuttingly in `open-questions.md`)
  belongs here in spirit: for each borrowed builtin the naming, argument shape, overflow/undefined
  edge cases (for example `ctz`/`clz` on a zero input), and vector variants must be pinned down.
  (borrowed feature)
- **[Open]** The **allowed vs disallowed directive syntaxes** are unresolved: whether a directive
  must come strictly *before* its target or may follow it, which elements each directive may attach to
  (declaration, type, field, parameter, call, statement), which argument forms are legal (`@aligned{N}`
  with braces vs a bare directive), and whether multiple directives may be **stacked** on one target
  and in what order. This overlaps the cross-cutting "should attributes come before or after their
  target, and which elements should they apply to?" reminder in `open-questions.md`. (allowed/disallowed
  syntax gap)
- **[Open]** Directives to **assert tail-call reduction and instruction-count thresholds** are only
  partially answered: `@tailcall` (see `functions.md`) already warns when a call cannot become a tail
  call, but a directive to assert that a function body stays below a given instruction count — a
  predictable-cost guarantee valuable for codec kernels — is not specified. How such a threshold would
  be expressed, measured, and enforced across optimization levels is open. (gap)
- **[Open]** A **struct-field assert specifier** — a directive attached to a field to assert an
  invariant about it (analogous to `@assert` for statements) — is called for but unspecified: where it
  attaches, when it is checked (init, every write, validation only), and how it interacts with the
  automatic field reordering in `memory-model.md` all need deciding. (gap; cross-ref `type-system.md`)
- **[Open]** **Debugging and printing** support has no home yet: whether inspection/printing is a
  builtin, a `import "compiler"` extension (see `modules-and-interop.md`), or a library, and how it
  behaves across release/debug/validation modes, is undecided. For codec work the cost and
  availability of a debug-print in a hot loop matters. (gap)
