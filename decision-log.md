# Design principles

- Choices that would result in concise code in my actual code base
- Quick to type -> prefer lowercase letters in language and standard library (no snake_case or camelCase), count in keystrokes (i.e. digits are 2 keystrokes with Maj)
- Explicit and readable operations, someone not familiar with the language should guess what it does
- Rely on familiarity with C
- Avoid hiding costly operations

# Don't forget

- directives to assert that: return reduces to tail call, instruction count is below threshold
- alternate execution block with equivalence validation
- check all GCC extensions and builtins
- struct field assert specifier
- features for debugging/printing?
- should attributes be before or after target? what elements should they apply to?
- describe struct reordering inputs and algorithm
- how can we formally validate that stack/heap does not run out of space?

# Misc

- Indentation: spaces and/or tabs (same as Python)
- Assignment operator: `=`
	- familiar to all programmers
	- pretty quick to type
	- can pair with arithmetic operators for combined operation+assignment
- Assignments are statements:
	- no expressions with side effects
- Comma-separated multiple assignments `a,b,c = 0,1,2`
	- straightforward expression of swap `a,b = b,a`
	- compatible with functions returning multiple values
	- facilitate multiple combined operation+assignment `a,b += 1,2`
- Ephemeral identifiers, prefixed with `_`, are scoped to the right half of next assignment (thus can be reused), ex. `_lo, _hi = ..., ...`
	- no need to find temporary names for long computations
	- easy to turn into named assignments
- Comments: `#` for single-line, no multi-line (no `"""` since not nestable)
	- supports shebang header
	- most editors support commenting in/out multiple lines
	- may add multi-line comments in the future
- Logical operators: `and`, `or`, `!`
	- quick to type
	- readable
	- `!` can be prefixed on a variable without a space
- Block structures: indentation-based (Python-style)
	- saves space
	- consistency between visual layout and actual structure
	- familiar to Python devs
	- no debate over brace placement
	- removing/adding blocks doesn't generate brace-only diff lines
	- deters deep nesting caused by complex branching code
- Semicolons: Optional, expressions may span multiple lines when delimited in parentheses/braces/brackets, statements after semicolon keep the same indentation level
	- familiar to both Python and C programmers
	- still easy to parse
- Memory management model: manual with type-aware alloc/free (storing type at negative offset) that work directly on memory parameters
	- easy to implement
	- no hidden memory or instrution overhead on pointers
	- familiar to C programmers
	- perfect FFI compatibility
	- dynamic allocation is really a minor thing in codecs
	- reference counting with copy-restricted pointer type may be added in the future when needed
- Indentation rules: allow tabs and spaces, disallow mixing both (as Python)
	- use tabs in code examples
- Inconsistent dedent handling: must match a previous indentation level (as Python)
- Empty lines within blocks can preserve indentation or be blank (as Python)
	- use and recommend the first style in code examples
- Multi-line expressions are indented by 2 levels deeper in code examples
- Grammar notation will be in PEG format (as Python)
	- easy to read and unambiguous
- Grammar targets the parser-generator rather than people reading it
	- people have the reference documentation
- No contextual keywords, all keywords are globally reserved
- Keywords are case-sensitive
- attributes and compiler directives are prefixed with `@`
	- `@constexpr expression` to require that the expression is resolved at compile-time
- Compiler extensions should be explicitly imported with `import "compiler" [prefixed comp.]` (optional prefix)
- Empty blocks are forbidden, `pass` must be used instead (as Python)
- Identifiers: `[a-zA-Z_][a-zA-Z0-9_]*`, leading underscores trigger ephemeral scope
- In binary operations, signs of operands must match, and the smaller operand is promoted to the size of the bigger if needed
	- no integer promotion ever changes sign
- Conditional blocks are `if expr: ...\nelif expr: ...\nelse: ...`
- Integers can be accessed like arrays `if mask[2]:`
- Bracket notation on vectors supports contiguous slices `a[0..3]` and `a[2..4..-1]` and shuffles `a[0,2,1,3]`
	- numbers cannot be omitted in slices
	- negative indices in shuffles yield 0, and `*` is a "don't care"
