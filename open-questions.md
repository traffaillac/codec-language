# Open Questions

This file is the home for the **cross-cutting and orphan** open questions of codec-lang — the
unresolved decisions that span several chapters or belong to none. It is deliberately **not** a
roll-up or index of the per-chapter backlogs: every question that has a clear single-chapter home
stays in that chapter's own "Open Questions" section and is not repeated here. Only questions with
no single owner live in this file. As everywhere in the Doc_Set, the semantic model is **C-based**
and the surface syntax is **Python-inspired**, the framing is the **codec-development** context
(predictable cost, hot loops, SIMD, tight C interop), and the authoritative source of truth for
every settled decision is the workspace-root `decision-log.md` — this file only collects the
questions those decisions leave open across chapter boundaries.

Questions here use `[Open]` for unresolved decisions, ambiguities, likely confusions, and gaps, and
`[Future]` for deliberately deferred items. References to other chapters name the target file (for
example `operators-and-expressions.md`).

## Cross-cutting reminders ("Don't forget")

*This group collects the decision-log "Don't forget" reminders — cross-cutting backlog items that
have no single-chapter home, each framed for codec work. Each reminder notes the chapter it leans
toward, but the reminder text lives only here and is not duplicated into those chapters.*

- **[Open]** Directives that let a codec author *assert* optimization outcomes rather than merely
  request them: one to assert that a `return` provably reduces to a tail call, and one to assert
  that a region's emitted instruction count stays below a threshold. These matter for hot loops
  where a silently non-tail call or an unexpectedly large body wrecks predictable cost. `@tailcall`
  (see `functions.md`) is only a partial answer — it *warns* when the last call cannot become a
  tail call but does not hard-assert the reduction, and there is no instruction-count directive at
  all. What the assertion forms look like, in which build modes they fire, and whether they live as
  directives in `builtins-and-directives.md` or as function attributes in `functions.md` is
  unresolved (affinity: `builtins-and-directives.md` / `functions.md`).

- **[Open]** An alternate-execution-block construct paired with equivalence validation. The `alt:`
  blocks in `control-flow.md` already express several hand-tuned variants of a hot region and lean
  on the validator to prove they have the same external effects (see `safety-and-validation.md`),
  but the reminder is broader than the current `alt:` sketch: what the full construct should offer a
  codec author (how many variants, how a per-target choice is pinned, what "equivalent" is allowed
  to ignore such as timing or scratch state) is still open (affinity: `control-flow.md` `alt:` /
  `safety-and-validation.md`).

- **[Open]** A full audit of the GCC extensions and builtins to decide, one by one, which codec-lang
  adopts as-is, which it adopts under a renamed/normalized spelling, and which it omits. codec work
  leans heavily on these (bit-counting, prefetch, overflow-checked arithmetic, vector intrinsics),
  so the surface that ends up in `builtins-and-directives.md` needs a deliberate inventory rather
  than ad-hoc addition, including how each adopted builtin behaves at the C boundary (affinity:
  `builtins-and-directives.md`).

- **[Open]** A struct-field assert specifier — a way to attach a checked invariant to an individual
  field (for example an expected offset, alignment, or value constraint) so that codec structs
  mapped onto packets or hardware layouts fail loudly when an assumption breaks. How it is spelled,
  whether it rides on the field type in `type-system.md` or on a directive from
  `builtins-and-directives.md`, and in which build modes it is enforced are all open (affinity:
  `type-system.md` / `builtins-and-directives.md`).

- **[Open]** First-class support for debugging and printing. codec-lang has no boolean or
  floating-point types and reorders struct fields by default, so a generic "print this value or
  struct" facility is not obvious; what debug/printing builtins should exist, how they render
  integers, vectors, and reordered structs, and how they interact with the release/debug/validation
  modes is unresolved (affinity: `builtins-and-directives.md`).

- **[Open]** The placement and applicability of attributes/directives: should an attribute come
  *before* or *after* the element it modifies, and exactly which elements (variables, fields,
  functions, statements, blocks) each attribute may attach to. Because codec-lang leans on `@`-forms
  for storage class, alignment, packing, inlining, and more, a consistent placement rule and a clear
  per-attribute applicability table are needed so authors are not guessing per directive (affinity:
  `builtins-and-directives.md` — directive placement).

- **[Open]** A precise description of the automatic struct field reordering: what inputs the
  compiler considers (field size, access frequency, code locality, `@order{n}` hints) and the exact
  algorithm and tie-breaking it uses to produce the final layout. Since reordering is the default
  and `@extern` is the opt-out (see `memory-model.md`), a codec author who needs to reason about
  cache behavior or interop cannot currently predict the resulting layout; the algorithm needs to be
  pinned down (affinity: `memory-model.md`).

