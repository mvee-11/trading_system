# C++ Features Needed for the Order Book

**Status:** reference material
**Audience:** leetcode-level C++ (STL basics, no systems programming), targeting slices 0–4 of `docs/plan_order_book.md`
**Contents:** topics and prose only — no code samples. Signatures and identifiers appear in backticks; the mechanics of each construct are described rather than shown.

---

## How to read this document

C++ knowledge is not a flat list of features. It falls into three tiers, and the tier matters more than any individual topic.

| Tier | Meaning | Cost of skipping it |
|---|---|---|
| **1. Won't compile** | Syntax and library names | Build errors, immediately visible |
| **2. Compiles but is wrong** | Undefined behaviour, lifetime, aliasing | Silent corruption, wrong answers, undebuggable |
| **3. Compiles, works, slow** | Layout, move semantics, allocator behaviour | Uninteresting in interviews |

**Tier 2 is the entire gap between leetcode C++ and systems C++**, and it is where this project lives. `cancel()` is pointer surgery on a linked list — if you do not own object lifetime, the resulting bug is a dangling pointer that surfaces thousands of operations later. Tier 3 is what separates a working system from an impressive one.

The rest of this document is organised by tier, then by the slices that need each part. Section 2.x is the substance; everything else can be skimmed and revisited.

---

## Tier 0 — Build basics (needed before slice 0)

Not C++, but nothing runs without it.

### CMake

The functions the project actually uses, and what each one does:

- `add_library` — declares a library target from a list of source files
- `add_executable` — declares an executable target
- `target_include_directories` — attaches an include search path to a target. The `PUBLIC` keyword propagates the path to anything that links this target, which is how consumers see the headers in `include/`
- `target_link_libraries` — declares a dependency between targets, and the scope keyword (`PUBLIC` / `PRIVATE` / `INTERFACE`) controls whether that dependency propagates onward
- `option` — declares a build option with a default, settable from the command line
- `find_package` — locates an installed dependency
- `FetchContent` — downloads and builds a dependency at configure time rather than using an installed one
- `enable_testing` / `add_test` — registers tests so `ctest` can find them

The important concept underneath: modern CMake operates on *targets*, not on directories or global variables. Each target declares its own sources, its own include paths, its own libraries, and its own compile options. A `PUBLIC` include path on `ts_book` is what lets `unit_tests` write `#include <ts/book/order_book.hpp>` without configuring anything itself.

### The warning flags

The project compiles with `-Wall -Wextra -Wpedantic -Werror` (`CMakeLists.txt:15`). The first three enable warning categories; **`-Werror` promotes every warning to a hard error**.

The practical consequence: a single unused variable, a signed/unsigned comparison, or a missing field initialiser will *fail your build* rather than print a note. This is deliberate — it keeps warning debt at zero, and warnings you cannot ignore are warnings you are forced to understand. Expect to hit roughly ten of these in the first week. Learn to read them; do not look for a flag to silence them.

The categories that matter most here:

- `-Wunused-variable`, `-Wunused-parameter` — code that does not do what its signature implies
- `-Wsign-compare` — comparing a signed and an unsigned value, where the comparison does not mean what it looks like it means. This is a real bug class, not just a warning
- `-Wreorder` — members initialised in a different order than declared. Has no visible effect here, but the compiler is right and the code is a lie
- `-Wold-style-cast` (under `-Wextra`-adjacent settings) — C-style casts bypass type safety

### Build types and generators

`CMAKE_BUILD_TYPE` selects the optimisation and flag set. **Single-configuration generators** (Ninja, Makefiles) do *not* default to anything — you get no optimisation flags at all unless you set it. This is why the project's sanitizer logic keys off an explicit `Debug` check rather than the generator.

The existing `build/` cache is already configured as `Debug` with Ninja, so ASan and UBSan turn on automatically (`CMakeLists.txt:26-32`). `compile_commands.json` is exported (`CMakeLists.txt:12`) for clangd and for tooling that needs to know the exact flags.

---

## Tier 1 — Core language (skim, fill gaps)

Most of this is already familiar. Scan for holes.

### Types and declarations