- Variables:
	- block scope, but stopping at function calls unless declared with `@stack` or `@alloc` specifier (i.e. register scope on a machine with infinite number of registers)
	- a new variable may not take the same name as a variable out of scope because of function but in the same block scope (to avoid confusion with C)
	- `a = 0` declaration and initialization (prefix type is forbidden)
	- `a = i8{*}` declaration without initialization (type is mandatory)
	- variables are immutable by default, unless declared with `@mut a` (solves the problem of not distinguishing declaration and assignment)
	- making a new definition for a immutable variable in same scope is forbidden (including variable shadowing)
	- may add the possibility to back a variable with a custom pointer in the future
- no header files, only a single type of source file, with import statements and modules
- variable defined in global scope makes global variable
- no support for bitfields at the moment since not needed, may add them in the future with iN/uN types

# Types

- Integer types: `i8`, `u8`, `i16`, `u16`, `i32`, `u32`, `i64`, `u64`, `isize`, `usize` (same as Rust)
	- slower to type than mixed letters but less frequent with type inference
	- more explicit sizes in structs
	- integer literals default to signed, except with `u` suffix
	- literal default size is the smallest that can fit the given value (with error if it overflows 64 bits)
	- use `i32.MIN` and `i32.MAX` to get integer type bounds
- No boolean type, no floating-point types
	- scalar comparisons return `i8` (-1 or 0), vector comparisons on `uN[M]` return `iN` (-1 or 0)
- Pointer type syntax: `T*`
	- familiar to C programmers
	- optional attribute `@stride{N}`, initialized to type size, will make `&a[i]` equal `a+i*N`
	- can query pointer stride with `T.STRIDE`
	- can query pointer alignment with `T.POINTED_ALIGNMENT`, initialized to GCD of stride and alignment of pointed type
	- use `a~[i]` or `a~.name` to request an unaligned access (note if the host machine has smaller registers, the unaligned access may be fragmented into multiple aligned accesses)
	- access should be unaligned when stride is not a multiple of type size, otherwise compiler will issue a warning
	- no separate null type handling, may be added in the future (ex. `&`)
	- no fat pointers either, may be added in the future
	- no `void*` equivalent, alias to `u8*` with external libraries
- Slices: `start..range..step`
	- internal type only valid inside `for` assignment, array index and `case` label
	- `start` is the index of the first element in the slice, and must be in the range of valid indices (when array size is known)
	- `range` is the number of elements in the slice (easier than `end` for slicing fixed-range vectors), and must be a compile-time constant greater than 0
	- `step` is the increment to apply to reach each next index, and must be a compile-time constant (possibly equal to 0 to get `range` times the same element)
	- write `start..range` when omitting `step` which will default to 1
	- `start` should always be present, even if zero
	- `range` may be omitted in `for` assignment only, with `start..` and `start....step`
	- with saturation specifier `start..|range`, every index in the slice is clamped to `[0,len(array)-1]` before subscripting
	- with wrapping specifier, `start..%range`, every index in the slice is modulo'ed to `len(array)` before subscripting
	- without an overflow specifier, out-of-array indices are undefined behavior
	- it is an error to use the saturation or wrapping specifier whan slicing anything other than an array with known size
