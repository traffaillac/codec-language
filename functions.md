# Functions

Functions are the primary unit of code and the usual unit of formal validation. Their *semantics*
follow **C** — pass-by-value by default, explicit interfaces, no hidden allocation — while their
*surface syntax* is **Python-inspired**, with an indented block after a `:` header. Every choice
below is weighed against the codec-development context: predictable cost, hot loops, SIMD-friendly
signatures, and tight C interop. Settled decisions come first; the unresolved decisions, edge
cases, and likely confusions are gathered in the "Open Questions" section at the end. FFI and
`@extern` interop details are shared with `modules-and-interop.md`; pointer and calling-convention
semantics touch `memory-model.md`; the directive vocabulary itself is cataloged in
`builtins-and-directives.md`; promotion rules used by overload resolution live in `type-system.md`.

## Definition and signatures

- **[Decided]** A function is defined `name(T a, T b) -> (T c, T d):` with the body indented on the
  following lines (Python-shaped block after the `:`).
- **[Decided]** Return-value names (`c`, `d` above) are **optional and documentation-only** — they
  do not create bindings the caller can name.
- **[Decided]** The parentheses around the return type may be omitted when there is a single return
  value, and the `->` arrow may be omitted entirely for a function with zero return values (a
  procedure).
- **[Decided]** There are **no forward declarations**: each function is declared and defined at the
  same place, and functions may be called before they textually appear. **(differs from C)** — C
  requires a prior declaration (or prototype) before a call; codec-lang has a single source model
  (see `modules-and-interop.md`) where the whole module's function set is visible, so no separate
  prototype is needed.

## Function types and function pointers

- **[Decided]** The type of a function is spelled `(T, T) -> (T, T)`. Unlike a definition header,
  the arrow **and** the parentheses can *not* be omitted here — the type form is always fully
  written so it is unambiguous in a type position.
- **[Decided]** A function-pointer type can be named with an alias: `type MyFunc = (T, T) -> (T, T)`.
- **[Decided]** Every function is already a pointer; there is no separate "address-of a function"
  step (see the namespace rules below). Function pointers support only calling and equality
  testing — no indexing and no ordering — as settled in `memory-model.md`.

## Namespace and referencing functions

- **[Decided]** Functions have their **own namespace**, separate from variables.
- **[Decided]** To obtain a first-class reference (a function pointer value), write
  `f = @function name`.
- **[Decided]** Because every function value is already a pointer, taking its address with
  `&@function name` is an **error**. **(differs from C)** — C lets you write both `f` and `&f` for a
  function; codec-lang has exactly one form (`@function name`).

## Returns and multiple returns

- **[Decided]** `return a, b` returns one or several values, which must match the function
  signature; a bare `return` returns from a procedure and is optional at the end of one.
- **[Decided]** Functions can return **multiple values**. On capture, individual results may be
  ignored with `*`: `a, *, b, * = function(...)`.
- **[Decided]** A single result can be read by **compile-time index**: `a = function(...)[0]`. There
  is no slicing of the return list, because the returned values need not share a type.
- **[Decided]** **Assignment is the only way to capture all values** from a multiple-return
  function; a call cannot yet be expanded directly into the argument list of another call.
- **[Decided]** `@unreachable` placed at the end of a function asserts it never returns. It counts
  as an assert: ignored in release mode, checked at runtime in debug mode, and checked in validation
  mode (mirrors the `@assert`/`@assume` behavior in `control-flow.md`).

## Overloading

- **[Decided]** Two functions may share a name but differ in argument types **only if both carry the
  `@overload` specifier**.
- **[Decided]** On a call to an overloaded name, the **first compatible overload in definition
  order** is chosen, applying automatic promotion where needed. "Compatible" and "promotion" follow
  the authoritative promotion matrix in `type-system.md`.
- **[Decided]** There is **no automatic promotion among function pointers** — when a function
  pointer is passed or assigned, the parameter types *and their attributes* must match exactly.
  **(differs from C)** — C allows more implicit compatibility between function pointer types; here
  the match is strict so the calling convention and attributes stay predictable.

## Parameter passing

- **[Decided]** Arguments are passed **by value by default** — integers, pointers, structs, and
  vectors alike (a struct or vector argument is copied). This is the C model.
- **[Decided]** Variable-length things (currently only strings) **cannot be passed as-is**, by value
  or by reference, and must be lowered to pointers first.
- **[Decided]** A parameter marked `@ref` is passed **by reference / by variable**: on return, the
  last value of that parameter carries back to the caller's variable, and the variable's scope
  extends past the call regardless of whether it was `@stack` or `@alloc`. Conceptually the function
  receives the variable's value in a register and hands the new value back in the same register.
  **(differs from C)** — C has no built-in by-reference parameter; you pass a pointer explicitly.
- **[Decided]** At the call site the caller chooses: `name(b)` passes `b` by value (copying if
  necessary); `name(@ref b)` requests pass-by-reference and **fails to compile if the target
  parameter is not `@ref`**.
- **[Decided]** An expression that is **not assignable** (e.g. `name(4)`) can only be passed by
  value.
- **[Decided]** `@ref` arguments have **block scope inside the function**.
- **[Decided]** `@ref` arguments are supported **only with the internal custom calling convention**
  (see below), not with the C convention.

## Calling conventions and `@extern`

- **[Decided]** By default the language uses a **custom optimized calling convention**.
- **[Decided]** The `@extern` function attribute switches a function to the **host default C calling
  convention** so it can interoperate with C. Because `@ref` relies on the custom convention, `@ref`
  parameters cannot be used on an `@extern` function. See `modules-and-interop.md` for the interop
  side (opaque-pointer export and C-compatible layout).

## Inlining

