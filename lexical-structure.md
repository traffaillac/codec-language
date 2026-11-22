# Lexical Structure

This chapter defines how codec-lang source text is tokenized and shaped before parsing:
comments, block structure, statement termination, indentation, identifiers, keywords, and the
grammar notation. The language's **semantics follow C**, but its **surface syntax is
Python-inspired**, so the lexer looks much more like Python's (significant indentation, `#`
comments, optional semicolons) than C's brace-and-semicolon model. Everything here is aimed at
codec work, where source is read far more often than written and predictable, low-noise layout
matters. Cross-references point to `variables-and-bindings.md` (ephemeral-identifier usage),
`control-flow.md` (block-based statements), `operators-and-expressions.md` (bracket-delimited
continuation), and `builtins-and-directives.md` (`@`-prefixed directive lexing).

## Comments

- **[Decided]** Single-line comments start with `#` and run to end of line. **(differs from C)** —
  C uses `//` for line comments and `/* ... */` for block comments; codec-lang uses neither
  C form.
- **[Decided]** There are no multi-line comments. The `"""` triple-quote form was rejected because
  it does not nest cleanly, and editors already support commenting/uncommenting line ranges of
  `#` comments. **(differs from C)** — C provides `/* ... */` block comments.
- **[Decided]** A shebang header (`#!...`) is supported at the top of a source file, riding on the
  fact that `#` already begins a comment.
- **[Future]** Multi-line comments may be added later; the current decision is to defer them rather
  than adopt a non-nestable form.

## Block structure and indentation

- **[Decided]** Block structure is indentation-based, Python-style: a deeper indentation level
  opens a nested block and a dedent closes it. (borrowed from Python) **(differs from C)** — C
  delimits blocks with `{ }` and treats whitespace as insignificant. This keeps the visual layout
  and the actual structure in lockstep, avoids brace-only diff lines, and discourages the deep
  nesting that complex branching tends to produce.
- **[Decided]** Indentation may use spaces or tabs, the same as Python, but mixing spaces and tabs
  is disallowed. (borrowed from Python) Code examples in the documentation use tabs.
- **[Decided]** An inconsistent dedent must line up with a previously established indentation level,
  as in Python; a dedent that lands between known levels is an error.
- **[Decided]** Empty lines inside a block may either preserve the block's indentation or be fully
  blank, as in Python; the documentation uses and recommends the indentation-preserving style in
  examples.
- **[Decided]** Multi-line expressions are indented two levels deeper than their statement in code
  examples, so a continued expression is visually distinct from a nested block.
- **[Decided]** Empty blocks are forbidden; use `pass` instead, as in Python. (borrowed from Python)
  See `control-flow.md` for where blocks appear.

## Statement termination and continuation

- **[Decided]** Semicolons are optional. Most statements are terminated by a newline. (familiar to
  both Python and C programmers) **(differs from C)** — C requires a terminating `;` on statements.
- **[Decided]** An expression may span multiple physical lines when it is delimited by parentheses
  `()`, braces `{}`, or brackets `[]`; the open delimiter suppresses newline termination until it
  is closed. See `operators-and-expressions.md` for the expression forms that use these delimiters.
- **[Decided]** When multiple statements are written on one line separated by a semicolon, the
  statements after the semicolon keep the same indentation level as the first.

## Identifiers

- **[Decided]** Identifiers match `[a-zA-Z_][a-zA-Z0-9_]*`: a letter or underscore followed by
  letters, digits, or underscores.
- **[Decided]** A leading underscore is not merely a naming convention; it triggers *ephemeral
  scope* for that identifier (see below). **(differs from C)** — in C a leading underscore is a
  reserved-identifier convention with no scoping effect.