- Vector type: `T[N]`
	- vectors are arrays of fixed size, and the only supported arrays at the moment (may add variable-length arrays in the future)
	- no alloca, end-of-struct `T[]`, or dynamic heap array yet
	- arrays of runtime size are directly reduced to pointers (i.e. compiler has no specific internal type and does not keep track of array size)
	- may add first-class dynamic heap array in the future with `T[+]`
	- cast-like initializer `i32[5]{0,1,2,3,4}`
	- can omit size when initializing `i32[]{0,1,2,3,4}`
	- `i32[5]{4}` initializes all lanes to one same value
	- C-style multidimensional support `v[i][j]` for `v+i*n+j`
	- slicing can only target the rightmost vector `v[i][0..4]`
	- indexing a vector with a variable requires it to have the `@stack` or `@alloc` attribute (or inherit from its containing struct)
	- vector indexing supports the overflow specifiers to wrap `v%[i]` the index to the vector bounds, or return 0 when out of bounds `v|[i]` (a more common hardware behavior than saturating the index inside bounds)
	- indexing vector with other vector `v[w]` does a swizzle operation, on the condition that both vectors have the same element type, returning a vector the same size as `w`, with support for wrapping and saturation specifiers too
	- vectors have a vector type (no decay to pointer like C), ex. `a = i32[4]{} # type i32[4]`
	- address of vector returns pointer to vector, ex. `&i32[4]{} # type i32[4]*`
	- take address of vector to cast to pointer of base type (must have `@stack` or `@alloc`) `i32*{&a}`
	- no decay to pointer when passing vector to pointer argument, ex. with `f(i32* p):...`, `f(i32[4]{})` will fail since expecting a pointer type and receiving an vector type
	- vector pointers can promote automatically to pointers of same base type, ex. `f(&i32[4]{})` works
	- vector pointers do not promote automatically to vector pointers of same base type and different size, ex. with `f(i32[2]* p):...`, `f(&i32[4]{})` fails
- Struct type: `struct MyContext:\n  i32 a\n  i32 b`
	- `ctx = MyContext{*}` declaration sets ctx with undefined values
	- automatic reordering based on size, usage frequency and code locality
	- `@extern` keyword prevents reordering and follows C padding rules
	- `@aligned{N}` on struct or field forces alignment (including decreasing, contrary to GCC)
	- may add a `@minaligned` attribute in the future if needed
	- `@extern MyContext*` in global scope signals the compiler to export an opaque pointer
	- use `@packed` on the structure to pack it like C (default alignment for all members then becomes 1)
	- use `@aligned{1}` on a struct member to pack it like C
	- anonymous `struct:` may be used inside a struct definition
	- `@order{n}` hint may be used to instruct compiler to put a certain field in a certain position
	- `..OtherContext oc` to include `oc` in the struct namespace as well as its fields
	- no encapsulation or notion of visibility
	- only designated initializer is allowed `MyContext{a=0, b=1}`
	- `union MyUnion:` and anonymous `union:` are supported to facilitate automatic conversion from C
	- `i32 MyContext.c = 10` in global scope adds a field to a struct with a default value (when initialized through `MyContext{...}`), useful to put constants to use when this struct is in scope, rather than through PC-relative addressing
	- may add explicit padding members `_` in the future
	- can query offset of member with `T.OFFSET_name`
- Type inference: Local inference with explicit function interfaces (arguments and return values)
	- `a = 4` with inference (`a = i8{4}` without inference)
	- `a = i32{4}` uses automatic promotion (`i8` to `i32`)
- Type qualifiers:
	- use `@restrict` qualifier to signal that a pointer will not alias any other
	- no support for mutability or constness of pointed values, only the variable can or cannot be assigned, may add `@readonly` and `@writeonly` directives in the future
- Type conversions:
	- type casts and initializations use the same syntax `T{a}`
	- implicit conversions are only integer promotions, the rest must be explicit
	- casts from any scalar to any scalar (pointers, integers) are allowed
	- cast of bigger to smaller integer assumes by default that the value is small enough, otherwise may yield any value on overflow
	- `T%{a}` casts with wrapping on overflow
	- `T|{a}` casts with saturation on overflow
	- it is an error to request wrapping or saturation when overflow is impossible or makes no sense on the given cast (ex. integer to pointer)
	- casts between vectors of same number of elements and element size simply apply cast to all elements (wrapping and saturation allowed)
	- casts between vectors of same overall bit size just reinterpret type without bit conversion, ex. `i32[4]{i8[16]{}}` (wrapping and saturation not allowed)
	- casts between vector of same number of elements promote or reduce elements (with care for overflow), ex. `i8[16]{i16[16]{}}` (wrapping and saturation only allowed when reducing size)
	- `signed{a}` and `unsigned{a}` to cast to signed or unsigned with same base type size (works for integers and vectors)