- **Primitive types**, and the promotion/conversion rules that apply when you mix them
- **The `<cstdint>` fixed-width set** — `int8_t`, `int64_t`, `uint64_t`, and friends. Unlike `long`, whose width is platform-defined, these are exactly the width their name says on every platform. This is why the order book uses them: a price's width is a protocol decision, not an accident of the machine
- **`using` type aliases** — a readable name for an existing type. The order book uses these pervasively so that `Price` and `Qty` are distinguishable at every call site
- **`typedef`** — the older form, same effect. Prefer `using`; it works for templates in ways `typedef` does not
- **`const`, `constexpr`, `consteval`** — three different promises. `const` means "will not change through this name". `constexpr` means "can be evaluated at compile time" and is a guarantee about the *value*, not just immutability. `consteval` forces compile-time evaluation. The project needs `constexpr` for sentinel constants and for the `static_assert` conditions
- **`enum class`** — a *scoped* enum. Plain `enum` leaks its enumerators into the enclosing scope and implicitly converts to an integer; `enum class` does neither. It also accepts an explicit underlying type, which here is a single byte. The dispatch on it later uses a `switch`
- **`auto` and type deduction** — worth using when the type is already obvious from the initialiser, not when it hides something
- **`decltype`** — the type of an expression, for when you need a type without a value
- **`static_assert`** — a check evaluated at compile time. Used here to pin down the size and alignment of the order struct, so that a later field addition that breaks the padding fails the build instead of silently costing performance
- **Alias templates** — a `using` that introduces a template alias. Not needed yet, but you will meet them in modern standard library code

### Functions