- **[Decided]** Ephemeral (`_`-prefixed) identifier scope — resolution #5 is the authoritative
  form: once an ephemeral identifier is assigned, it carries over all statements and branches that
  follow it (including a following `if`/`elif`/`else` chain and loop conditions) until the next
  assignment statement; at that next assignment it is available on the right-hand side and is then
  reset, so it can be reused. Example: `_lo, _hi = ..., ...`. This removes the need to invent
  temporary names for long intermediate computations and makes it easy to promote an ephemeral to a
  named binding later. See `variables-and-bindings.md` for binding rules and `control-flow.md` for
  usage in `if`/`elif` and loop conditions.

## Keywords

- **[Decided]** There are no contextual keywords; every keyword is globally reserved and cannot be
  used as an identifier anywhere.
- **[Decided]** Keywords are case-sensitive.

## Grammar notation

- **[Decided]** The grammar is written in PEG (Parsing Expression Grammar) notation, as Python
  does, because it is easy to read and unambiguous. (borrowed from Python)
- **[Decided]** The grammar targets the parser-generator rather than human readers; people are
  expected to use the reference documentation instead of the grammar itself.

## Open Questions

- **[Future]** Multi-line comments are deliberately deferred. If added later, the open design point
  is which delimiters to use given the explicit rejection of non-nestable `"""`, and whether a
  nestable block-comment form (for example paired markers that count nesting depth) is worth the
  lexer complexity in a codebase where line comments already cover most needs.

- **[Open]** The tabs-vs-spaces rule says mixing is disallowed "as Python", but the exact unit of
  disallowal is unresolved: is mixing forbidden per file, per block, or per physical line, and does
  a single line that begins with tabs then spaces (or vice versa) get rejected outright or
  normalized? For codec sources that are often edited across several tools, the precise rule
  determines how portable a file is between contributors' editors. (borrowed from Python, edge case)

- **[Open]** Dedent matching "must match a previous indentation level" needs a precise definition
  when tabs and spaces coexist across sibling blocks: what tab width (if any) is assumed when
  comparing a tab-indented outer block to a space-indented inner one, and is any tab-width
  assumption made at all, or are the two indentation styles compared only as literal
  whitespace-prefix strings? An ambiguous dedent in a deeply nested hot loop should fail loudly
  rather than silently reattach to the wrong block. (borrowed from Python, edge case)

- **[Open]** Statement-continuation edge cases around brackets and semicolons are only partially
  pinned down: whether an implicit continuation inside `()`/`{}`/`[]` can itself contain a
  semicolon, whether a trailing operator (for example a binary `+` at line end outside brackets)
  ever implies continuation the way some languages allow, and whether an explicit line-continuation
  escape exists at all for expressions that are not bracket-delimited. C programmers used to `\`
  line splices and Python programmers used to backslash continuation will both have expectations
  here. (allowed/disallowed syntax gap)

- **[Open]** How semicolon-joined statements interact with block openers is unspecified: can a line
  like `if cond: a = 1; b = 2` place both `a = 1` and `b = 2` inside the `if` body, and is a
  block-opening statement (one ending in `:`) allowed before or after a semicolon on the same line
  at all? The one-line `if` form is already blessed in `control-flow.md`, so the lexer's treatment
  of semicolons around the `:` needs to agree with it. (ambiguity, allowed/disallowed syntax gap)

- **[Open]** Ephemeral `_`-identifier carry-over (resolution #5) has subtle interactions that a
  reader will trip over: within an `if`/`elif`/`else` chain, does an assignment to `_` inside one
  branch reset it for the sibling `elif`/`else` conditions and bodies, or only for statements that
  textually follow within the same branch? The rule is written as a linear carry-over, but branches
  are not linear, so the reset point across branch boundaries needs to be stated exactly. (likely
  confusion, edge case)

- **[Open]** Ephemeral `_` in loop conditions raises a per-iteration question: in
  `for _=0.. while a[_]:` the ephemeral is assigned in the `for` clause and read in the `while`
  clause, but it is unclear whether a reassignment of `_` inside the loop body is visible to the
  next iteration's condition, or whether the condition always sees the value established by the
  loop's own assignment clause. This directly affects find-first-set and scan loops common in codec
  inner loops. (likely confusion, edge case; see `control-flow.md`)

- **[Open]** Nested ephemerals and multiple live `_`-names need a rule: `_lo, _hi = ..., ...`
  establishes two ephemerals at once, so it must be clear whether each ephemeral name is tracked
  independently for its own "until next assignment" window, or whether any assignment to any
  ephemeral resets the whole ephemeral namespace. Nested blocks that each assign their own
  ephemeral compound the question. (ambiguity, likely confusion)

- **[Open]** The distinction between a plain `_` and a `_`-prefixed named ephemeral (for example `_`
  versus `_tmp`) is not spelled out: is bare `_` a throwaway/wildcard with the same carry-over
  semantics, or is it special (discard target), and can two different `_`-prefixed names be live
  simultaneously without interfering? C and Python both give `_` informal conventions that readers
  will import by habit. (likely confusion)

- **[Open]** Shebang handling is only stated as "supported"; the compiler's exact behavior is
  undefined. It is unresolved whether the `#!` line is required to be the first physical line (and
  rejected elsewhere), whether it is passed through verbatim, stripped, or validated, and whether
  its presence affects line numbering in diagnostics. For codec tools shipped as runnable scripts
  this determines how the file is invoked. (allowed/disallowed syntax gap, edge case)