- Type system:
	- explicit type expression
	- nominal type compatibility and equivalence (two structures can be used interchangeably if they have the same type name)
	- static type checking
- can query type size with `T.SIZE` (in bytes) and `T.BITSIZE`
- can query the type of an expression with `typeof(a)`
- can query type alignment with `T.ALIGNMENT`

## Promotion matrix

This matrix gives the automatic promotion rules from row types to column types, i.e. when type A may be promoted to type B ("A->B" in the matrix) without requiring an explicit cast.
In this matrix, `isize` and `usize` are omitted and replaced by their `iN` and `uN` aliases depending on the target native bit width.
Two types are said to be compatible when at least one may promote to the other.
Two types are said to be equivalent ("A<->B" in the matrix) when both may promote to the other.

Promotion and compatibility are used for:
- binary operations that check at compile time whether two operands may be operated together
- function overloading that ensure that every overloaded function differs from other overloads by at least one incompatible input argument
- function call that tries to promote all input arguments to expected types
- the resolution of an overloaded function call, that matches the first overload for which all input arguments can be promoted to expected types
- assignments to already declared variables, that ensure the input value type can be promoted to the receiving variable type

|| iB | uB | B\* | B[N] | B[N]\* | struct&nbsp;B | union&nbsp;B |
|-|-|-|-|-|-|-|-|
| iA | A<=B | no | no | no | no | no | no |
| uA | no | A<=B | no | no | no | no | no |
| A\* | no | no | A<->B **and** A\*.POINTED_ALIGNMENT % B\*.POINTED_ALIGNMENT == 0 | no | A<->B **and** A\*.POINTED_ALIGNMENT % B[N]\*.POINTED_ALIGNMENT == 0 | no | no |
| A[M] | no | no | no | A<->B and M==N | no | no | no |
| A[M]\* | no | no | A<->B **and** A[M]\*.POINTED_ALIGNMENT % B\*.POINTED_ALIGNMENT == 0 | no | A<->B **and** A[M]\*.POINTED_ALIGNMENT % B[N]\*.POINTED_ALIGNMENT == 0 | no | no |
| struct&nbsp;A | no | no | no | no | no | A==B | no |
| union&nbsp;A | no | no | no | no | no | no | A==B |

# Operators

- no custom operators, no operator overloading
- integer division truncates toward 0 (same as C)
- modulo matches C too
- Signed AND unsigned integer overflow is undefined behavior unless explicitly allowed with saturation `|` or wrapping `%`
- division by zero is undefined behavior (including undefined value and crash)
- may add a third variant in the future returning the value and a boolean exception test, ex. `res, err = a +? b`, `res, err = i8?{a}`, `res, err = a /? b` (note `/` can also overflow, and with vectors err is a vector of same size)
- right shift `a <<Ω b` has overflow specifier on the right targeting shift amount, with wrapping taking `b % width`, and saturation clearing the result when `b >= width`
- left shift `Ω<<Ω` adds an overflow specifier on the left targeting result, with wrapping discarding shifted bits, and saturation clamping the exact result to output type bounds
- unsigned specifier `$` or its absence for binary operators acts as an assertion, and must match signedness of both arguments (except right shift for which specifier applies to left argument)
- no rotate operators
- builtin functions for bit counting `popcount`, `ctz`, `clz`
- negative values are defined as two's complement representation, thus compatible with bitwise operations
- logical operators `and`, `or` do short-circuit
- ternary condition `if(cond, t, f)` does not short-circuit
- logical operators work on integers and pointers (as in C), not on vectors, structs
- chained comparisons are supported (same as Python)
- pointer comparisons are allowed withing the same object (same as C)
- whole vectors/structs comparisons must go through memcmp, to avoid hiding a costly operation
- pointer arithmetic is allowed, with no checking as in C
- no dereferencing operator, use `T[0]` instead
- `->` is used to access struct fields through a pointer (same as C, no merging with `.` to avoid making the inner working unclear)
- assignments are statements, not expressions
- no chained assignments, use `a, b, c = d` to assign one value to multiple variables
- no increment/decrement operators
- no bounds checking on vector indexing (task for formal validator)
- order of evaluation in an expression follows operator precedence, with left-to-right evaluation inside each operator

