# Variables and Bindings

This chapter covers how variables are declared, bound, mutated, scoped, and stored in
codec-lang. The semantic model follows **C**; the surface syntax is **Python-inspired**.
Because codec work depends on predictable cost, hot loops, SIMD, and tight C interop, the
binding and storage rules are designed so that where a value lives (register, stack, heap)
and whether it can change are visible in the source rather than inferred silently.

## Declaration and initialization

- **[Decided]** `a = 0` is both declaration and initialization; a prefix type is
  forbidden in this form (the type is inferred from the initializer). See
  `type-system.md` for the inference rules.
- **[Decided]** `a = i8{*}` is declaration without initialization; here the type is
  mandatory and `{*}` marks the value as intentionally uninitialized (undefined value).
- **[Decided]** The assignment operator is `=`, and assignments are statements, not
  expressions — there are no assignments with side effects embedded in larger expressions.
  Combined operation-and-assignment is expressed by pairing an arithmetic operator with
  `=` (see `operators-and-expressions.md`). **(differs from C)** — in C, `=` is an
  expression that yields a value, so constructs like `while ((c = getchar()) != EOF)` are
  legal; codec-lang forbids assignment inside an expression.

## Mutability

- **[Decided]** Variables are immutable by default. To make a variable mutable, declare it
  `@mut a`. Requiring `@mut` also disambiguates a declaration from a later reassignment,
  since a bare `a = ...` that is not the first binding would otherwise be indistinguishable
  from a mutation. **(differs from C)** — C variables are mutable by default and have no
  keyword requirement to permit mutation; `const` is the opt-in, whereas here mutability is
  the opt-in.
- **[Decided]** Mutability applies to the variable binding only, not to the value behind a
  pointer. Const-ness of pointed-to values is not modeled today (see `type-system.md` for
  the `@readonly`/`@writeonly` direction). **(differs from C)** — C attaches `const` to the
  pointed-to type via the declarator (`const T*` vs `T* const`).

## Redefinition, shadowing, and name reuse

- **[Decided]** Redefining an immutable variable in the same scope is forbidden, and this
  explicitly includes variable shadowing. **(differs from C)** — C permits an inner scope
  to shadow an outer variable of the same name.
- **[Decided]** A new variable may not reuse the name of a variable that is out of scope
  only because of a function boundary but is still within the same block scope. This rule
  exists specifically to avoid the C-style confusion where a name silently refers to a
  different entity across a call boundary. **(differs from C)** — C has no such
  cross-function-boundary name-reuse restriction within a block.

## Storage classes and scope

- **[Decided]** The default storage/scope model is block scope, but a variable's lifetime
  stops at function calls unless it is declared `@stack` or `@alloc`. Conceptually this is
  register scope on a machine with an infinite number of registers: values live in
  (virtual) registers by default and do not automatically survive across a call.
  **(differs from C)** — C has no register-by-default value model; C automatic variables
  live on the stack and their addresses are always available and stable across calls.
- **[Decided]** `@stack` places a variable in stack storage so it has a stable address and
  survives across calls; `@alloc` places it in heap-backed storage. These storage classes
  are also what make a variable's address usable in the ways `type-system.md` and
  `memory-model.md` describe (for example, indexing a vector with a runtime variable or
  taking the address of a vector to cast it to a base-type pointer requires `@stack` or
  `@alloc`, or inheritance of that property from a containing struct).
- **[Decided]** A variable defined in global scope is a global variable.

Note on `@alloc`: the token `@alloc` is used both as a storage-class attribute here
(`@alloc a`) and as a heap-allocation expression elsewhere (`p = @alloc T{...}`, see
`memory-model.md`). Whether to keep that dual meaning or rename one use is an unresolved
cross-cutting question tracked in `open-questions.md`; it is not repeated in full here.

## Multiple assignment

- **[Decided]** Comma-separated multiple assignment is supported: `a,b,c = 0,1,2`. This
  enables swap without a temporary (`a,b = b,a`), capture of a function's multiple return
  values, and combined op-assign across several targets (`a,b += 1,2`). The Python-shaped
  surface makes these read naturally while the underlying semantics remain C-like value
  assignments.