- **[Open]** String and character literal lexing is not described in the settled content at all:
  which quote forms exist (single, double, both), what escape sequences are recognized, whether raw
  or byte-string prefixes exist, whether multi-line/triple-quoted strings are allowed given that
  `"""` was rejected for comments, and how these literals relate to the `u8*`/vector types in
  `type-system.md`. Codec code handling headers and magic bytes needs a defined byte-literal story.
  (ambiguity, allowed/disallowed syntax gap)

- **[Open]** Numeric literal forms are unspecified lexically even though `type-system.md` fixes
  their default sign and size: it is open which bases are accepted (decimal, `0x` hex, `0b` binary,
  and whether C-style `0` octal is kept or dropped), whether digit separators (underscores, as in
  Rust) are allowed, what suffixes beyond `u` exist, and whether a `.`-bearing token is a lexical
  error given there is no floating-point type. (allowed/disallowed syntax gap; see `type-system.md`)

- **[Open]** The keyword list itself is not enumerated anywhere in the settled content, only the
  rules that keywords are globally reserved and case-sensitive. Which words are reserved (for
  example `pass`, `if`, `elif`, `else`, `for`, `while`, `break`, `continue`, `switch`, `case`,
  `default`, `goto`, `and`, `or`, `struct`, `union`, `pass`, `alt`) is an open gap, and because
  there are no contextual keywords, every reserved word permanently removes an identifier the codec
  author might otherwise want. (gap, likely confusion)

- **[Open]** How `@`-prefixed directives and attributes tokenize sits on the boundary between this
  chapter and `builtins-and-directives.md`: whether `@name` is a single token or an `@` operator
  applied to an identifier, whether whitespace is allowed between `@` and the name, and how this
  interacts with the identifier grammar. The precise lexical treatment affects whether directives
  can ever collide with identifiers. (ambiguity; see `builtins-and-directives.md`)

- **[Open]** Whether a maximum line length, maximum indentation depth, or maximum identifier length
  is imposed or merely conventional is undecided. The design ethos in `design-principles.md`
  discourages deep nesting, but it is open whether that is enforced lexically or left to style.
  (gap)

- **[Open]** The PEG grammar's audience decision (parser-generator only, not human readers) leaves
  open whether any human-facing grammar is published at all, or whether the reference docs are the
  sole human specification. If the two ever diverge, it is unclear which is normative for
  implementers building codec tooling around the language. (borrowed from Python, likely confusion)

- **[Open]** Unicode handling in identifiers, comments, and (eventual) string literals is
  unaddressed. The identifier regex `[a-zA-Z_][a-zA-Z0-9_]*` is ASCII-only, so it is open whether
  non-ASCII bytes are rejected everywhere outside comments, what source encoding is assumed (UTF-8
  presumably), and whether a byte-order mark is permitted before a shebang. (gap, edge case)
