# Memory Model

The memory model defines how values are laid out, how pointers behave, and where objects live. Its
semantics follow **C** while its surface syntax is **Python-inspired**, and every rule below is
weighed against the codec-development context: predictable cost, hot loops, SIMD-friendly layouts,
and tight C interop. Settled decisions come first; the unresolved decisions, edge cases, and likely
confusions are gathered in the "Open Questions" section at the end. Pointer *types* are described in
`type-system.md`; the no-arithmetic / indexing-only rule and other operator behavior live in
`operators-and-expressions.md`; storage classes on variables are in `variables-and-bindings.md`; and
the checked-vs-UB choices and the formal validator are in `safety-and-validation.md`.

## Layout and alignment

- **[Decided]** Structs follow C layout rules with automatic padding, but members are laid out in
  any **compiler-chosen order** unless the struct is `@extern`. **(differs from C)** — C fixes
  members in declaration order; codec-lang is free to reorder to shrink padding and group hot fields,
  and only `@extern` pins the order back to the source.
- **[Decided]** `@extern` guarantees a **bit-for-bit C-compatible** struct layout (it exports a C
  struct): fields stay in declaration order and follow C padding, so the struct can be shared across
  the FFI boundary. See `modules-and-interop.md` for the interop side.
- **[Decided]** `@packed` auto-inserts the packed C attribute (member default alignment becomes 1),
  and it is also applied on members whose alignment is forced below the default. See
  `type-system.md` for `@aligned{N}` / `@packed` on structs and fields.
- **[Decided]** Default alignments: an integer aligns to its size (same as extern); a pointer itself
  aligns to its size (same as extern); a struct aligns to the max of its member alignments (same as
  extern); a union aligns to the max of its member alignments (same as extern); and an **array
  aligns to the GCD of the biggest host SIMD size and the array size** (this one is different from
  extern). **(differs from C)** — C aligns an array to its element alignment; codec-lang widens array
  alignment toward the host's SIMD width so that whole-array SIMD loads land on aligned addresses in
  hot loops.
- **[Decided]** There is **no maximum alignment**; the `BIGGEST_ALIGNMENT` constant gives the
  largest alignment ever needed for any CPU operation on the target.
- **[Decided]** Padding bytes have **undefined values** both at initialization and when read. This
  is *not* UB — reading padding cannot crash, it just yields an unspecified value.
- **[Decided]** There is **no runtime type introspection**; the closest thing is the layout exported
  for a debugger.
- **[Future]** No big-endian host support yet; endianness may be added later.

## Pointer semantics

