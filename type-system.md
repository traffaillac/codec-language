# Type System

The type system is explicit, statically checked, and **nominal**. Its semantics follow **C** while
its surface syntax is **Python-inspired**, and every choice below is weighed against the
codec-development context: predictable cost, hot loops, SIMD-friendly layouts, and tight C interop.
Settled decisions come first; the unresolved decisions, edge cases, and likely confusions are
gathered in the "Open Questions" section at the end. Pointer *semantics*, memory *layout*, and the
authoritative homes of a few rules referenced here live in `memory-model.md`; operator behavior
lives in `operators-and-expressions.md`.

## Integers and literals

- **[Decided]** Integer types are `i8`, `u8`, `i16`, `u16`, `i32`, `u32`, `i64`, `u64`, plus the
  native-width aliases `isize` and `usize` (same set as Rust) — the fixed widths keep struct layout
  and SIMD lane counts explicit for codec work.
- **[Decided]** Integer literals default to **signed** unless given a `u` suffix.
- **[Decided]** A literal's default size is the **smallest type that fits its value**; a value that
  overflows 64 bits is an error.
- **[Decided]** Type bounds are queried with `i32.MIN` / `i32.MAX`.
- **[Decided]** There is **no boolean type and no floating-point type**. **(differs from C)** — C
  provides `_Bool`, `float`, and `double`; codec-lang has none, so predicates and DSP-style math are
  expressed with integer and vector types only.
- **[Decided]** Scalar comparisons return `i8` (`-1` for true, `0` for false); vector comparisons of
  two `uN[M]` operands return `iN[M]` per-lane (`-1`/`0` in each lane) — resolution #8(b).
  **(differs from C)** — C comparisons yield `int` with value `0` or `1`, not an all-ones `-1` mask.
  The `-1` mask form is chosen because it feeds directly into bitwise select/blend patterns in hot
  SIMD loops.

## Pointers (type side)

Pointer *types* are described here; pointer *arithmetic*, comparison, provenance, and the
address-space model are settled in `memory-model.md`, and the no-arithmetic / indexing-only rule is
in `operators-and-expressions.md`.