## Operator precedence

Arithmetic and binary operators are split in two precedence groups, multiplicative and additive. Since shifts are equivalent with multiplication and division, and bitwise AND can do modulo, they are put in the multiplicative group. The rest is additive. No further distinction is made, as best practice is to settle ambiguities with parentheses anyway.

In all operators:

- `Δ` denotes an alignment specifier, which may be `~` (unaligned), or absent (aligned).
- `Ω` denotes an overflow specifier, which may be `|` (saturation), `%` (wrapping), or absent (undefined). 
- `Σ` denotes a signedness specifier, which may be `$` (unsigned), or absent (signed)

| Operator | Description |
|-|-|
| `(x)`, `{x}` | expression in parentheses or braces |
| `xΔΩ[index]`, `xΔ.attr`, `xΔ->attr`, `x(args)`, `xΩ{args}`, `x*` | **Suffixes**: indexing, struct offset access, struct pointer access, call, type cast, pointer type |
| `+x`, `-Ωx`, `~x`, `!x`, `..x`, `..?x`, `@attr x` | **Prefixes**: positive, negative, bitwise NOT, logical NOT, key iterator, non-zero key iterator, attribute before an identifier |
| `*Ω`, `/Σ`, `%Σ`, `Ω<<Ω`, `Σ>>Ω`, `&` | **Multiplicative**: multiplication, division, remainder, shifts, bitwise AND |
| `+Ω`, `-Ω`, `++`, `^`, `|` | **Additive**: addition, subtraction, concatenation, bitwise XOR, bitwise OR |
| `<Σ`, `<=Σ`, `>Σ`, `>=Σ`, `==`, `!=` | **Comparisons** |
| `and` | Logical AND |
| `or` | Logical OR |

# Control flow

- all control flow blocks are statements, not expressions, thus do not return values like expressions
- if statement:
	- a block may fit on one line `if condition: statement(); statement()`, though a second block must start on a new line (ex. `else:`)
	- does not include assignment slot like `for`, use ephemeral assignment if needed `_ = expression()\nif _ < threshold:`, `_` will be available to `elif` conditions too
- loop statement: `for assigment() while condition():`
	- both `for` clause and `while` clause are optional, but cannot swap
	- `while 1:` infinite loop
	- `for i=0..10:` range loop
	- `while valid:` while loop
	- `for i=0.. while a[i]:` optional condition
	- `for _=0.. while a[_]:` ephemeral identifier carries to the condition
	- `for i,j=0..10,a..b:` zip indices
	- `for i=..vector:` iterate on indices of vector
	- `for i=..?vector:` iterate on indices of non-zero values (or set bits of integer)
	- `break [n]` inside a loop, break from a given number of loops (default 1)
	- `continue [n]` inside a loop, continue from a given number of loops (default 1)
	- `0..10`, `..vector` and `..?vector` return internal iterator type that can only be read from, not assigned
	- supports loops over bitmasks
	- supports find-first-set SIMD pattern
	- do-while can be obtained with `@passonce` attribute
	- as with if statement, loop variables have block scope
- Branch table `switch i:`:
	- this is both a jump table based on a single integer value, and an area in the code where `goto` jumps are available (only within the table)
	- the table may only contain cases and labels, statements may be found only inside cases and labels
	- `case 0:` for a target bound to a value
	- `case 0..10,12:` for cases with multiple values
	- variables declared in a case are scoped to that case (cross-variables must be declared outside)
	- `default:` for the label to reach when no case matches
	- `goto case 0` inside the table to jump to a given case
	- no fallthrough behavior, use `goto case ...` to jump to the next case
	- with `switch:` the control flow jumps directly at `default:`
	- `label:` and `goto label` for custom jump targets (still block-scoped)
	- `break` does not affect `switch`, only the enclosing loops
	- use `@unreachable` in the default statement to signal that all possible cases are covered
