# Operators and Expressions

Operators keep C's arithmetic meaning while dropping the parts that hide cost or invite silent
bugs in a codec hot loop. Overflow, alignment, and signedness are made explicit through a small
glyph vocabulary rather than left implicit, and the surface syntax borrows a few readability wins
from Python (chained comparisons, an `if(cond, t, f)` ternary). The semantic model is C; the
syntax model is Python-inspired.

## Operator set and overloading

- **[Decided]** No custom operators and no operator overloading. Every operator has one fixed
  meaning, so reading an expression never requires knowing which type is in play.
- **[Decided]** Logical operators `and` / `or` work on integers and pointers (as in C), not on
  vectors or structs. (borrowed from C)
- **[Decided]** No rotate operators. Bit rotation, when needed, goes through builtins rather than a
  dedicated operator.
- **[Decided]** Builtin bit-counting functions `popcount`, `ctz`, `clz` are provided instead of
  operator forms.
- **[Decided]** Negative values use two's-complement representation, so signed values interoperate
  cleanly with bitwise operators.

## Arithmetic and overflow

- **[Decided]** Integer division truncates toward zero, and modulo matches C. (borrowed from C)
- **[Decided]** Signed **and** unsigned integer overflow is undefined behavior unless it is
  explicitly allowed with a saturation (`|`) or wrapping (`%`) specifier. **(differs from C)** — C
  defines unsigned overflow as modular wraparound with no opt-in required; here even unsigned
  overflow is UB by default, so a codec author must state the intended overflow behavior at the
  operator. The checked-vs-UB consequences of these specifiers are described in
  `safety-and-validation.md` (resolution #4).
- **[Decided]** Division by zero is undefined behavior (it may produce an undefined value and/or
  crash).
- **[Decided]** The unsigned specifier `$` (or its absence, meaning signed) on a binary operator
  acts as an assertion: it must match the signedness of both operands. The one exception is the
  right shift, where the specifier applies to the left argument. Signedness handling ties into the
  promotion rules in `type-system.md`.

## Shifts

- **[Decided]** The right-shift operator is `>>` (resolution #8a corrects an earlier `<<` typo).
  Its overflow specifier sits on the **right**, targeting the shift amount: with wrapping the amount
  is taken as `b % width`, and with saturation the result is cleared when `b >= width`. Written in
  full with glyphs the operator is `Σ>>Ω`.
- **[Decided]** The left-shift operator `Ω<<Ω` carries an overflow specifier on the **left**,
  targeting the result: wrapping discards the shifted-out bits, and saturation clamps the exact
  result to the output type's bounds.
- **[Decided]** The asymmetry is deliberate — a shift can overflow in two independent places (the
  amount and the result), so the two specifier slots name which one you mean.

## Comparisons

- **[Decided]** Chained comparisons such as `lo <= x < hi` are supported. (borrowed from Python) —
  C has no chained comparison; there `lo <= x < hi` parses as `(lo <= x) < hi`, comparing a 0/1
  result against `hi`, which is almost never intended. Here the chain reads as the mathematical
  conjunction a codec author expects.
- **[Decided]** Comparison operators can carry a signedness specifier (`<Σ`, `<=Σ`, `>Σ`, `>=Σ`);
  `==` and `!=` do not. Scalar comparisons return `i8` (`-1`/`0`) and per-lane vector comparisons
  return `iN[M]`; see `type-system.md` for the result-type rules (resolution #8b).
- **[Decided]** Whole-vector and whole-struct comparisons must go through `memcmp` rather than a
  bare `==`, so a byte-by-byte compare over a large aggregate never hides behind an innocent-looking
  operator.
- **[Decided]** Pointer comparison behavior lives with the memory model: comparisons are well
  defined under the circular (wrapping) address space, and creating/comparing out-of-bounds
  pointers is allowed while only dereferencing them is UB (resolution #3, see `memory-model.md`).

## Logical and conditional expressions

- **[Decided]** `and` and `or` short-circuit: the right operand is not evaluated when the left
  already determines the result.
- **[Decided]** The ternary conditional is written `if(cond, t, f)` and does **not** short-circuit —
  both `t` and `f` are evaluated. **(differs from C)** — C's `cond ? t : f` evaluates only the
  selected branch. Because both arms always run here, side effects or costly calls in the unused arm
  still happen, which matters when reasoning about predictable cost in a hot loop.

## Pointers, indexing, and access

- **[Decided]** **No pointer arithmetic** — indexing only. `p[i]` reads an element and `&p[i]` takes
  its address; `p + i` is illegal (resolution #1, which supersedes the earlier "pointer arithmetic
  is allowed as in C" note). **(differs from C)** — C allows `p + i`, `p - i`, and `p++`. The
  `&a[i] == a + i*N` stride relationship documented in `type-system.md` and `memory-model.md` is an
  explanatory identity about addresses, not a syntax you can write.
- **[Decided]** **No dereference operator** — read through a pointer with `p[0]` (resolution #8c,
  correcting an earlier `T[0]` phrasing). **(differs from C)** — C uses the unary `*p`. Indexing is
  the single spelling for both element access and dereference.
- **[Decided]** `->` accesses a struct field through a pointer, kept deliberately distinct from `.`
  so the indirection stays visible at the use site. (borrowed from C) — the two are not merged into
  a single `.` as some languages do.
- **[Decided]** No bounds checking is performed on vector indexing; that is the formal validator's
  job (see resolution #4 and `safety-and-validation.md`).

## Assignments

- **[Decided]** Assignments are statements, not expressions. **(differs from C)** — C's `=` is an
  expression, so `while ((c = next()))` and `a = b = 0` are idioms there; neither works here.
- **[Decided]** No chained assignment; assign one value to several targets with `a, b, c = d`.
- **[Decided]** No increment/decrement operators. **(differs from C)** — there is no `++` or `--`;
  write `i += 1`. (The `++` token exists, but as the concatenation operator, not increment.)

## Precedence and evaluation order

- **[Decided]** Evaluation follows operator precedence, and within a single operator, left to right.
- **[Decided]** Arithmetic and binary operators are split into just **two** precedence tiers,
  multiplicative and additive. Shifts sit in the multiplicative tier because a shift is equivalent
  to multiplying or dividing by a power of two, and bitwise AND sits there because it can act as a
  modulo by a power of two. Everything else is additive. No finer distinction is drawn — the
  intended practice is to settle any remaining ambiguity with parentheses.
- **[Decided]** Three specifier glyphs recur across the operators, each with a small closed set of
  values:
  - `Δ` — **alignment**: `~` (unaligned) or absent (aligned).
  - `Ω` — **overflow**: `|` (saturation), `%` (wrapping), or absent (undefined).
  - `Σ` — **signedness**: `$` (unsigned) or absent (signed).
- **[Decided]** Precedence, from tightest-binding to loosest:

  | Tier | Operators | Description |
  |-|-|-|
  | Grouping | `(x)`, `{x}` | expression in parentheses or braces |
  | Suffixes | `xΔΩ[index]`, `xΔ.attr`, `xΔ->attr`, `x(args)`, `xΩ{args}`, `x*` | indexing, field/offset access, pointer-field access, call, type cast, pointer type |
  | Prefixes | `+x`, `-Ωx`, `~x`, `!x`, `..x`, `..?x`, `@attr x` | positive, negative, bitwise NOT, logical NOT, key iterator, non-zero key iterator, attribute before an identifier |
  | Multiplicative | `*Ω`, `/Σ`, `%Σ`, `Ω<<Ω`, `Σ>>Ω`, `&` | multiplication, division, remainder, shifts, bitwise AND |
  | Additive | `+Ω`, `-Ω`, `++`, `^`, `\|` | addition, subtraction, concatenation, bitwise XOR, bitwise OR |
  | Comparisons | `<Σ`, `<=Σ`, `>Σ`, `>=Σ`, `==`, `!=` | comparisons (chainable) |
  | `and` | `and` | logical AND |
  | `or` | `or` | logical OR |

## Open Questions

- **[Open]** Signed and unsigned integer overflow is undefined behavior unless a `|` or `%`
  specifier is present, which is a major break from the C mental model — a C programmer expects
  unsigned arithmetic to wrap silently, and much existing codec code (hash mixing, checksum
  accumulation, ring-buffer index math) leans on that. How aggressively should the docs and the
  compiler warn when unsuffixed arithmetic could overflow, and is per-operator opt-in the right
  ergonomic granularity for a hot loop that wraps in dozens of places?

- **[Open]** With pointer arithmetic disallowed and indexing the only spelling (resolution #1),
  common C idioms built on `p + i`, `p++` walking a buffer, `end - begin` length computation, and
  two-pointer sweeps need a canonical translation. What are the recommended `p[i]` / `&p[i]`
  equivalents for each idiom, and is there a gap (for example, expressing "one past the end" or a
  pointer difference) that indexing alone cannot cover cleanly?

- **[Open]** The right shift is `>>` (resolution #8a) and its overflow specifier sits on the right,
  targeting the shift amount (`Σ>>Ω`), whereas the left shift's specifier sits on the left,
  targeting the result (`Ω<<Ω`). This split placement is subtle: the same glyph means "clamp the
  amount" on one operator and "clamp the result" on the other. How is this taught so a reader does
  not misread which quantity a given specifier constrains, and are there shift forms where both the
  amount and the result could overflow at once?

- **[Open]** Chained comparisons are borrowed from Python (`lo <= x < hi`), but the audience is
  C programmers for whom that expression means something else entirely. The exact evaluation
  semantics need pinning down: is the middle operand evaluated once (as in Python), how do the
  signedness specifiers distribute across a chain, and how does a chained comparison interact with
  the two-tier precedence and with `and`/`or`?

- **[Open]** The `if(cond, t, f)` ternary does not short-circuit, unlike C's `?:`, so both arms are
  always evaluated. For codec work this is a cost and correctness question: an unused arm with a
  side effect, a division, or an out-of-bounds index still runs. Should there be a short-circuiting
  alternative, or clear guidance that the ternary is only for cheap, side-effect-free arms?

- **[Open]** Only two arithmetic precedence tiers exist (multiplicative and additive), with shifts
  and bitwise AND folded into the multiplicative tier and XOR/OR into the additive tier. This
  differs from both C (which gives shifts, relational, and each bitwise operator their own levels)
  and from most readers' muscle memory. Expressions like `a & b + c`, `a << b | c`, or `a ^ b + c`
  will bind differently than a C programmer expects; where exactly are readers most likely to be
  surprised, and should the compiler require parentheses in the ambiguous cases rather than silently
  applying the two-tier rule?

- **[Future]** A third overflow variant that returns the value together with a boolean exception
  test may be added, e.g. `res, err = a +? b`, `res, err = i8?{a}`, and `res, err = a /? b` (note
  `/` can also overflow, and for vectors `err` would be a same-size vector). This would give a
  checked path alongside the UB / saturation / wrapping choices; its exact syntax, precedence, and
  interaction with multiple-return assignment are still open.