- **[Open]** How to *formally* validate that the stack and heap do not run out of space. codec-lang
  models values as register-scoped by default and lets the compiler silently spill to stack or heap
  (see `memory-model.md`), and it aims for validation-mode guarantees rather than runtime checks
  (see `safety-and-validation.md`); what contract or analysis proves a bounded-memory codec never
  exhausts either region, and what the author must state for that proof to go through, is open
  (affinity: `safety-and-validation.md` / `memory-model.md`).

## @alloc overloaded meaning

*This group states, once, the cross-cutting `@alloc` overloaded-meaning question, which spans
`variables-and-bindings.md`, `memory-model.md`, and `builtins-and-directives.md`.*

- **[Open]** `@alloc` currently carries two distinct meanings (inconsistency-resolution #6, left
  undecided). As a variable storage-class attribute, `@alloc a` gives the variable heap/global-scope
  storage (see `variables-and-bindings.md`); as a heap-allocation expression, `p = @alloc T{...}`
  returns a `T*` allocated on the heap with alignment and initialization handled (see
  `memory-model.md`). The same `@`-token therefore reads as an attribute in one position and as an
  allocating expression in another, which also pulls in how `@`-directives are parsed and
  disambiguated (see `builtins-and-directives.md`). Whether to keep the dual meaning or rename one
  use — and, if renamed, which one and to what — is unresolved. It is stated once here because it
  belongs to no single chapter; `variables-and-bindings.md`, `memory-model.md`, and
  `builtins-and-directives.md` only note the clash and point back to this file.

## Other cross-cutting / orphan questions

- **[Open]** The specifier-glyph system is applied at many independent sites, but no single chapter
  owns whether it is *globally* consistent. The three glyphs — `Ω` (overflow: `|` saturation, `%`
  wrapping, or absent = undefined), `Σ` (signedness: `$` unsigned, or absent = signed), and `Δ`
  (alignment: `~` unaligned, or absent = aligned) — recur on binary operators and the split-placement
  shifts (`operators-and-expressions.md`), on casts and narrowing conversions (`type-system.md`), on
  slices and on vector/array indexing including the swizzle form (`type-system.md`), and they connect
  to alignment and unaligned access in `memory-model.md`. The open question is whether the same glyph
  means the same thing at every site: are the placement rules uniform (the right shift already flips
  where `Ω` and `Σ` sit relative to the other operators), are there sites where a specifier is
  silently ignored or is an error, are there operations where two specifiers legitimately apply at
  once (an index *and* a result can each overflow), and does the "absent" default consistently mean
  signed / aligned / UB-on-overflow everywhere? A codec author has to carry one mental model for
  these glyphs across `operators-and-expressions.md`, `type-system.md`, and `memory-model.md`, and it
  is unresolved whether that model is actually uniform.

- **[Open]** codec-lang defines three build modes — **release**, **debug**, and **validation** — and
  a surprising number of features change behavior across them, yet no chapter owns the mode model as
  a whole. `@assert` is ignored in release, checked at runtime in debug, and becomes a proof
  obligation in validation, while `@assume` is never checked in release or debug and is only a
  validation-mode hint (`control-flow.md`); `alt:` alternate blocks are compared against the first
  variant at runtime in debug and proved equivalent in validation (`control-flow.md`); bounds
  checking on indexing is UB in release but checked in validation, and integer-overflow checking is
  driven by the `%`/`|` specifiers (`safety-and-validation.md`, `operators-and-expressions.md`,
  `type-system.md`); and `@unreachable` behaves as a mode-dependent assert (`control-flow.md`,
  `functions.md`). What each mode guarantees, how a codec is meant to progress from debug through
  validation to a release build, whether a program can behave differently — or expose UB — only in
  release, and how a mode is selected in the first place are cross-cutting questions that touch
  `control-flow.md`, `safety-and-validation.md`, `operators-and-expressions.md`, and
  `type-system.md`.

- **[Open]** The SIMD / vector programming model — the core of codec work — is split across four
  chapters with no unifying home. Vector types, swizzles (`v[w]`), and the three vector-cast cases
  live in `type-system.md`; per-lane comparison masks (`-1`/`0` rather than `0`/`1`) and the vector
  overflow specifiers live in `operators-and-expressions.md` and `type-system.md`; array alignment
  widened to the host SIMD width and the no-decay vector-pointer rules live in `memory-model.md`; and
  the find-first-set / bitmask loops (`for i=..?vector:`) live in `control-flow.md`. Whether these
  compose into one coherent, portable story is unresolved: how a mask produced by a comparison flows
  into a bitwise select/blend and then into a set-bit iteration loop, how lane count and element
  width are tracked end-to-end through casts and swizzles, and how host-SIMD-width assumptions
  (array alignment, `alt:` variant selection per target) stay predictable when a codec is built for
  different CPUs. This spans `type-system.md`, `operators-and-expressions.md`, `memory-model.md`, and
  `control-flow.md`.