- **[Decided]** `@noinline` and `@alwaysinline` may be applied to a **function definition** (setting
  the default inlining strategy for that function) or to an **individual call** (to prevent or
  require inlining at that site).
- **[Decided]** `@alwaysinline` **fails** when inlining is definitely impossible — for example a
  call through a function pointer, a recursive function, or a library function — and it **warns**
  when it detects that LLVM has reintroduced subfunctions after an inline.
- **[Decided]** Inlining works **across modules** for functions defined inside modules.
- **[Decided]** `@hot` and `@cold` are hints to the compiler about a function's expected execution
  frequency.

## Tail calls

- **[Decided]** `@tailcall` placed on a statement containing a function call **warns** if the final
  call cannot be converted into a tail call — a predictable-cost guarantee useful for codec state
  machines and loops expressed as recursion.

## Inline assembly

- **[Decided]** `asm(...)` inserts inline assembly and uses the **same syntax as GCC**. (borrowed
  from GCC)

## Variadics and default parameters

- **[Decided]** Creating variadic functions is **not supported** — variadic C functions can be
  *called*, but you cannot define one. **(differs from C)** — C lets you define `...` variadic
  functions.
- **[Decided]** There are **no default parameter values** (deemed unnecessary).

## Generic blocks

- **[Decided]** A `generic` block defines a function once **per listed type**, and the generated
  functions automatically receive the `@overload` attribute. Single-type form:
  `generic T = i8x16, i16x8, i32x4, i64x2:` generates one function per `T`.
- **[Decided]** Multi-type iteration pairs the type parameters:
  `generic T, U = (i8, i32), (i16, i32), (i32, i32):` iterates three times, binding `(T, U)` to each
  tuple in turn.
- **[Decided]** `generic` blocks **cannot be nested**.
- **[Decided]** Example — a SIMD zip-low helper defined generically:

  ```
  generic T = i8x16, i16x8, i32x4, i64x2:
  	ziplo32(T a, T b) -> i32x4:
  		return (i32x4%{a} ++ i32x4%{b})[0, 4, 1, 5]
  ```

## Open Questions

- **[Open]** Overload resolution — "first compatible overload in definition order, with automatic
  promotion" — can produce surprising matches: a call that a C programmer expects to hit an exact
  overload may instead bind to an earlier one reachable by promotion, and reordering definitions
  silently changes which overload wins. The exact interaction with the promotion matrix (which
  promotions are considered, whether an exact match anywhere beats a promoted earlier match, and how
  ambiguity or "no compatible overload" is diagnosed) needs worked codec examples so hot-path
  dispatch stays predictable. Cross-reference `type-system.md`.
- **[Open]** `@alwaysinline` warns when it detects that LLVM has *reintroduced* subfunctions after
  inlining, but the trigger is underspecified: when exactly does that detection fire, what does the
  programmer do about it, and is the warning reliable across LLVM versions? For codec kernels that
  depend on inlining for predictable cost, a false negative (no warning while subfunctions exist) or
  a noisy false positive both matter. The precise contract of the warning is open.
- **[Open]** `asm(...)` borrows GCC inline-assembly syntax, but how much of GCC's behavior carries
  over is unresolved: which constraint letters, clobber lists, `volatile`/`goto` asm qualifiers,
  operand-numbering, and template-substitution rules are supported, and how do register operands
  interact with codec-lang's register-by-default value model and its custom calling convention?
  Nailing this down matters wherever a hot loop drops to hand-written SIMD. (borrowed from GCC)
- **[Open]** `@ref` and calling conventions interact in ways that need spelling out: `@ref` works
  only with the internal custom convention, so an `@extern` (C-convention) function cannot take
  `@ref` parameters, and it is unclear how a `@ref`-taking function is exposed to or wrapped for C,
  whether a function pointer to a `@ref` function can ever be passed across a C boundary, and how
  the "value returns in the same register" model composes with multiple `@ref` parameters plus
  multiple return values. Cross-reference `modules-and-interop.md` and `memory-model.md`.
- **[Open]** Generic-block limits leave gaps: blocks cannot be nested and there are no generics
  beyond them, so it is unclear how to express a function generic over *two independent* axes
  without enumerating every tuple, how a `generic` interacts with `@overload` written by hand for
  the same name, whether the type list can be computed or must be a literal list, and how a generic
  function is referenced with `@function name` when it expands into several overloads.
- **[Open]** Multiple-return capture has edge cases worth pinning down: whether the ignore marker
  `*` may drop *every* value, how a compile-time index `function(...)[0]` behaves when the index is
  out of range or when a `@ref` parameter also carries a value back, whether an ignored (`*`) return
  slot still forces the callee to compute it, and what happens when the capture arity does not match
  the signature. Because expanding a call into another call's arguments is not yet supported, the
  ergonomics of chaining multiple-return functions are also open.
- **[Open]** Non-returning and always-inlined functions raise a validation question: `@unreachable`
  at the end of a function asserts it never returns and is checked only in debug/validation modes,
  so its interaction with the formal validator (see `safety-and-validation.md`) — whether a missing
  `return` on a path is UB, an error, or relies on `@unreachable` — needs a precise rule.
- **[Future]** No lambda functions (this is not a functional language) and no nested functions;
  both are deliberately deferred.
- **[Future]** No generics beyond `generic` blocks; a broader generics facility may be added if a
  real need appears.
- **[Future]** No `@inline` **ordering** hint yet; an attribute may be added later to tell the
  compiler in which order to attempt inlining calls (for example ordering calls by decreasing impact
  when inlined).
- **[Future]** A `@callconv{"..."}` attribute to select calling conventions other than the custom
  and C ones may be added later (see `modules-and-interop.md`).
- **[Future]** An FFI layer with automatic import of C header files via `import "library.h"` may be
  added later (shared with `modules-and-interop.md`).