- functions can have multiple returns
- no defer statement (incompatible with no-function scope), context manager (like `with` in Python), or RAII (like C++)
- may add RAII or context manager in the future
- `@assert{condition, message}` with optional message, puts a condition:
	- not checked in release mode
	- checked at runtime in debug mode, displaying the message otherwise
	- checked in validation mode, displaying the message otherwise
- `@assume{condition}` puts a condition:
	- not checked in release and debug mode
	- used as a hint to prune search paths in validation mode
- Alternate blocks `alt: ... else: ... else: ...`
	- Allows expressing multiple optimized versions of a same code block
	- Compiler is free to choose fastest version for each host target
	- in debug mode, option chosen is compared with first `alt:` block at runtime
	- in validation mode, compiler formally checks that all blocks have the same effects outside their scopes

# Functions

- Function definition: `name(T a, T b) -> (T c, T d):`
- Return value names are optional, for documenting only
- parentheses around return types may be omitted for one argument, and arrow can be omitted for zero argument
- Corresponding type: `(T, T) -> (T, T)` (cannot omit arrow nor parentheses)
- no forward declarations, each function is declared and defined at the same time -> functions can be called before they are defined
- Defining a function pointer type: `type MyFunc = (T, T) -> (T, T)`
- When assigning from a function with multiple return values, any value may be ignored with `a,*,b,* = function(...)`
- One may also access the function result with a compile-time index `a = function(...)[0]` (but no slicing since return values are not necessarily of the same types)
- assignment is the only way to get all values from a multiple-return function, no support for expanding in another function at the moment, may add in the future if needed
- Two functions with same name but different arg types are allowed if both have `@overload` specifier
- On call to overloaded function, the first compatible in definition order is chosen (with automatic promotion if needed)
- no support for generics at the moment, may add them if needed
- `return a, b` to return one or multiple values (must match function signature)
- `return` to return from a procedure (optional)
- functions have their own namespace, `f = @function name` to reference them
- Every function is already a pointer, so `&@function name` is an error
- no lambda functions (not a functional language)
- no nested functions
- arguments are passed by value by default (integers, pointers, structures, vectors)
- Things of variable length (at the moment only strings) cannot be passed as-is (by value or reference) and must be lowered to pointers
- `name(T @ref a) -> (...):` to support passing a parameter by reference (or variable) -> when returning from the function, the last value of `a` will carry on to the variable that was passed to a, and its scope will extend past the function call (no matter if it has attribute `@stack`/`@alloc`)
- passing by reference or variable can be thought as if the function receives the variable's value in a certain register, and returns the new value back in the same register
- when a parameter has `@ref` it can be passed by value or reference, as the choice of the caller
- `name(b)` to call the function and pass b by value, making a copy if necessary
- `name(@ref b)` to request passing `b` by reference, failing to compile if the target parameter does not specify `@ref`
- an expression that is not assignable can only be passed by value, ex. `name(4)`
- arguments passed by reference have block scope inside the function
- put `@unreachable` at the end of a function to signal it never returns (it counts as an assert, thus ignored in release mode, checked at runtime in debug mode, and checked in validation mode)
- No automatic promotion among function pointers, parameter types must match along with their attributes
- `@noinline` or `@alwaysinline` may be used on function definitions (default inlining strategy per function) or calls, to prevent or require inlining
- `@alwaysinline` fails when definitely impossible to comply (ex. due to call to function pointer, recursive function, or library function), and issues a warning if it detects that LLVM reintroduced subfunctions
- no `@inline` hint at the moment, may add an attribute in the future to tell in which order the compiler should try inlining calls (i.e. ordering calls by decreasing impact if inlined)
- inlining works across modules for functions defined inside modules
- `@tailcall` on statement containing function call warns if the last function call cannot be converted into a tail call
- may add FFI layer with automatic import of C header files with `import "library.h"` in the future
- `@extern` function attribute automatically sets the default C call convention on the host system, otherwise the language uses a custom optimized convention
- `@ref` arguments are only supported with the internal custom convention
- may add a function attribute `@callconv{"..."}` in the future to use other calling conventions
- `asm(...)` statement takes the same syntax as GCC and allows inserting inine ASM
- no support for creating variadic functions, can only call them
- no default values for parameters since no need for them
- `generic T = i8x16, i16x8, i32x4, i64x2:` to define functions multiple times with T replaced by each given type (functions automatically have `@overload` attribute)
- example of generic function: `ziplo32(T a, T b) -> i32x4: return (i32x4%{a} ++ i32x4%{b})[0, 4, 1, 5]`
- `generic T, U = (i8, i32), (i16, i32), (i32, i32):` supports iterating with multiple generic types (here it will iterate 3 times)
- `generic` blocks cannot be nested
- use attributes `@hot` and `@cold` as hints to compiler