## Open Questions

- **[Open]** Immutability-by-default plus the `@mut` opt-in is the most likely early
  stumbling block for C programmers, who expect a bare declaration to be mutable and expect
  `const` to be the exception. The mental-model break should be surfaced prominently; open
  question is how aggressively the compiler should diagnose an attempted mutation of a
  non-`@mut` binding, and whether the error message should suggest adding `@mut`.
  **(differs from C)** — mutability is opt-in here, the reverse of C's `const` opt-in.
- **[Open]** The rule that a bare `a = ...` doubles as both "declare-and-initialize" and,
  for a `@mut` variable, "reassign" leaves an ambiguity: within a single scope, is the
  first `a = ...` always the declaration and every later `a = ...` always a mutation (which
  requires the first to have been `@mut`), or are there positions where `a = ...` is
  rejected outright? The exact diagnostic boundary between "redeclaration" and "assignment"
  needs to be pinned down.
- **[Open]** The no-shadowing rule interacts subtly with the same-block / function-boundary
  name-reuse restriction. It is not fully specified what counts as "the same scope" for
  redefinition purposes versus "the same block scope" for the cross-function-boundary reuse
  rule, especially across nested blocks (loop bodies, `if`/`elif`/`else` arms, `switch`
  cases). C programmers relying on inner-scope shadowing will need the precise boundary.
- **[Open]** Interaction between the no-shadowing / name-reuse rules and ephemeral
  (`_`-prefixed) identifiers is unclear. Ephemeral identifiers carry across following
  statements and branches until the next assignment (see `lexical-structure.md`,
  resolution #5); it is unresolved whether an ephemeral name is exempt from the
  no-redefinition rule (since reuse is its whole point), whether an ephemeral name can
  collide with a regular binding of the same spelling, and how the two scoping regimes
  compose inside nested blocks.
- **[Open]** Storage-class promotion visibility: the model says a value lives in a
  (virtual) register by default and only lands on the stack or heap when declared `@stack`
  or `@alloc`. When does the compiler *silently* promote a register-scoped variable to
  stack or heap storage — for example when its address is implicitly needed, when it must
  survive a call, or when it is indexed by a runtime value — and how visible is that
  promotion to the programmer? For codec hot loops the register-vs-spill decision is
  performance-critical, so whether promotion is a warning, an error, or a silent action is
  an open decision.
- **[Open]** It is unresolved whether `@stack` and `@alloc` can be combined with `@mut`
  freely, and what the ordering/stacking rules for these attributes are (attribute
  placement and stacking is itself a cross-cutting directive question — see
  `builtins-and-directives.md`).
- **[Open]** Allowed vs disallowed declaration syntaxes need tightening: `a = 0` forbids a
  prefix type while `a = i8{*}` requires one, but the status of forms like `i8 a = 0`
  (C-style prefix-typed initialization) or a prefix type combined with an explicit cast
  initializer is not stated. C programmers will reach for prefix-typed declarations by
  reflex, so which of these are rejected outright should be documented.
- **[Open]** The `{*}` uninitialized marker in `a = i8{*}` raises questions for codec
  safety work: is reading an uninitialized `@stack`/`@alloc` variable undefined behavior,
  and does the formal validator (see `safety-and-validation.md`) track
  definitely-initialized state before first use?
- **[Open]** Global-variable semantics are only lightly specified: initialization order
  across globals, whether globals are implicitly immutable or require `@mut`, and how a
  global's storage relates to the `@stack`/`@alloc`/register model all need to be resolved.
- **[Open]** The `@alloc` token is overloaded as both a storage-class attribute (`@alloc a`)
  and a heap-allocation expression (`p = @alloc T{...}`). The full cross-cutting question —
  whether to keep the dual meaning or rename one use — lives in `open-questions.md` because
  it spans this chapter, `memory-model.md`, and `builtins-and-directives.md`; it is only
  cross-referenced here.
- **[Future]** The ability to back a variable with a custom pointer may be added later,
  letting a binding be stored through a caller-supplied pointer rather than in a
  compiler-chosen register/stack/heap slot.