- **Declaration versus definition** — a declaration promises; a definition delivers. The order book splits these across `.hpp` and `.cpp` (see section 2.3)
- **Parameter passing**: by value (copy), by reference (`&`, aliases the caller's object), by const reference (`const T&`, aliases without copying and forbids mutation), by pointer (may be null, may be reseated, obscures the direction of aliasing)
- **Return by value versus by reference** — returning a reference extends the lifetime of a temporary that is already gone. A classic and quiet source of dangling references
- **`const` member functions** — a distinct overload, not a qualifier on the return type. See section 2.4
- **`noexcept`** — a specification, and part of the function's type. See section 2.9
- **Overloading** — distinguished by parameter types, never by return type alone
- **Default arguments** — must be trailing, and are substituted at the call site, so changing a default changes behaviour for every caller
- **Recursion and stack depth** — nothing in this project recurses, but the stack limit is worth knowing

### Control flow

- `if` / `else`, `for`, range-based `for` (which works on anything with begin/end)
- **`switch` on an `enum class`** — the message dispatch in the program's main loop. Each `case` needs a `break`; forgetting one is a silent fallthrough
- `break`, `continue`, the ternary operator
- Order of evaluation of operands in an expression — unspecified in many cases, which matters if two operands have side effects

### Namespaces

- `namespace` declaration, nested namespaces, and the C++17 form that defines a nested namespace in one step
- Namespaces are *not* hierarchical for lookup purposes — a nested namespace does not automatically see names in its enclosing namespace without qualification, which is why `ts::book` types are written out in full
- **Why this project needs them**: `ts::book` and `ts::feed` will both have a `Price` type, and `bench` will have its own. Namespacing is the only thing keeping them distinct, and it is the compiler enforcing it

---

## Tier 2 — What leetcode never taught you

This is the real work. Each section states the mental model, why the language has the feature, and what specifically goes wrong in this codebase.

### 2.1 Object lifetime and storage duration

Every object in a running program has three separate properties: an **identity** (its address), a **type**, and a **lifetime** (the span of program execution during which it exists as that object). Confusing these is where most serious C++ bugs start.

The lifetime question has three parts — when the object begins, when it ends, and who decided. This is exactly where the four storage durations differ.

- **Automatic** (stack): begins at its declaration, ends at the close of the enclosing scope. The compiler decides, cleanup is free, and there is nothing to get wrong.
- **Static** (namespace-scope, function-local `static`): begins before `main`, ends after it. Initialised once, lives forever.
- **Dynamic** (heap): begins when you create it, ends when you destroy it. **You** decide both. This is the only duration with a failure mode.
- **Thread-local**: one instance per thread. Deferred to week 9.

The distinction that actually matters here: **an object's lifetime and the storage it occupies are not the same thing.** Destroying an object ends its lifetime, but the memory still exists and still holds the old bytes. That memory is now garbage — it looks like a valid object and is not one. Reading through a pointer to it is a dangling access, and it may still show you the values you remember writing, which is what makes the bug so confusing.

A third case that does not exist in your head yet but will by week 5: you can end one object's lifetime and begin a *new* object's lifetime in the same storage. That is what an object pool does when it hands a block back out. It is not a resurrection of the old object — it is a different object that happens to occupy the same bytes. Getting this wrong means the old object's state leaks into the new one.

**Applied to the order book.** Every `Order` is dynamic, so the book is responsible for ending its lifetime exactly once. The `PriceLevel`s live inside the maps, so the maps own them. The raw pointers inside a level's linked list are **non-owning** — they point at storage someone else owns, which means the list is not a second owner and must never free anything it points to.

Every pointer you hold into the book must have an answer to two questions: *who destroys this?* and *have they done it yet?* If you cannot answer both about a pointer, it is a bug waiting for a specific sequence of operations to trigger it. When `cancel()` unlinks an order and frees it, every pointer that still refers to it — in another level, in a local variable, in a future ordering — is now dangling, and nothing will warn you.

### 2.2 Undefined behaviour

Undefined behaviour is not "unspecified result" or "platform-dependent result". It is a **formal promise you make to the compiler** that a certain situation cannot occur, and in exchange the compiler is licensed to generate code that assumes it does not. When the promise is broken, the compiler is not required to do anything sensible — including discarding the code containing the break, entirely.

This exists because it is the licence that makes C++ fast. For a pointer dereference: if the compiler may assume the pointer is valid, it can hoist the load out of a loop, eliminate a redundant bounds check, or hold the value in a register across what would otherwise be a function call. All of that is only legal under the assumption.

The practical consequences follow directly:

- The same program can be correct at low optimisation and wrong at high optimisation, because a different code path exposed a different symptom.
- It can pass every test and fail in production, because the test never exercised the path.
- Sanitizers catch only the classes they were built to instrument. A clean ASan run is **evidence, not proof**.
- The symptom is often far from the cause. A write through a dangling pointer corrupts whatever else later occupies those bytes, so the visible failure may be an unrelated object, minutes into a run.

The specific forms you will meet in this project:

- Reading or writing through a pointer to destroyed storage
- Reading a variable that was declared but never assigned
- Indexing outside a container's bounds
- Dereferencing a null pointer
- Accessing storage through a pointer of insufficiently strict alignment
- **Signed integer overflow** — this one is worth emphasising. Signed overflow is UB, *not* wraparound, and the optimiser is entitled to assume it cannot happen. A quantity guard you believed you had is not necessarily a guard the compiler respected

**The diagnostic habit worth building:** when a bug appears and disappears after adding or removing one unrelated line, suspect UB before suspecting logic. That symptom is almost never caused by the line you just added.

### 2.3 Headers, linkage, and the One Definition Rule

This is the sharpest day-one change from leetcode, where everything is one file. Here there will be roughly fifteen, and three rules govern how they combine.

A **translation unit** is one `.cpp` file. The compiler processes each independently and emits an object file. The **linker** then merges them into a program. When the compiler processes your file, it has no idea what is in the others.

A **declaration** announces that a thing exists. A **definition** allocates storage for it or supplies its body. A program may contain any number of declarations of an entity, but exactly **one** definition. This is the **One Definition Rule**, and violating it is undefined behaviour rather than a guaranteed error.

**Headers are not compiled units.** An `#include` is *textual substitution* — the header's contents are pasted into your `.cpp` at that point and compiled as part of it. So if a header contains a *definition* of a plain function and ten `.cpp` files include it, the linker sees ten definitions and rejects the program. This is why functions defined in a header must be marked `inline` — and `inline` does not mean "the compiler may optimise this", it means "this definition is permitted in multiple translation units; the linker keeps one and discards the rest." Templates and `constexpr` entities get this treatment implicitly, for the same reason.

**Linkage** classifies how widely a name is visible:

- *External* — other translation units may refer to it
- *Internal* — confined to its own translation unit, which namespace-scope `static` requests
- *None* — the name is only visible inside the entity itself (class members, function-local statics)

`const` at namespace scope is a special case with a history people trip over: it has internal linkage by default, so a plain `const` in a header gives every translation unit its own copy. That is why the project's constants are `constexpr`.

Forward declarations — declaring a type or function without its definition, so it can be named before it is complete — reduce header dependencies and compile time. Worth using once the header graph gets non-trivial.

**Practically:** expect a duplicate-symbol link error during slice 0, and expect the cause to be a missing `inline` on a header-defined function or a missing `static` on a file-local helper. Both are the same mistake: a definition where a declaration belonged.

### 2.4 `const` correctness

`const` is a **promise to readers and a licence to the compiler**. It is not memory protection — the underlying storage is entirely mutable, and `const` says nothing about thread-safety or about what other aliases can do.

A `const` **member function** is not a qualifier on the return value. It is a property of the function, and a `const` and non-`const` version of the same signature are two *different* functions that can coexist and be overloaded. This matters because the distinction tells the compiler the function does not modify the object, which unlocks optimisations: repeated reads of a member can be cached in a register, and reordering across a call becomes legal. Part of why `best_bid()` can be both `noexcept` and `const` and fast is this.

The subtle part is the gap between **logical** and **physical** constness. A function is logically const if it changes no *observable* state, physically const if it changes no bytes. These usually coincide, but a `mutable` member or a pointer-to-non-const reachable from the object lets a logically-const function physically modify something. That is legitimate and occasionally necessary, but it is a hole in the promise, so it should be rare and commented.

`const` propagates through pointers and references. Two failure modes to know: forgetting `const` on a by-value parameter, which silently copies and then refuses to help you; and over-constraining an interface so callers cannot move out of what you hand them.

**The payoff here is structural, not stylistic.** A class where every query is `const` and every mutation is a non-const method taking an `OrderId` is a class whose thread-safety story is one sentence: *readers do nothing but read, one designated writer does all mutation, and the interface prevents a second writer from appearing by accident.* That sentence is the plan's single-writer rule made compiler-enforced — which is why the pointer is written without `const` even though it is only read. The alternative, returning `const Order&`, blocks callers from caching the pointer for their own use.

### 2.5 Memory layout, alignment, and padding

A struct's footprint is decided by the compiler, and the arithmetic you would do by hand is usually wrong. Fields are placed in declaration order, padding is inserted between them so each field's address satisfies its alignment requirement, and the total size is rounded up to a multiple of the struct's own alignment. **Adding one field can inflate the struct by far more than that field's width** — which is exactly why the plan's exercise fixes a deliberately-padded struct and watches the size change.

**Alignment** is the requirement that an object's address be a multiple of some value. A primitive type's natural alignment is tied to its size. **Cache lines** are the unit of memory the CPU actually transfers from L2/L3/DRAM — 64 bytes on x86-64 — and one line transfer brings back all 64 bytes whether you wanted one or all of them. `sizeof` is how much storage an object occupies; `alignof` is the alignment it requires; the two differ whenever padding is involved.

`alignas` raises a type's alignment above its natural value. It does not make access directly faster — it makes *sharing* more expensive and more predictable. When two objects used by two different threads occupy the same cache line, each core's write invalidates the line in the other core's cache, and the line shuttles back and forth on every access. Throughput can collapse by an order of magnitude even though each thread is doing trivial work. This is **false sharing**, and it is the entire reason the `Order` struct is cache-line aligned and why a `static_assert` on its size is worth writing down: the padding becomes a decision someone made and can defend, rather than a side effect of field order that silently changes when the next person adds a field.

**Honest scoping note:** doing the alignment now is correct, but it cannot be *justified with numbers* until benchmarks exist. Claiming a speedup now would be dishonest in exactly the way the plan is trying to prevent.

One more layout idea to carry forward: **data locality**. Contiguous objects fetched together are far cheaper than scattered ones, because a line fetch brings 64 bytes of neighbours for free. This is why the plan calls the subject "data-oriented design" — the layout of the data usually matters more to latency than the cleverness of the algorithm.

Endianness belongs here too, and becomes real at week 6 when bytes arrive off a wire: the protocol is big-endian (network byte order) while your CPU is almost certainly little-endian, and the conversion has to be explicit.

### 2.6 Value categories, copying, and moving

Every expression is one of four categories, distinguished by whether it names something that persists or names a temporary. An **lvalue** has identity and outlives the expression. An **xvalue** is an lvalue that may be stolen from. A **prvalue** is a temporary with no identity, destroyed at the end of the full-expression. An **xprvalue** covers cases that are both. In practice you need two working buckets: named things you must not steal from, and temporaries you may.

**Copy** duplicates an object entirely. **Move** transfers ownership of the object's resources — for a heap-backed object, the pointer is taken and the source is left in a valid but *unspecified* state whose contents you must not read. Moving is O(1) where copying is O(size).

`std::move` is the most misunderstood name in C++. **It performs no move.** It is a cast that marks an expression as an xvalue so overload resolution selects the move constructor rather than the copy constructor; the move itself happens inside whichever constructor got picked. This is why moving an object that does not declare a move constructor silently copies.

**Rule of Zero, Three, and Five.** Declare none of destructor, copy constructor, copy assignment, move constructor, move assignment, and the compiler generates all of them correctly — you write nothing (Rule of Zero, and the default you want). Declare one or two and you are now responsible for all of them, because declaring a destructor suppresses the implicit move operations (Rule of Three, for types that manage raw resources). Do it fully and you have Rule of Five, including move operations and the `noexcept`-ness the move constructor must have to be usable during a container reallocation. The book needs none of this, which is the point of the Rule of Zero: holding pointers and letting the maps own the levels means the compiler handles it.

**The consequence that shapes the design is reallocation.** A `std::vector` stores elements contiguously, so growing it means allocating new storage and relocating every element. Anything holding a pointer, reference, or iterator into that vector has all of them invalidated by the growth. The book holds raw pointers between orders, so a vector of orders would dangle every one of those pointers on the first growth. That is why each `Order` is individually allocated — trading a little allocation cost for pointer stability, which is precisely the trade week 5's pool exists to optimise *without* reintroducing the hazard.

Also worth knowing: **copy elision**. Modern C++ guarantees that a temporary used to initialise an object need not be copied at all, so a copy constructor is often never called even when one exists. Do not put required side effects in a copy constructor.

### 2.7 Templates and function objects

A template is a blueprint the compiler uses to generate real code, once per distinct set of type arguments. Nothing about a template exists in the binary until it is instantiated — which is why template errors surface as pages of diagnostics from deep inside the standard library rather than one clear message.

The concept that matters for the book is the **function object** (functor): a type defining a call operator, so instances can be used anywhere a callable is expected, *including as a non-type template argument*. `std::greater` and `std::less` are function objects, not functions — the template parameter is a **type**, and the compiler instantiates the comparison against `Price`. That is how the book gets a descending bid map and an ascending ask map from the same container.

The hard requirement is that a comparator be a **strict weak ordering**: irreflexive, asymmetric, transitive, and with transitive incomparability. The map relies on all four and the compiler cannot check them. A merely "mostly correct" comparator produces a container whose iteration order and complexity guarantees are not what the standard promises — typically an order-dependent crash much later, or a tree silently degrading toward linear.

C++20 added **three-way comparison**: a single `operator<=>` returning an ordering type (`std::strong_ordering` and friends) replaces the whole family of six comparison operators. You will meet it throughout any modern codebase, and its absence in the order book should read as a deliberate hot-path decision you can explain.

Variadic templates you will not write here, but you will read them constantly inside doctest.

### 2.8 RAII

Resource acquisition is initialisation: a resource is acquired in a constructor and released in a destructor, so both are tied to the same object's lifetime and the compiler guarantees the release. The name is a deliberate pun.

It exists because C++ has many exits from a scope — falling off the end, returning early, `break`, `throw` — and cleanup repeated at each of them is cleanup that will eventually be forgotten at one. The destructor is the compiler's guarantee that cleanup happens on all of them, including the ones you did not write.

"Resource" is much broader than memory. It is anything you must acquire and later give back: a file handle, a socket, a mutex lock, a thread, a fixed-size memory block, a timer registration. Every one of those is an RAII object in this project's later weeks — the socket in week 6, a lock guard around a shared book, and the pool's block handle in week 5, which returns the block to the pool precisely when the handle is destroyed. The handle *is* the RAII object; the pool knows nothing about scope.

RAII becomes *more* important in a codebase that forbids exceptions, not less. Without stack unwinding there is no mechanism to clean up on the way out, so cleanup has to be structural — bound to scope — rather than reactive.

**Deterministic destruction** gives you something a garbage collector cannot: you know exactly when a resource is released. That is what lets you reason about when memory is recycled and when a socket is actually closed, and it is the philosophical counterweight to a plan whose second rule is "no allocations after warmup."

### 2.9 Exceptions and the cost of forbidding them

An exception propagates by **stack unwinding**: the runtime walks outward, running the destructor of every object in each frame it leaves, until it finds a handler. That requires the compiler to emit unwind tables describing what to destroy in every frame, and it means functions on the path cannot be optimised as freely because control flow might leave abruptly.

`-fno-exceptions` removes the machinery entirely: no unwind tables, smaller code, better optimisation, and a compile error if you accidentally throw. The plan enables it for bench and release builds (`trading_sys_plan.md:49`) and separately forbids exceptions in the hot path as a design rule (`trading_sys_plan.md:43`).

Without exceptions, errors are reported structurally, three ways, and the project uses all three: **return codes or `bool`** for expected failures, **`std::optional`** when "no value" is a normal outcome rather than an error, and **`assert`** for programming errors that must never happen — where failing loudly in a debug build is more valuable than recovering. The book mostly uses assertions, because a `best_bid()` on an empty side is not an error, it is a legitimate empty book; that is what `kInvalidPrice` and the `std::optional` alternatives are for.

The related benefit: a function marked `noexcept` tells the compiler control cannot leave it by exception, so it can be inlined and reordered more aggressively. That is why the book's cheap queries are `noexcept` and the mutating operations are not — mutation allocates and can fail, and claiming otherwise would be a lie the optimiser is entitled to exploit.

---

## Tier 3 — Containers, in depth

Not "what a map is" — how it works, because the book depends on the details.

### `std::map<K, V, Compare>`

- Red-black tree underneath; **node-based**, not contiguous
- **O(log n)** insert and lookup, **O(1)** `begin()`/`end()`, O(n) traversal
- The complexity guarantees are *guaranteed* — a standard-library map may not degrade to linear on sorted input. This is a stronger promise than most containers make, and it is why the complexity is worth discussing
- **Custom comparator as the third template parameter** — descending order for bids, ascending for asks
- **`operator[]` versus `insert` versus `find` versus `at`** — `operator[]` on a missing key silently default-constructs a value. For the book that is a silent `PriceLevel` appearing where none existed; `find` returning `end()`, or `at` throwing, are the honest options
- **Iterator stability** — insert and erase never invalidate iterators to *other* elements. This is the property that makes holding a `PriceLevel*` across a later insert safe
- **`erase` invalidates only the erased element's iterator** — and destroying the node it pointed to. The other common dangling-pointer source
- **Node-based means no reallocation** — every node is its own allocation, so a growing map does not move existing elements. This is why the book can safely hold pointers into levels, and it is the counterpart to why it cannot hold pointers into a `vector`

### `std::unordered_map<K, V>`

- **Hash table**, open addressing or chaining
- **O(1) average, O(n) worst case** — and the worst case arrives when keys collide heavily, which an adversary can arrange
- **`std::hash` for integers is the identity function**, so `OrderId` keys hash to themselves. That is fast and fine for sequential ids, but a patterned key distribution degrades it
- **Iterator invalidation is much worse than `map`** — a rehash invalidates *everything*, because the elements move to new buckets
- **`reserve()` and `max_load_factor`** matter here for a project-specific reason: the rule is no allocations after warmup, and a rehash mid-run *is* an allocation. This control is the connection between this bullet and the week's actual work

### Other containers and utilities

- **`std::optional<T>`** — a value that may be absent, the alternative to exceptions and to sentinel values. The `modify` operation may want it
- **`std::pair`, `std::tuple`, `std::get`**, structured bindings for unpacking them
- **`std::move`, `std::forward`, `std::exchange`, `std::swap`**
- **Iterator categories** — at minimum: a `std::map` iterator is not a pointer, it is invalidated by erase of that element, and `std::distance` works on it
- **Iterator invalidation as a concept** — the single most important STL topic for this project

---

## Tier 4 — Specific to this codebase

Smaller, but you hit them on day one.

- **`assert`** and `<cassert>` — and the `NDEBUG` trap: in a release build `assert` compiles to *nothing*. That is both a performance tool and a hazard, because a function whose only validation was an `assert` is unguarded in release
- **Sentinels versus `optional`** — the real design question your `best_bid()` return type is answering. A sentinel is free but occupies a valid value; `optional` is explicit but larger and slower to pass. For a hot query, the sentinel wins
- **`std::numeric_limits`** — for the `kInvalidPrice` sentinel, and why the min of each integer type is the right "impossible" value rather than zero (zero is a legal price in some conventions, and a legal quantity in none)
- **Bitwise operations on types** — `std::uint8_t` flags with or/and/not. Probably unused here, but standard in exchange code
- **Preprocessor**: `#include` versus forward declaration, `#define`, conditional compilation such as `#ifdef NDEBUG` for the invariant checks. The preprocessor is not C++ — no scopes, textual substitution, which is why macro bugs exist and why `const`/`constexpr` are almost always the better answer

---

## Tier 5 — Testing (needed before slice 3)

- **doctest macros**: `TEST_CASE`, `SUBCASE`, `CHECK`, `REQUIRE`
- **`CHECK` versus `REQUIRE`** — `CHECK` logs the failure and continues, `REQUIRE` aborts the current case. Knowing this is the difference between one clear failure and a cascade of confusing ones
- **Assertions versus exceptions in tests** — a failing assert is the right tool; an exception escaping a test is a harness bug
- **Fixtures** — doctest's `TEST_CASE_FIXTURE`, or a plain setup function, and why a fresh book per test is non-negotiable for a stateful structure
- **`<random>`**: `std::mt19937` (the generator), `std::uniform_int_distribution` (the mapping to a range), `std::seed_seq` (seeding)
- **Why a fixed seed matters** — reproducibility of a failure. A randomised test that cannot be replayed is a coin flip
- **Property-based testing** — the idea: assert *invariants* over generated input rather than asserting one expected output. The randomised cross-check in slice 3 is exactly this

---

## Study order, tied to the slices

Read each block immediately before the slice that needs it. Do not front-load.

| Before | Learn |
|---|---|
| **Slice 0** | CMake targets, `-Werror` warning categories, include guards, the preprocessor, translation units |
| **Slice 1** | Type aliases, `enum class`, `struct`, `static_assert`, `constexpr`, **alignment / `sizeof` / `alignof` / padding** |
| **Slice 2** | **All of Tier 2.** Plus `map` and `unordered_map` in depth, const correctness, iterators and invalidation, `new`/`delete`, UB |
| **Slice 3** | doctest, `<random>`, property-based testing, RAII for fixtures |
| **Slice 4** | `switch` on `enum class`, control flow — the easy one |

**Honest estimate:** Tier 0 and Tier 1 together are about a day of skimming. Tier 2 is the real work and is the part that shows up in interviews. The concepts are individually simple; the difficulty is that they interlock, and `cancel()` exercises lifetime, ownership, undefined behaviour, and const all at once.

---

## Cross-references

- `docs/plan_order_book.md` — the build plan this supports, with the slice exit conditions
- `trading_sys_plan.md` — the 12-week plan; week 2 covers memory layout, week 3 covers memory ordering, week 4 covers flags and measurement
- The single-writer rule in `rules.md` is the direct motivation for slice 2's const design (section 2.4)
- The allocation seam in `docs/plan_order_book.md` is the direct motivation for sections 2.1 and 2.6

## The one self-test

If you want to check comprehension without writing the book, the highest-value question is one the plan already asks at `trading_sys_plan.md:133` — *why does a memory ordering matter here at all*. That is week 3 content, not this slice. If the answer feels like it should be obvious, you have the right instinct about why the SPSC queue is the next component rather than more of the book.