- **[Decided]** Pointer type syntax is `T*` (familiar to C).
- **[Decided]** A pointer may carry an optional `@stride{N}` attribute, initialized to the pointed
  type's size; it makes `&a[i]` conceptually equal to `a + i*N`. This is *explanatory only* — there
  is no pointer arithmetic (see resolution #1 in `operators-and-expressions.md`).
- **[Decided]** Stride and alignment are queryable: `T.STRIDE`; and `T.POINTED_ALIGNMENT`,
  initialized to the GCD of the stride and the pointed-type alignment.
- **[Decided]** Unaligned access is written `a~[i]` / `a~.name`; such an access may fragment into
  several aligned accesses on hosts with smaller registers. An access *should* be unaligned when the
  stride is not a multiple of the type size, otherwise the compiler warns.
- **[Decided]** There is **no `void*` equivalent**; alias to `u8*` when talking to external
  libraries. **(differs from C)** — C uses `void*` as the generic pointer; codec-lang uses `u8*` so
  that the pointed size and stride stay concrete.

## Slices (internal type)

- **[Decided]** A slice is written `start..range..step` and is valid **only** inside a `for`
  assignment, an array index, and a `case` label — it is an internal type, not a first-class value.
- **[Decided]** `start` is the first index (must be in the valid range when the size is known);
  `range` is the element count (a compile-time constant greater than 0); `step` is a compile-time
  constant increment (may be 0 to repeat the same element).
- **[Decided]** `start..range` omits the step (defaults to 1); `start` is always present even when
  it is 0; `range` may be omitted only in a `for` assignment (`start..`, `start....step`).
- **[Decided]** Overflow specifiers on slices: saturation `start..|range` clamps each index to
  `[0, len-1]`; wrapping `start..%range` takes each index modulo `len`; without a specifier,
  out-of-bounds indices are UB.
- **[Decided]** Using a saturation or wrapping specifier on anything other than a **known-size**
  array is an error.

## Vectors and arrays

- **[Decided]** The vector type `T[N]` is a fixed-size array and is the only array form supported
  now.
- **[Decided]** Runtime-size arrays reduce directly to pointers (the compiler keeps no size): there
  is no `alloca`, no end-of-struct `T[]` flexible member, and no dynamic heap array yet.
- **[Decided]** Initializers: element list `i32[5]{0,1,2,3,4}`; size-omitting `i32[]{...}`; and
  broadcast `i32[5]{4}`, which sets every lane to `4`.
- **[Decided]** C-style multidimensional indexing `v[i][j]` means `v + i*n + j`; slicing targets
  only the rightmost vector, e.g. `v[i][0..4]`.
- **[Decided]** Indexing a vector with a *variable* index requires the vector to be `@stack` or
  `@alloc` (or to inherit that from a containing struct); see `variables-and-bindings.md`.
- **[Decided]** Vector index overflow specifiers: wrap `v%[i]`, or return 0 when out of bounds
  `v|[i]`. Returning 0 was chosen over saturating the index inside bounds because it matches common
  hardware behavior in codec kernels.
- **[Decided]** `v[w]` (a vector indexed by a vector) is a **swizzle** when both operands share the
  same element type; the result size equals the size of `w`, and it supports the wrap/saturation
  specifiers.
- **[Decided]** Vectors have a genuine vector type and **do not decay to pointers**. **(differs from
  C)** — a C array silently decays to a pointer to its first element; codec-lang keeps the vector
  type so lane count and layout are never lost. Consequences:
  - `&i32[4]{}` has type `i32[4]*`; `i32*{&a}` casts a vector address to a base-type pointer (and
    requires `@stack`/`@alloc`).
  - Passing a vector to a pointer parameter does **not** decay: `f(i32* p)` rejects a vector
    argument.
  - Vector pointers auto-promote to same-base-type pointers (`f(&i32[4]{})` works) but **not** to
    different-size vector pointers.
- **[Future]** Variable-length arrays and a first-class dynamic heap array `T[+]` may be added
  later.

## Structs and unions

- **[Decided]** A struct is declared `struct MyContext:` with indented fields;
  `ctx = MyContext{*}` leaves the fields with undefined values.
- **[Decided]** Fields are **automatically reordered** by the compiler based on size, usage
  frequency, and code locality. **(differs from C)** — C preserves declaration order; codec-lang
  reorders for packing and cache behavior. `@extern` opts out (see `memory-model.md`).
- **[Decided]** Layout directives: `@extern` prevents reordering and follows C padding (exporting a
  C-compatible struct); `@aligned{N}` forces alignment on a struct or field, including *decreasing*
  it (contrary to GCC); `@packed` packs like C (member default alignment becomes 1); and
  `@aligned{1}` on a member packs just that member.
- **[Decided]** Anonymous `struct:` and `union:` are allowed inside a struct, and a named
  `union MyUnion:` is supported (it eases conversion from C).
- **[Decided]** `@order{n}` pins a field to a position; `..OtherContext oc` includes `oc`'s fields
  and namespace.
- **[Decided]** There is no encapsulation or visibility. Only designated initializers are allowed,
  e.g. `MyContext{a=0, b=1}`.
- **[Decided]** `i32 MyContext.c = 10` at global scope adds a defaulted field — a constant available
  while the struct is in scope, which avoids PC-relative addressing.
- **[Decided]** A member's offset is queried with `T.OFFSET_name`.
- **[Future]** `@minaligned` may be added; explicit padding members named `_` may be added.
- **[Future]** No bitfield support yet — bitfields may be added later on top of the `iN`/`uN` types.

## Inference, qualifiers, and conversions

- **[Decided]** Type inference is local, with explicit function interfaces: `a = 4` infers the
  type, `a = i8{4}` states it explicitly, and `a = i32{4}` applies automatic promotion (`i8` →
  `i32`).
- **[Decided]** The `@restrict` qualifier declares that a pointer will not alias any other.
- **[Future]** There is no mutability/constness of *pointed* values (only the variable binding is
  mutable or immutable); `@readonly` / `@writeonly` directives may be added later.
- **[Decided]** Casts and initializations share the `T{a}` syntax; the only implicit conversions are
  integer promotions, and everything else is explicit.
- **[Decided]** Scalar↔scalar casts are allowed, but **pointer↔integer casts are restricted to
  `usize`/`isize`** (resolution #2; authoritative home `memory-model.md`). **(differs from C)** — C
  allows casts to and from arbitrary integer widths.
- **[Decided]** A narrowing cast assumes the value fits by default (otherwise the result is any
  value on overflow); `T%{a}` wraps and `T|{a}` saturates. Requesting wrap or saturation where
  overflow is impossible or nonsensical (e.g. int → pointer) is an error.
- **[Decided]** Vector casts follow three cases: same count and element size → per-element cast
  (wrap/sat allowed); same overall bit size → reinterpret without conversion, e.g.
  `i32[4]{i8[16]{}}` (wrap/sat **not** allowed); same count but different element size →
  promote/reduce each element, e.g. `i8[16]{i16[16]{}}` (wrap/sat allowed **only** when reducing).
- **[Decided]** `signed{a}` / `unsigned{a}` cast to the signed/unsigned form of the same base size,
  for both integers and vectors.
- **[Decided]** The type system uses explicit type expressions with **nominal** compatibility and
  equivalence: two structs are interchangeable **iff** they share the same type name; checking is
  static. **(differs from C)** — C treats struct compatibility structurally/by-declaration rather
  than by name interchangeability in this sense.
- **[Decided]** Type queries: `T.SIZE` (bytes), `T.BITSIZE`, `typeof(a)`, and `T.ALIGNMENT`.

## Promotion matrix

- **[Decided]** The promotion matrix is authoritative for row→column promotions; `isize`/`usize`
  map to their native-width `iN`/`uN` aliases. Two types are *compatible* when at least one promotes
  to the other, and *equivalent* (`A<->B`) when both promote. The matrix drives binary-op operand
  checks, overload distinctness, call-argument promotion, overload resolution (first match in
  definition order), and assignment to declared variables.
- **[Decided]** Binary-op sign rule: operand signs must match; the smaller operand is promoted to
  the bigger operand's size if needed; promotion never changes sign.
- **[Decided]** Pointer promotion (resolution #7; authoritative home `memory-model.md`): `A* → B*`
  when `A<->B` and `A*.POINTED_ALIGNMENT % B*.POINTED_ALIGNMENT == 0`.

## Open Questions

- **[Open]** Promotion-matrix corner cases are underspecified for codec-relevant mixes: how do
  pointer rows and vector rows interact in the same matrix, and what exactly happens at the
  alignment-modulo boundary of pointer promotion (`A*.POINTED_ALIGNMENT % B*.POINTED_ALIGNMENT`)
  when strides and pointed-type alignments disagree? The row→column table needs concrete entries for
  every scalar/vector/pointer pairing a hot loop might produce.
- **[Open]** Aliasing `void*` to `u8*` for external libraries is settled, but the interop ergonomics
  are not: how does a `u8*` obtained from a C API get re-typed to a concrete `T*` at the boundary,
  which casts are implicit vs. required, and does the pointer↔integer restriction (`usize`/`isize`
  only) force extra casts when a C header hands back an integer handle? Cross-reference
  `memory-model.md` and `modules-and-interop.md`.
- **[Open]** Slice and vector overflow specifiers (`start..|range`, `start..%range`, `v|[i]`,
  `v%[i]`) have an allowed-vs-error placement gap: they are an error on anything other than a
  known-size array, but the rules for multidimensional indexing (`v[i][j]`), for the swizzle form
  `v[w]`, and for slices whose `range`/`step` are themselves derived at compile time are not fully
  pinned down. Where exactly is a specifier accepted, and where is it a compile error?
- **[Open]** The literal defaults — signed unless a `u` suffix, smallest type that fits, error above
  64 bits — will surprise C programmers in edge cases: a bare literal that fits in `i8` may not
  combine with a `u32` under the sign-match binary-op rule, and mask/bit-pattern constants that
  "look unsigned" default to signed. The exact set of surprising cases and any diagnostics for them
  is open.
- **[Open]** Swizzle semantics for `v[w]` need to be nailed down for codec kernels: what are the
  out-of-range lane rules under the wrap (`%`) and saturation/return-0 (`|`) specifiers, whether the
  index vector `w` must share element *width* (not just base type), whether duplicate indices are
  allowed, and how a swizzle differs from a plain gather when `w` comes from memory.
- **[Open]** Automatic struct field reordering interacts with several type-side features in ways
  that are not fully specified: how `@order{n}` pins coexist with the reordering heuristic, how
  `..OtherContext oc` field inclusion is ordered, and how `T.OFFSET_name` stays meaningful when the
  compiler is free to move fields. The reordering inputs and algorithm are described in
  `memory-model.md`; the type-system-facing consequences (offset queries, initializer order,
  `@extern` opt-out) need to agree with it.
- **[Open]** Unaligned access via `a~[i]` / `a~.name` is likely to confuse: the compiler *warns*
  when stride is a multiple of type size (i.e. when the `~` is unnecessary) and an access *should*
  be unaligned when it is not, but the exact rule for when `~` is required vs. optional vs. warned,
  and how a fragmented unaligned access composes with the overflow specifiers, is open. This has a
  direct cost impact in hot loops, so cross-reference `memory-model.md`.
- **[Open]** No bool and no float is a large C-mental-model break for porting: predicates now yield
  an `i8` mask (`-1`/`0`) rather than `0`/`1`, and any DSP or fixed-point math that a C codec would
  express with `float`/`double` must be reworked in integer/vector form. The idioms for porting such
  code, and whether any helper types are expected, remain open.
- **[Open]** Comparison result types are a subtle confusion point: scalar comparisons give `i8`
  `-1`/`0` and vector comparisons give `iN[M]` per-lane `-1`/`0` (resolution #8(b)), but a C
  programmer expects `0`/`1`. How these results feed conditions, ternary `if(cond,t,f)`, and bitwise
  select needs worked examples; cross-reference `operators-and-expressions.md` and
  `control-flow.md`.
- **[Future]** Null-pointer handling has no dedicated type yet (a form such as `&` may be added),
  and fat pointers are not supported yet — both are deferred.
- **[Future]** Dynamic arrays `T[+]`, bitfields (on top of `iN`/`uN`), and explicit padding members
  named `_` are deferred.
- **[Future]** A `@minaligned` directive is a deferred capability alongside the existing
  `@aligned{N}`/`@packed` set.
- **[Future]** Constness/read-only or write-only qualification of *pointed* values via `@readonly` /
  `@writeonly` is deferred; today only the variable binding, not the pointed data, carries
  mutability.
