# Modules and Interop

How code is split into files, how those files see each other, and how the language talks to C.
The *semantics* follow **C** — a flat, predictable object model with no hidden runtime — while the
*surface syntax* is **Python-inspired**, so files are organized with `import` statements rather than
textual `#include`. Every choice below is weighed against the codec-development context: tight C
interop for dropping into existing codec pipelines, predictable symbol layout, and no hidden cost at
module boundaries. Settled decisions come first; the unresolved decisions, edge cases, and likely
confusions are gathered in the "Open Questions" section at the end. The `@extern` attribute is
shared with `functions.md` (calling conventions) and `memory-model.md` / `type-system.md`
(C-compatible struct layout); the directive vocabulary itself is cataloged in
`builtins-and-directives.md`.

## Source files and modules

- **[Decided]** There are **no header files**. A program is built from a single type of source
  file that carries both declarations and definitions, wired together with `import` statements and
  modules. **(differs from C)** — C splits an interface into a header (`.h`) that is textually
  `#include`d and an implementation into a translation unit (`.c`); codec-lang has one kind of file
  and no preprocessor-style textual inclusion, so there is no header/implementation split to keep in
  sync.
- **[Decided]** A variable defined in **global scope is a global (module-level) variable** — it
  belongs to the module it is written in rather than to any function. This is the module-level
  storage that the block/register scoping rules in `variables-and-bindings.md` stop at.

## Importing compiler extensions

- **[Decided]** Compiler extensions are **explicitly imported**, never implicitly in scope. The
  import form is `import "compiler" [prefixed comp.]`, where the `[prefixed comp.]` part is
  **optional**: with it, the imported names are reached through the `comp.` prefix; without it, they
  are brought in unprefixed. Making the import explicit keeps the compiler-provided surface out of
  ordinary code unless a file asks for it. The catalog of what the compiler exposes lives in
  `builtins-and-directives.md`.

## Interop with C: `@extern`

- **[Decided]** `@extern` on a **function** makes it use the **host's default C calling
  convention** so it can be called from — or call into — C directly. Without `@extern`, the language
  uses its own **custom optimized calling convention** (see `functions.md`, where the interaction
  with `@ref` pass-by-reference is settled: `@ref` relies on the custom convention and so cannot be
  used on an `@extern` function). **(differs from C)** — in C every function already uses the
  platform C convention by default; here the C convention is the opt-in, and the default is a
  custom, optimization-friendly one.
- **[Decided]** `@extern MyContext*` written in **global scope** signals the compiler to **export an
  opaque pointer** — a handle a C caller can hold and pass back without seeing the pointed-to
  layout. This is the intended way to hand a codec context across the C boundary without exposing
  its fields.
- **[Decided]** `@extern` on a **struct** additionally **guarantees C-compatible layout**: fields
  keep their declared order and follow C padding rules, so the struct can be shared with C
  byte-for-byte. The layout details (field-reordering opt-out, alignment, `@packed`) are settled in
  `memory-model.md` and `type-system.md`; `@extern` is the single switch that opts a struct out of
  the language's automatic field reordering.

## Deferred interop

- **[Future]** An **FFI layer with automatic C-header import** — writing `import "library.h"` to
  pull a C library's declarations in directly — may be added later, so external C code can be used
  without hand-writing `@extern` declarations. Shared with `functions.md`.
- **[Future]** A `@callconv{"..."}` function attribute to select **calling conventions other than**
  the custom and C ones may be added later (also noted in `functions.md`).

## Open Questions

- **[Open]** The **`import` statement's full syntax is unspecified**: what a module path looks like
  (a bare identifier, a dotted path like Python's `a.b.c`, or a quoted string like the
  `import "compiler"` form), whether selective imports (`from ... import name`), aliasing
  (`import x as y`), and wildcard imports exist, and how a path maps to a file or module on disk.
  For codec projects that vendor a handful of source files, the mapping from `import` target to
  actual file needs to be pinned down. (allowed-syntax gap)
- **[Open]** The **optional-prefix rule** shown for `import "compiler" [prefixed comp.]` needs a
  general grammar: is `[prefixed <name>.]` a syntax available to *every* import or only to the
  compiler-extension import, may an ordinary module be imported with or without a prefix the same
  way, what happens to unprefixed names that collide with local identifiers, and can a module choose
  its own default prefix. (allowed-syntax gap / likely confusion)
- **[Open]** **What `import "compiler"` actually exposes** is not enumerated: which extensions,
  builtins, and intrinsics become visible, how they are named (and whether the `comp.` prefix
  applies uniformly to all of them), and how that surface overlaps with the `@`-directives and
  builtins already available without an import. Cross-reference `builtins-and-directives.md`.
  (ambiguity)
- **[Open]** **Module visibility and name resolution** are unspecified: whether every module-level
  name is exported by default or there is a public/private distinction (`variables-and-bindings.md`
  notes structs have "no encapsulation / visibility", but it is unclear whether that extends to
  module-level symbols), how an imported name is qualified at the use site, and how the separate
  function namespace (see `functions.md`) is addressed across a module boundary. (ambiguity)
- **[Open]** **Circular imports** — two modules that import each other — have no defined behavior:
  whether they are allowed (as C's include-guard-protected headers effectively permit mutual
  reference), rejected, or resolved by some ordering. Because there are no forward declarations
  (`functions.md`), the rule for cross-module cyclic references needs stating. (edge case)
- **[Open]** How **modules interact with the single-source-file model** is unclear: whether a module
  is a whole file, a named region inside a file, or a directory of files; whether one file may
  declare several modules or a module may span several files; and how module-level globals and the
  "whole module's function set is visible" property (`functions.md`) scope when imports are
  involved. (ambiguity)
- **[Open]** **Linking against existing C libraries** is only partly covered by `@extern`: it is
  unresolved how the toolchain is told which C objects or libraries to link, how symbol names are
  mangled or preserved across the boundary, how an opaque `@extern MyContext*` export is consumed on
  the C side (what header or declaration a C caller writes), and how `@extern` functions that take
  or return structs marked C-compatible are matched to their C prototypes. Cross-reference
  `functions.md` and `memory-model.md`. (differs from C / edge case)
- **[Future]** The **automatic C-header FFI import** (`import "library.h"`) is deliberately
  deferred; when added it will need rules for translating C declarations into codec-lang types
  (mapping `void*` to `u8*` per `type-system.md`, handling C's `_Bool`/`float`/`double` which have
  no codec-lang equivalent, and reconciling C's declaration-order struct layout with automatic field
  reordering).
