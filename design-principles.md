# Design Principles

This chapter records the overall ethos that guides every other decision in the language: a
codec-development language whose **semantic model follows C** and whose **surface syntax is
inspired by Python**. The design is judged against the codec-development context — predictable
cost, tight hot loops, SIMD-friendly data layout, and close C interop — so the principles below are
tie-breakers for the concrete rules documented in the other chapters (see `type-system.md`,
`operators-and-expressions.md`, `memory-model.md`, and the rest of the Doc_Set).

## Language model

- **[Decided]** The semantic model is **C-based**: values, integers, pointers, undefined behavior,
  and the cost of each operation follow C's mental model, so a C programmer can predict what the
  generated code does. Individual departures from C are called out with `(differs from C)` in the
  chapter that owns them.
- **[Decided]** The syntax model is **Python-inspired**: indentation-based blocks, `#` comments,
  chained comparisons, and a compact, lowercase surface (details in `lexical-structure.md`,
  `operators-and-expressions.md`, and `control-flow.md`). The Python influence is surface-level
  only; it never changes the C semantics underneath.

## Core principles

- **[Decided]** Favor choices that produce concise code in the author's real codebase — the
  language is optimized around actual codec code, not synthetic examples.
- **[Decided]** Optimize for quick typing: prefer lowercase for the language and standard library
  (no `snake_case` or `camelCase`), and count cost in keystrokes (a digit counts as two keystrokes
  because of the Shift press). This is why type names such as `i32`/`u8` and directives are short
  and lowercase (see `type-system.md`).
- **[Decided]** Operations must be explicit and readable: someone unfamiliar with the language
  should be able to guess what a piece of code does. This principle drives, for example, keeping
  `->` distinct from `.` for pointer field access (see `operators-and-expressions.md`).
- **[Decided]** Rely on the reader's familiarity with C. The language reuses C's names, operators,
  and semantics wherever it can so that existing C knowledge transfers directly, and only spends
  novelty where it buys predictable cost or safety.
- **[Decided]** Avoid hiding costly operations. Anything with a non-obvious runtime cost is made
  visible in the syntax rather than implied — for instance, whole-vector and whole-struct
  comparisons must go through an explicit `memcmp` rather than a hidden `==` (see
  `operators-and-expressions.md`), and there is no operator overloading that could disguise an
  expensive call.

## Open Questions

- **[Open]** "Concise" and "quick to type" are stated as goals but not defined objectively. It is
  ambiguous whether the metric is raw keystroke count (the stated digits-count-as-two heuristic),
  token count, or subjective readability, and these can conflict — the shortest-to-type spelling is
  not always the most readable one. A codec author needs a tie-breaking rule for when brevity and
  clarity disagree in a hot loop.
- **[Open]** "Someone not familiar with the language should guess what it does" has no validation
  procedure. It is unclear who the reference reader is (a C programmer, a Python programmer, or a
  codec specialist) and how a proposed syntax is tested against this goal before it is adopted,
  which makes the principle hard to apply consistently across chapters.
- **[Open]** The "avoid hiding costly operations" principle sits in tension with ergonomic sugar
  such as combined op-assign (`a,b += 1,2`), multiple assignment, and swaps (see
  `variables-and-bindings.md`). Where the line falls — which conveniences are considered
  cost-transparent and which are considered hidden cost — is unresolved and needs a stated
  criterion (for example, "sugar is allowed only when it lowers to obvious, constant-cost code").
- **[Open]** Borrowed-feature tension: the Python readability ethos and C familiarity can point in
  opposite directions (Python favors expressive, high-level constructs; C favors explicit,
  low-level control). When they conflict — for example around implicit conversions, truthiness, or
  iteration protocols — it is undecided which ethos wins, and a default ("C semantics win, Python
  only shapes the surface") should be confirmed as the governing rule.
- **[Open]** The C-based semantics / Python-inspired syntax split has no stated boundary. Readers
  will carry both mental models, and it is a likely confusion which surface features are purely
  cosmetic (indentation, `#` comments) versus which change behavior relative to C. A short,
  authoritative "what Python gives you here is only syntax" statement would prevent Python
  programmers from assuming Python semantics (dynamic typing, arbitrary-precision integers,
  reference semantics) that the language does not provide.
- **[Open]** "Rely on familiarity with C" is a double-edged principle: every place the language
  departs from C (immutability by default, no pointer arithmetic, overflow as UB unless specified,
  no `bool`/`float`) is exactly where a confident C reader is most likely to guess wrong. It is an
  open design question how aggressively the design should lean on C intuition versus signalling
  "this is not C here", beyond the per-item `(differs from C)` callouts, so the familiarity
  principle does not become a trap in codec code.
- **[Open]** The principles are optimized for "the author's real codebase," which is a single codec
  codebase. It is ambiguous how to weigh a choice that is concise for that codebase but awkward for
  the broader codec-development audience the language targets, and whether the author's codebase is
  a hard oracle or just a strong signal.
- **[Open]** The principles are stated as unordered peers, but they routinely conflict (concise vs
  readable, C-familiar vs hidden-cost, quick-to-type vs explicit). There is no documented priority
  ordering to resolve conflicts, so two designers could justify opposite decisions by citing
  different principles. A ranked or lexicographic ordering of the principles is an open decision.
- **[Open]** The keystroke-cost model is under-specified as a design tool: it accounts for digits
  and Shift but says nothing about symbol keys, keyboard layouts other than the author's, or
  editor assistance (autocomplete, snippets). Whether keyboard-specific cost should influence the
  language surface at all — versus assuming tooling smooths it over — is unresolved.
- **[Open]** "No hidden costs" is scoped to source-visible operations, but the language also relies
  on compiler-driven behavior with real cost, such as automatic struct field reordering (see
  `memory-model.md`) and `alt:` block selection (see `control-flow.md`). Whether compiler-chosen
  costs count as "hidden" under this principle, and how they are surfaced to the author, is a
  cross-chapter ambiguity that the principle alone does not settle.
- **[Open]** It is undecided whether these design principles are binding constraints that every new
  feature must satisfy, or aspirational guidance that can be traded away. Codec work will surface
  cases where a fast, hardware-matching construct is neither concise nor guessable; the process for
  granting an exception to a principle is not defined.
- **[Future]** A concrete, measurable rubric for the subjective principles (a keystroke/readability
  scoring method, or a small "guess what it does" review checklist applied to each proposed syntax)
  is deferred until enough of the language is settled to calibrate it against real codec code.
