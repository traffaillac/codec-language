# Concurrency and Parallelism

This chapter covers the shared-memory concurrency model and the atomic operations available to
codec-lang programs. Its semantics follow **C** while the surface syntax is **Python-inspired**, and
every choice is weighed against the codec-development context: frame-level and slice-level
parallelism, hot loops, SIMD, and tight C interop. The area is deliberately thin — the
`decision-log.md` settled only the atomics story and deferred nearly everything else — so the
settled decisions below are short and the "Open Questions" section at the end is the substance of
this chapter. Data races are undefined behavior; that rule and the broader UB catalog live in
`safety-and-validation.md`, and the underlying memory layout and storage model live in
`memory-model.md`.

## Atomics

- **[Decided]** Atomic operations are available through C's `stdatomic.h`. (borrowed from C) codec
  workloads that need lock-free counters, sequence locks, or work-stealing across frame/slice
  workers reuse the C atomics surface directly rather than a bespoke language feature, keeping the C
  mental model and C interop intact.
- **[Decided]** The **default memory ordering is relaxed**. **(differs from C)** — C's
  `stdatomic.h` operations default to `memory_order_seq_cst` (sequential consistency); codec-lang
  makes relaxed ordering the default instead, because most codec-side atomics (counters, progress
  flags, statistics) do not need cross-thread ordering guarantees, and paying for sequential
  consistency by default would add hidden cost to hot paths. Stronger orderings must be requested
  explicitly where synchronization is actually required.

## Open Questions

- **[Future]** Language-level concurrency support may be added once the needs are clearer. Today
  there is no `spawn`, `thread`, `parallel for`, task, or channel construct in the surface language;
  concurrency is expressed only through C atomics and whatever the host runtime provides. A
  first-class story is deferred until real codec parallelism patterns (frame-level and slice-level
  workers) make the required primitives obvious.
- **[Future]** Volatile semantics are not supported at the moment and may be added later if needed.
  There is currently no `volatile` qualifier or equivalent directive, so memory-mapped I/O,
  compiler-barrier-style "do not optimize away this access" patterns, and similar C idioms have no
  spelling yet; codec code that needs them must reach into C.
- **[Open]** The threading model is entirely unspecified. codec parallelism — dispatching slices or
  frames across worker threads, running hot loops concurrently, and combining that with SIMD within
  each worker — has no defined story: there is no statement about how threads are created, scheduled,
  joined, or pinned, nor about what a "thread" even is at the language level. This gap must be closed
  before the concurrency semantics can be nailed down, and it interacts with the register-by-default
  storage model in `memory-model.md` (what does register scope mean across threads?).
- **[Open]** The relaxed default ordering is likely to surprise programmers who expect sequential
  consistency — both C programmers (whose `stdatomic.h` defaults to `seq_cst`) and anyone porting
  code that silently relied on stronger ordering. An atomic that "looks fine" may reorder in ways
  that break a hand-rolled synchronization protocol; the docs need a clear account of when a
  codec author must upgrade to acquire/release or sequentially consistent ordering, and how that is
  spelled.
- **[Open]** How atomics interact with the memory model is undefined. codec-lang has **no strict
  aliasing** and does **not track pointer provenance** (see `memory-model.md`), and any pointer may
  alias any other; it is unclear how an atomic access to a location that is also reached through an
  aliasing non-atomic pointer, or through a type-punned pointer, is supposed to behave. The
  interaction of atomicity with `@noalias`/`@restrict` assertions also needs a defined rule.
- **[Open]** How atomics interact with the register-by-default storage model is undefined. Values
  default to a register (register scope) unless declared `@stack` or `@alloc` (see
  `memory-model.md`); an atomic operation fundamentally needs a stable memory address, so it is
  unclear whether taking part in an atomic operation forces a value to `@stack`/`@alloc`, whether
  atomics are only permitted on already-addressable storage, and how the silent register→stack/heap
  spill is expected to interact with shared, concurrently accessed locations.
- **[Open]** Data races are undefined behavior (the catalog lives in `safety-and-validation.md`),
  but "data race" cannot be defined precisely without a full memory model. There is no
  happens-before relation, no notion of release/acquire synchronization, and no defined set of
  atomic orderings in the language spec yet, so the exact boundary between a well-synchronized
  program and one with a race — and how the formal validator is meant to reason about it — remains
  open.
- **[Open]** Whether SIMD and data parallelism are a language concern or left to the
  compiler/autovectorizer is unresolved. codec hot loops lean heavily on SIMD (see the vector types
  in `type-system.md`); it is unclear where the line sits between explicit vector operations the
  programmer writes, autovectorization the compiler performs, and any future data-parallel construct
  — and whether "parallelism" at the SIMD/lane level is even the same concern as thread-level
  concurrency covered by this chapter.
- **[Open]** Availability of memory fences and barriers is unspecified. Beyond the ordering argument
  on individual atomic operations, there is no decision on whether standalone fence/barrier
  primitives (e.g. a `stdatomic.h`-style `atomic_thread_fence`, or a compiler-only reordering
  barrier) are exposed, how they are spelled, and how they compose with the relaxed default.
- **[Open]** Thread-local storage has no story. There is no thread-local storage class or attribute,
  and it is unclear how per-worker scratch buffers (a common codec pattern) would be declared, how
  that would interact with the `@stack`/`@alloc` storage classes, and whether the register-by-default
  model even admits a thread-local concept.
- **[Open]** Which concurrency syntax is allowed versus disallowed is undefined. Because no
  concurrency constructs exist in the grammar yet, there are no rules for what a program may or may
  not write when it needs threads, atomics beyond the C surface, orderings, or fences; the
  allowed/disallowed-syntax boundary for this area is a blank slate to be filled once the threading
  model is chosen.