- **[Decided]** **No pointer arithmetic** — only indexing (`p[i]`, `&p[i]`); `p + i` is illegal
  (resolution #1). **(differs from C)** — C allows `p + i`. The behavior is settled in
  `operators-and-expressions.md`; it is restated here because it shapes the whole pointer model.
- **[Decided]** Function pointers allow only **calling and equality testing** — no indexing, no
  ordering comparison.
- **[Decided]** The language does **not track pointer provenance**: a pointer is just an address,
  with no compiler-maintained record of which object it was derived from.
- **[Decided]** There are **no restrictions on explicit pointer↔pointer casts**, but
  **pointer↔integer casts are only to and from `usize`/`isize`** (resolution #2). **(differs from
  C)** — C allows casts to and from arbitrary integer widths; codec-lang forces address-sized
  integers so that a round-trip cannot silently truncate an address. See the conversions rules in
  `type-system.md`.
- **[Decided]** Automatic pointer promotion happens **only per the promotion matrix** (resolution
  #7; this is the authoritative home): `A* → B*` when `A<->B` (the pointed types are
  promotion-equivalent) **and** `A*.POINTED_ALIGNMENT % B*.POINTED_ALIGNMENT == 0` (the source
  alignment is a multiple of the target alignment). This supersedes the earlier "same pointed type
  and same stride" phrasing. The matrix itself lives in `type-system.md`.
- **[Decided]** Type comparison for pointer compatibility is **recursive and nominative** (not
  structural): two pointer types match by name-based, structurally-recursive identity, and only
  promotion-compatible pointers can be compared.
- **[Decided]** `null` is guaranteed to be **all-zeros** (same as C).
- **[Decided]** **Cross-object pointer comparison is well-defined** under a circular (wrapping)
  address space (resolution #3): out-of-bounds pointers may be created and compared, and comparisons
  between pointers into different objects are meaningful; only **dereferencing** an invalid pointer
  is UB. **(differs from C)** — C only defines comparison of pointers into the same object (or one
  past its end); this supersedes the earlier "within the same object" rule.

## Allocation and lifetime

- **[Decided]** The theoretical machine has a **stack**, a **heap**, and a bank of **infinitely many
  registers**, each register holding a value of any size. Each value **defaults to a register**
  unless it is declared `@stack` or `@alloc`. **(differs from C)** — C has no register-by-default
  value model. Storage classes on variables are covered in `variables-and-bindings.md`.
- **[Decided]** Storage class determines scope: register storage → register scope; `@stack` → block
  scope; `@alloc` (heap) → global scope. The compiler may **silently spill** a value to the stack or
  heap when a shorter-lived scope runs out of space.
- **[Decided]** The compiler does **not track heap-object lifetimes**: there are no delete methods,
  no dangling-pointer support, no move semantics, and no borrowing rules.
- **[Decided]** There is **no ownership mechanism** (contrast with Rust): every pointer use is
  acceptable — including two mutable pointers to the same data — but inter-function interfaces must
  be described explicitly for the formal validator (see `safety-and-validation.md`).
- **[Decided]** `p = @alloc T{...}` returns a heap `T*`, handling both alignment and initialization;
  `@free p` calls `free` on the l-value and then sets it to `null`; the C `malloc`/`free` family is
  still usable directly; and a **failed allocation returns `null`**. (The `@alloc` token also names a
  storage-class attribute on variables, so it carries two meanings — that overloaded-meaning clash is
  a cross-cutting open question and is stated once in `open-questions.md`; see the note in the Open
  Questions section below.)
- **[Decided]** There is **no garbage collection**, no circular-reference detection, and no arena
  allocator.
- **[Future]** Support for alternate allocators may be added later.

## Aliasing and type punning

- **[Decided]** **Any pointer may alias any other**, regardless of type; the compiler must guarantee
  that reordering accesses through different pointers has no observable impact. **(differs from C)**
  — C's strict-aliasing (TBAA) rules let the compiler assume pointers of unrelated types do not
  alias; codec-lang drops that assumption so that hand-tuned aliasing in codec kernels is always
  safe.
- **[Decided]** `@noalias` on a pointer **asserts** that it aliases no other pointer (letting the
  compiler recover the optimizations that strict aliasing would have allowed); the related
  `@restrict` qualifier lives in `type-system.md`.
- **[Decided]** **Type punning** — viewing the same address through a different pointer type — is
  allowed, either via a cast or via a union. **(differs from C)** — C treats most type punning
  through incompatible pointer casts as UB and only blesses it through unions.

## Cache and SIMD alignment

- **[Decided]** Object-storage alignment (for a stack or heap object) is specified by writing
  `@aligned{N}` in front of the type, which is how a codec pins buffers to cache-line or SIMD-width
  boundaries.

## Open Questions

- **[Open]** Compiler-chosen field reordering versus the `@extern` C layout is a prime interop
  surprise: a struct that a C programmer reads top-to-bottom is silently reordered unless `@extern`,
  so a field's actual offset, the padding between fields, and even which fields are hot-grouped can
  differ from the source. When a struct is passed to C without `@extern`, when `@order{n}` pins fight
  the heuristic, and when `..OtherContext oc` inclusion changes the layout, the surprises multiply —
  the reordering inputs and the reordering algorithm themselves are still to be described (see the
  "Don't forget" reordering-algorithm reminder in `open-questions.md`), and the type-side
  consequences (`T.OFFSET_name`, initializer order) must agree per `type-system.md`.
- **[Open]** The array alignment rule — GCD of the biggest host SIMD size and the array size — has
  edge cases that need worked answers for codec buffers: what happens for a tiny array (size smaller
  than a SIMD lane), for an array whose size is coprime with the SIMD width (GCD collapses toward a
  small alignment), for arrays nested inside structs, and for arrays whose element type already
  carries a large alignment. The rule is different from extern layout, so the same array can align
  differently depending on whether its containing struct is `@extern`.
- **[Open]** The circular (wrapping) address space that makes cross-object pointer comparison
  well-defined (resolution #3) is subtle for a C audience: comparisons never trap and out-of-bounds
  pointers are legal to form and compare, so ordering two pointers into different objects "works" but
  its meaning depends on the wrapping model. How comparison interacts with the largest addresses
  (wraparound past the top of the space), whether `null` participates in ordering, and how this
  coexists with the no-provenance rule all need pinning down.
- **[Open]** Restricting pointer↔integer casts to `usize`/`isize` (resolution #2) is an
  allowed-vs-disallowed-syntax gap that will bite at the C boundary: a C API that hands back a
  handle as `u32`/`u64`, or code that packs tag bits into a pointer, must route through `usize`/
  `isize`, and it is unclear which conversions are implicit, which require an explicit cast, and what
  the diagnostic looks like when someone casts a pointer to a non-address-sized integer. Cross-
  reference `type-system.md` and `modules-and-interop.md`.
- **[Open]** Pointer promotion through the matrix (resolution #7) turns on the alignment-modulo
  condition `A*.POINTED_ALIGNMENT % B*.POINTED_ALIGNMENT == 0`, and the corner cases are open: how
  `POINTED_ALIGNMENT` (initialized to the GCD of stride and pointed-type alignment) behaves when
  `@stride{N}` is set unusually, whether promotion is transitive when three pointer types chain, and
  what happens when `A<->B` holds by promotion-equivalence but the alignments disagree. The matrix in
  `type-system.md` needs concrete entries for the pointer rows a hot loop produces.
- **[Open]** Dropping strict aliasing (any pointer may alias any other) together with no provenance
  tracking has real optimization implications that must be spelled out: the compiler cannot assume
  unrelated-typed pointers are distinct, so it must conservatively serialize accesses unless
  `@noalias`/`@restrict` is present. Where the burden of proof sits, how much codec performance rides
  on annotating no-alias in hot loops, and how type punning through a cast versus a union is expected
  to be optimized are all open. Cross-reference `safety-and-validation.md` for the type-punning UB
  boundary.
- **[Open]** Brief cross-reference only: `@alloc` is overloaded — it is both a storage-class
  attribute on a variable (`@alloc a`) and a heap-allocation expression (`p = @alloc T{...}`). The
  full cross-cutting question of whether to keep the dual meaning or rename one use lives in
  `open-questions.md` (resolution #6, spanning `variables-and-bindings.md`, this chapter, and
  `builtins-and-directives.md`); it is not restated here.
- **[Future]** Big-endian host support and alternate allocators are deferred; both would change the
  layout and allocation rules above and are called out where they land.