# Safety model

- explicit choice of checked or UB bounds checking on array access
- UB on null pointer dereferences
- explicit choice of checked or UB integer overflow for signed and unsigned
- UB on uninitialized memory reads
- UB on reinterpreting a value with different type
- however formal validation is available to ensure that no UB can actually happen:
	- for each unit of test (usually a function), preconditions are defined on inputs that the validator can expect to hold
	- the validator checks that with these preconditions no UB can happen
	- it also checks the equivalence of all alternate paths in `alt:` blocks
	- finally it checks postconditions on the outputs
- This is a different approach to Rust. Programmers are given more freedom (ex. possible to have 2 mutable pointers to a same object), but have to manually specify the contracts between functions.

# Memory model

- Memory layout
	- Structures follow C layout rules with automatic padding for alignment
	- Members of a struct are laid out in any order the compiler chooses, unless `@extern` is defined on the struct
	- no support for big-endian hosts at the moment, may add endianness support in the future
	- the language guarantees bit-for-bit compatible struct layout with C when the attribute `@extern` is present (it actually exports a C struct)
	- packed C attribute is automatically inserted on `@packed` struct, and on members for which alignment is forced smaller than default
	- default alignments:
		- integer = its size (same for extern)
		- pointer (for the pointer itself, not the pointed data) = its size (same for extern)
		- struct = the max of alignments of all its members (same for extern)
		- array = the GCD of biggest host SIMD size and array size (different than extern)
		- union = the max of alignments of all its members (same for extern)
	- there is no maximum for alignment
	- language constant `BIGGEST_ALIGNMENT` gives the largest alignment ever needed for any CPU operation on the target machine
	- padding bytes have undefined values, both at initialization and when accessed (not UB though, they cannot crash the software)
	- no runtime introspection of type layout, the closest to obtain is the layout exported for debugger
- Pointer semantics
	- pointer arithmetic is not allowed, only indexing allowed
	- function pointers only allow calling and equality testing, not indexing or order comparison
	- the language does not keep track of pointer provenance
	- no restrictions on explicit pointer-to-pointer casts
	- pointer-to-integer casts are only allowed to/from `usize` or `isize`
	- automatic promotion between pointers in only allowed on same pointed type and same stride (could be more permissive but needs to be explicit)
	- comparison of types is recursive and nominative (not structural)
	- only promotion-compatible pointers can be compared
	- `null` is guaranteed to be all zeros (same as C)
- Allocation and lifetime
	- theoretical machine has a stack and heap, and a bank with infinite registers each able to hold a value of any size
	- each value is allocated in a register by default, unless declared with attributes `@stack` or `@alloc`
	- register storage has register scope, stack storage block scope, and heap storage global scope
	- compiler may allocate on stack or heap silently if out of space in any shorter scope
	- compiler does not keep track of the lifetimes of heap objects
	- no delete methods, no support for dangling pointers, no move semantics, no borrowing rules
	- no ownership mechanism as opposed to Rust, so all pointer uses are acceptable (ex. two mutable pointers to the same data), but on the flip side require explicit descriptions of interfaces for the validator to check that no memory violations happen
- Memory management
	- `p = @alloc T{...}` returns a `T*` allocated on the heap, while taking care of alignment and initialization
	- `@free p` calls `free` on the given l-value and sets it to `null`
	- can still use `malloc`/`free` and other functions from standard C library if needed
	- failed allocation returns `null`
	- no garbage collection, no detection of circular references, no arena-based allocator
	- may add support for alternate allocators in the future
