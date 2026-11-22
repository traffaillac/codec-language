# Language Documentation (working draft)

This is a rebuilt documentation for the language, regenerated from `decision-log.md`
during a project pause. It is a hybrid between a language reference and a design walkthrough:
most of the language is settled, but a number of design choices are still open. The goal of this
version is to make the current state easy to re-read later and to reformulate the remaining
questions so they are relevant to the codec-development context.

## About the language

- Domain: codec development ([edge264](https://github.com/tvlabs/edge264) and similar), where dynamic allocation is rare and
  hot loops, SIMD, and predictable cost dominate.
- Semantics: C is the reference model. Deviations from C are deliberate and are called out.
- Syntax: inspired by Python (indentation blocks, `and`/`or`, `#` comments, multiple assignment).
- Guiding value: concise, quick-to-type, explicit code with no hidden costs.

## How to read this documentation

Every notable item carries a status marker so open decisions stay visible:

- **[Decided]** — settled in the decision log.
- **[Open]** — a choice still to be made, an unresolved contradiction, or a gap I could not fully
  reconcile against the decision log.
- **[Future]** — deliberately deferred ("may add later").

When a described feature is a subtle deviation from C, it is marked **(differs from C)** and the
deviation is spelled out, so the C mental model does not silently leak in.

## Suggested reading order

1. `design-principles.md` — the philosophy that drives every other choice.
2. `lexical-structure.md` — source files, blocks, comments, identifiers, literals.
3. `variables-and-bindings.md` — declaration, mutability, storage classes, scope.
4. `type-system.md` — integers, pointers, slices, vectors, structs, strings, casts, promotion.
5. `operators-and-expressions.md` — operators, specifiers, precedence.
6. `control-flow.md` — conditionals, loops, branch tables, `alt` blocks, asserts.
7. `functions.md` — signatures, returns, overloading, references, generics, inlining.
8. `memory-model.md` — layout, alignment, pointer semantics, allocation, aliasing.
9. `safety-and-validation.md` — UB model and the formal validator.
10. `concurrency-and-parallelism.md` — atomics, ordering, threading (mostly open/future).
11. `modules-and-interop.md` — modules, imports, C FFI.
12. `builtins-and-directives.md` — built-in functions and every `@` attribute/directive.
13. `open-questions.md` — the consolidated backlog of everything still to decide.

## Source of truth

`decision-log.md` remains the authoritative record of decisions, including the
"Inconsistency resolutions (session-pause review)" section appended at its end. Where this
documentation and the decision log disagree, the decision log wins and this doc should be fixed.