- Aliasing and Type Punning
	- any pointer may alias any other pointer regardless of their type, it is the compiler's duty to ensure that reordering their accesses has no impact
	- use `@noalias` on a pointer to assert that is aliases no other pointer
	- type punning (accessing one same memory address with a different pointer type) is allowed with casting or unions
- Concurrency Memory Model
	- available through stdatomic.h
	- default memory ordering is relaxed
	- may add language support in the future when my needs are clearer
- Volatile semantics
	- no support at the moment, may add if needed in the future
- Undefined Behavior
	- same as C unless otherwise specified
	- signed and unsigned integer overflow are UB without overflow specifiers
	- memory space is modeled as a circular buffer, i.e. addresses overflow with wrapping, and pointer comparisons are done with wrapping (i.e. if `a=0xffffffff` and `b=0` then `a<b`)
	- out-of-bounds pointers may be created and used in comparisons, only accessing them is UB
	- `null` pointer dereference, out-of-bounds array access, use after free, double free, uninitialized variable read, type punning through incompatible pointer casts, data race, are all UB
	- there should be a comprehensive UB catalog
	- all UB will be checked with formal validator
- Cache and SIMD Alignment
	- alignment of object storage (on stack or heap) is specified with `@aligned{N}` in front of the type
	

# Inconsistency resolutions (session-pause review)

Resolutions to contradictions found earlier in this document. Where a resolution conflicts with a statement above, the resolution wins.

1. **Pointer arithmetic is NOT allowed.** Only indexing is permitted: `p[i]` and `&p[i]`. Writing `p + i` is illegal. The equivalence `&a[i] == a + i*N` in the `@stride` description is explanatory only, not valid syntax. (Supersedes the Operators-section claim that "pointer arithmetic is allowed as in C".)

2. **Pointer <-> integer casts are restricted to `usize`/`isize`.** A pointer may only be cast to/from `usize` or `isize`, not arbitrary integer types. (Narrows the Type-conversions "any scalar to any scalar" statement.)

3. **Cross-object pointer comparison is well-defined** under the global wrapping (circular address space) model. Out-of-bounds pointers may be created and compared; only dereferencing them is UB. (Supersedes the "within the same object, as in C" statement.)

4. **Bounds checking on array/vector indexing depends on the overflow specifier:**
- No specifier -> no bounds checking in release mode (out-of-bounds access is UB); checked in validation mode by the formal validator.
- Wrapping `%` (e.g. `v%[i]`) -> index wrapped to the array bounds.
- Saturation `|` (e.g. `v|[i]`) -> out-of-bounds access returns 0.
(Reconciles "no bounds checking (validator's task)" with "explicit choice of checked or UB": the explicit choice is expressed through overflow specifiers.)

5. **Ephemeral (`_`-prefixed) identifier scope:** once assigned, an ephemeral identifier carries over all statements and branches that follow it (including a following `if`/`elif`/`else` chain and loop conditions), until the next assignment statement, at which point it is available in that assignment's right-hand side and is then reset (reusable).

6. **OPEN QUESTION - `@alloc` overloaded meaning.** `@alloc` is currently used both as a variable storage-class attribute (`@alloc a` -> heap/global scope) and as a heap-allocation expression (`p = @alloc T{...}`). Whether to keep this dual meaning or rename one use is undecided.

7. **Pointer promotion rule: the promotion matrix is authoritative.** `A* -> B*` is allowed when the pointed types are equivalent (`A<->B`) and `A*.POINTED_ALIGNMENT % B*.POINTED_ALIGNMENT == 0`. (Supersedes the memory-model phrasing "same pointed type and same stride".)

8. **Typo corrections (confirmed):**
- Right shift operator is `>>` (the Operators bullet showing `a <<Ω b` for right shift is a typo).
- Vector comparison of two `uN[M]` operands returns `iN[M]` (a per-lane result), not scalar `iN`.
- The no-dereference-operator note should read "use `p[0]`" (indexing the pointer value), not `T[0]`.
