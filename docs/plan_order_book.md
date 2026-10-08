# Order Book — Build Plan

**Status:** approved, not yet implemented
**Scope:** the limit order book and the minimum scaffolding needed to run and test it
**Source:** slices derived from `trading_sys_plan.md` Weeks 1, 2 and 7, reordered to start at the heart of the system instead of the teaching order

---

## What "slices" means

A **slice** is the smallest chunk of work that leaves the repository in a *working state*.

That is the whole idea. After every slice:

- the project compiles,
- the tests pass,
- the program runs and does something real.

There is never a moment where the tree is half-broken. If you stop after slice 3, you have a tested order book and no program entry point — annoying, but not broken. If you stop after slice 2, you have a compiling library with no tests — you just have to trust it for a day.

Each slice below states an **exit condition**: the concrete thing you can do when it is finished. Read those as the definition of done. If you cannot satisfy the exit condition, the slice is not finished, regardless of how much code exists.

```
slice 0   foundations        rules, directories, doctest wired in
  |
slice 1   core types         Price, Qty, Order, PriceLevel — no logic yet
  |
slice 2   the book           add / cancel / modify / best_bid / best_ask
  |
slice 3   invariant tests    proven correct against a naive reference
  |
slice 4   runnable main      a program that generates messages and prints a book
  |
slice 5   rough numbers      placeholder latency measurements (optional)
```

Slices 0–2 build the thing. Slice 3 proves it. Slice 4 gives it a purpose. Slice 5 measures it. Slices 0–4 are the real work; slice 5 is optional and separable.

### Why this component, and why first

The order book is the only component in the system with **zero prerequisites**. It does not need sockets, threads, allocators, a framework, or a data feed. It is pure single-threaded logic over integers, and every other part of the system either writes into it (the feed handler, the matching engine) or reads out of it (the strategy, the gateway, the Python backtest).

`trading_sys_plan.md` schedules it for Week 7, but that is a *teaching* order, not a *dependency* order. The things scheduled before it — the SPSC queue, the pools, the sockets — are infrastructure that surrounds the book, not inputs to it. Building the book first means the hard, correctness-critical part gets the most attention, and the infrastructure later has a real consumer to be designed against.

### Why single-threaded first

Threading is added in one later step, after the book's behaviour is known to be correct. Two reasons:

1. **Debuggability.** A wrong answer from a single-threaded book has one cause. The same wrong answer from a threaded book has two candidate causes — the logic, or the concurrency. Separating them is far cheaper than reasoning about both at once.
2. **The invariant tests only work single-threaded.** The randomized cross-check in slice 3 calls `add` and `best_bid` in a tight loop and compares against a reference after every single call. That is inherently sequential. Once it is green, the threading step becomes a mechanical split: feed thread produces, book thread consumes, one queue between them, and the same tests still apply to the book itself.

---

## Slice 0 — Foundations

**Exit condition:** `cmake` configures, an empty `unit_tests` binary builds and runs, and `rules.md` is committed.

| File | Action |
|---|---|
| `rules.md` | **New.** The four rules from `trading_sys_plan.md:41-45` |
| `include/ts/book/` | **New directory.** Public headers live under `include/ts/`, mirroring the skeleton in `trading_sys_plan.md:24-27` |
| `docs/` | **New directory** (this file lives here) |
| `CMakeLists.txt` | Add doctest via `FetchContent` (v2.4.11), add the `ts_book` static library target, add a `unit_tests` executable, call `enable_testing()` |

**On `rules.md`:** the plan calls for this on day 1 of week 0 (`trading_sys_plan.md:41`). It is a real artifact, not ceremony — it is a short list of constraints that later slices would otherwise violate by accident. Specifically, the third rule, *every shared struct has a single writer*, is the justification for building single-threaded first and is what makes the eventual thread split safe.

**On doctest:** chosen over Catch2 because it is a single header, so there is no build-system friction between "installed" and "not installed". The plan leaves the choice open (`trading_sys_plan.md:72`). It is fetched at configure time, not vendored, so nothing is committed.

**Note on the existing CMake file:** the blocks for `ts_feed`, `ts_engine`, `ts_gateway` and `ts_strategy` (around `CMakeLists.txt:58-92`) are templates with a comment warning that they must stay commented until the files exist. They stay commented. CMake fails at *configure* time on a missing source file, not at build time, so uncommenting them breaks the whole workflow, not just one target.

**Sanitizers are already wired.** The existing `CMakeLists.txt:26-32` turns on ASan+UBSan for `CMAKE_BUILD_TYPE=Debug`. The existing `build/` cache is already configured as `Debug` with Ninja, so `cmake --build build` picks this up with no extra flags.

---

## Slice 1 — Core types

**Exit condition:** headers compile, `sizeof` and `alignof` assertions hold, and a test can construct an `Order` and a `PriceLevel`.

This slice contains no behaviour. It is the vocabulary the rest of the system is written in, and getting it right here is cheaper than retrofitting it later — every function signature in slice 2 depends on these four typedefs being the right width and signedness.

### `include/ts/book/types.hpp`

Four typedefs and one enum.

```cpp
namespace ts::book {

using Price   = std::int64_t;
using Qty     = std::int64_t;
using OrderId = std::uint64_t;

enum class Side : std::uint8_t { Buy, Sell };

}
```

| Type | Width | Why this type |
|---|---|---|
| `Price` | signed 64-bit | **Prices are integer ticks, never floating point.** A price is the number of ticks above some reference, not a decimal like `101.25`. This is the single most important type decision in the whole component, and the classic quant-developer trap: floats cannot represent most decimal prices exactly, so `0.1 + 0.2 != 0.3`, and equality comparisons on prices start failing in ways that are nearly impossible to debug. Signed because a price difference can be negative, and because the sentinel for "no price" needs a value outside the valid range. |
| `Qty` | signed 64-bit | Quantity, also in integer units. Signed for the same sentinel reason, and so that intermediate subtraction during matching cannot silently wrap. |
| `OrderId` | unsigned 64-bit | A handle assigned to each order. Unsigned because it is a pure label — it is compared for equality and never subtracted. 64 bits because exchange order IDs are large and because a monotonically increasing counter will not wrap in any realistic run. |
| `Side` | `uint8_t` enum class | Two states. `enum class` rather than a bare `enum` so the enumerators do not leak into the enclosing namespace, and so the type cannot implicitly convert to `int`. One byte because the book stores this in every order and the level, and cache footprint is the constraint that matters here. |

**On integer prices specifically.** The conversion happens once, at the protocol boundary in a later slice, and never again. A price of `101.25` for an instrument quoted in quarter-dollar ticks is `405` (`101.25 * 4`). Inside the book, `405` is just an integer and every comparison, hash and ordering is exact. A float-based book is slower (bigger, and comparisons are not always single-cycle) *and* wrong. Both problems disappear together by making this one decision up front.

### `include/ts/book/order.hpp`

The individual resting order.

```cpp
namespace ts::book {

struct alignas(64) Order {
    OrderId  id;
    Price    price;
    Qty      qty;
    Side     side;
    Order*   prev;      // previous order at the same price, FIFO
    Order*   next;      // next order at the same price, FIFO
};

}
```

| Field | Role |
|---|---|
| `id` | The handle the caller uses to cancel or modify this order. Also the key into the book's lookup index. |
| `price` | Integer tick price. Redundant with the level it sits in, but stored here so that a cancel can locate its level without a reverse pointer. |
| `qty` | Remaining quantity. **Remaining, not original** — it is decremented as the order is partially filled, and that is the value the FIFO ordering and the aggregate volume both care about. |
| `side` | `Buy` or `Sell`. The side determines which of the book's two level maps the order belongs to. |
| `prev` / `next` | Raw pointers forming an intrusive doubly-linked list with the other orders at the same price. Intrusive means the links live *inside* the object, so the list needs no separate node allocation. |

**Why `prev` *and* `next`?** A singly-linked list can cancel from the front cheaply but not from the middle. A cancel can arrive for any order, not just the oldest at a level, so the middle must be O(1) too. Two pointers buy that.

**Why `alignas(64)`?** A cache line is 64 bytes on x86-64 — the granularity at which the CPU fetches memory from L2/L3/DRAM. If two `Order` objects share a line and two threads touch different orders, the line ping-pongs between cores and throughput collapses. This is **false sharing**, and it is the reason for the alignment. It costs memory (padding every order to a full line, even though the useful data is ~30 bytes) and buys predictable per-object cache cost.

Pair with it a `static_assert` on `sizeof(Order) == 64` and one on `alignof(Order) == 64`. The point is not the assertion itself — it is that the padding becomes a *deliberate, documented* decision instead of an accident of field ordering that changes when someone adds a field.

**Honest scoping note:** alignment is correct to do now, but it cannot be *justified* with measurements until the benchmarks exist. `trading_sys_plan.md:101` makes the padding exercise a deliberate week-2 task for exactly this reason. Doing the alignment now and measuring later is fine; claiming a speedup now would not be.

### `include/ts/book/price_level.hpp`

All orders resting at one single price.

```cpp
namespace ts::book {

struct PriceLevel {
    Price          price;
    Qty            total_qty;    // cached sum of member order quantities
    std::size_t    order_count;  // number of resting orders
    Order*         first;       // head of FIFO — oldest, highest priority
    Order*         last;        // tail of FIFO — newest
};

}
```

| Field | Role |
|---|---|
| `price` | The tick price this level represents. |
| `total_qty` | **Cached aggregate** of every member order's remaining quantity. Maintained incrementally on every add, cancel and fill. |
| `order_count` | Number of resting orders. Cached for the same reason, and because emptiness can be tested either way. |
| `first` | Oldest order at this price. In price-time priority this is the one that trades first. |
| `last` | Newest order at this price — the tail that new orders are appended behind, and the one a cancel most often targets. |

**Why cache the aggregate?** `volume_at_level` is on the hot path — a strategy deciding whether a level is deep enough, a matching engine sizing an order against available liquidity. Summing the whole list on every call is O(orders at that price). Keeping the running total makes it O(1). The trade-off is that the cache must be updated on *every* mutation, which is a bug surface; slice 3's tests exist largely to catch exactly that class of bug.

**The two-level picture:**

```
                        OrderBook
                            |
              +-------------+-------------+
              |                           |
        bids_ (descending)           asks_ (ascending)
              |                           |
        +-----+-----+               +-----+-----+
        | 105 | 104 |               | 107 | 108 |
        +---+---+   |               |   +---+---+
            |       |               |       |
        +---+---+   |               |   +---+---+
        | 103 | ... |               | ...| 110 |
        +-----+     |               |     +-----+
                    |               |
            best_bid is the        best_ask is the
            FRONT of bids_        FRONT of asks_
```

Each entry in the maps is a `PriceLevel`, and each `PriceLevel` heads a FIFO list of `Order`s. Reading `best_bid()` is reading the front entry of one map — no traversal of the list, no scan across prices.

---

## Slice 2 — `OrderBook`

**Exit condition:** `ts_book` compiles into a static library, and a test can add orders, cancel them, and read a correct top-of-book.

### `include/ts/book/order_book.hpp`

```cpp
namespace ts::book {

class OrderBook {
public:
    // --- mutation -------------------------------------------------------
    void add(OrderId id, Side side, Price price, Qty qty);
    void cancel(OrderId id);
    void modify(OrderId id, Price new_price, Qty new_qty);

    // --- queries (all const, all noexcept where possible) ---------------
    Price best_bid() const noexcept;
    Price best_ask() const noexcept;
    Qty   volume_at_level(Price price) const;
    Price best_bid_at_or_better(Price limit) const noexcept;

    // --- introspection (for tests and assertions) -----------------------
    std::size_t level_count(Side side) const noexcept;
    std::size_t order_count() const noexcept;
    bool        contains(OrderId id) const noexcept;
    bool        crossed() const noexcept;
    void        check_invariants() const;

private:
    // members described below
};

}
```

#### Mutation

**`void add(OrderId id, Side side, Price price, Qty qty)`**

Places a new resting order.

- `id` — caller-assigned handle. Must be unique; a duplicate is a programming error and should assert.
- `side` — selects which map the order goes into.
- `price` — integer tick price. The level is created on first arrival at that price and reused thereafter.
- `qty` — remaining quantity. Must be positive.

Steps: validate inputs, find-or-create the level, append the order to the tail of that level's FIFO list, increment the level's cached `total_qty` and `order_count`, insert into the id index, and update the cached best price if this order improves it.

Cost: O(log n) for the map lookup, O(1) for everything else.

**`void cancel(OrderId id)`**

Removes a resting order entirely.

- `id` — the handle passed to `add`.

Steps: look up the order in the id index; if absent, this is a no-op (a cancel for an order that was never added, or was already filled or cancelled, is normal in real venues and must not be an error). Otherwise unlink it from its level's FIFO list, subtract its quantity from the level's cached aggregate, decrement the level's count, erase the index entry, and free the order. If the level became empty, erase the level from the map and, if it was the best level, advance the cached best price.

Cost: O(log n). Note the order knows its own price and side, so no reverse pointer from level to order is needed.

**`void modify(OrderId id, Price new_price, Qty new_qty)`**

Changes a working order's price and/or quantity, in place, without going through a cancel.

- `id` — the handle.
- `new_price` — the replacement price. If equal to the current price, this is a quantity-only modification.
- `new_qty` — the new total quantity. The convention to adopt (and to match most venues) is that this is the *new total remaining*, not a delta.

**The rule, which is the whole point of this function:**

| Case | Behaviour | Time priority |
|---|---|---|
| `new_price` equals current price | Update the quantity in place. Level's cached aggregate adjusted by the difference. | **Kept.** The order stays where it is in the FIFO. |
| `new_price` differs | Cancel and re-add at the new price. | **Lost.** The order goes to the back of the new level's queue. |

This mirrors real exchange behaviour: reducing or increasing size does not change your place in line, but moving to a different price does, because you have left the queue you were in. An implementation that silently kept priority across a price change would be a subtle and expensive bug — it would let a strategy jump the queue, and the resulting fills would be unreachable in a backtest.

Cost: O(log n) either way.

#### Queries

**`Price best_bid() const noexcept`**

Highest price at which someone is willing to buy. Returns the sentinel `kInvalidPrice` if the bid side is empty.

Reads a **cached field** updated whenever the best level changes, so this is a single memory read — no map traversal, no list traversal. That is what gets it under the plan's 100ns bar (`trading_sys_plan.md:229`), and it is why the cached field exists at all: `std::map::begin()` would be O(1) too, but returning a member the book already maintains is cheaper and cannot get out of sync with the map, because the same code path updates both.

**`Price best_ask() const noexcept`**

Lowest price at which someone is willing to sell. Mirror of the above; sentinel if the ask side is empty.

**`Qty volume_at_level(Price price) const`**

Total resting quantity at one price.

- `price` — the tick price to look up.

Returns `0` if no level exists there. O(log n) for the map lookup, then O(1) to read the cached aggregate. Does not walk the order list.

**`Price best_bid_at_or_better(Price limit) const noexcept`**

Highest resting bid that is at or above `limit` — the price a market-sell order would hit. Returns the sentinel if no such bid exists. Included because it is the query a market-order matcher needs, and adding it now keeps slice 2's interface honest about what the book is for.

#### Introspection

These exist for tests and assertions. They are cheap, and they are the difference between a book you can prove correct and one you can only eyeball.

| Signature | Returns |
|---|---|
| `std::size_t level_count(Side side) const noexcept` | Number of distinct prices with resting orders on that side. |
| `std::size_t order_count() const noexcept` | Total resting orders across both sides. |
| `bool contains(OrderId id) const noexcept` | Whether `id` is currently live. Lets a test assert an order was actually removed. |
| `bool crossed() const noexcept` | Whether `best_bid >= best_ask`. A resting book must never be crossed. |
| `void check_invariants() const` | Asserts the full invariant set. No-op in release builds via `assert`. |

`check_invariants()` is the one that matters. It verifies, in one call: bid strictly less than ask; no duplicate prices; each level's cached `total_qty` equals the sum of its member orders' quantities; `order_count()` equals the sum of all level counts; every order in the index is reachable from some level's FIFO list, and vice versa.

#### Private members

```cpp
std::map<Price, PriceLevel, std::greater<Price>> bids_;  // descending
std::map<Price, PriceLevel, std::less<Price>>    asks_;  // ascending
std::unordered_map<OrderId, Order*>              index_; // id -> live order
Price best_bid_ = kInvalidPrice;  // cached, maintained on every change
Price best_ask_ = kInvalidPrice;
```

**Why the two orderings.** `std::map` keeps keys sorted. Bids sort *descending* and asks sort *ascending*, so in both maps the best price is the *front* entry. One convention, one access pattern, no special-casing, and "next best price after the best is emptied" is just "the next entry forward". Sorting the other way and using `rbegin()` works too, but then best-price access differs between the two sides and every future edit has to remember which side is which.

**Why `index_`.** Without it, `cancel(id)` has nowhere to start — the id is not stored in the levels, and scanning every level for a matching order would be O(orders). The index is the standard price/time trade: pay O(1) expected lookup and O(n) memory to make cancel O(1) instead of O(n). `Order` is stored in a `std::unordered_map` keyed by `OrderId`, which is exactly what a `std::unique_ptr<Order>` in an `unordered_map` would give, minus the indirection.

**Memory note.** `index_` is roughly one pointer plus a hash-table slot per live order. For a book holding millions of resting orders this is the largest single memory consumer, and a flat-array index or open-addressing map is a standard later optimisation. Not now — noting it so it is a decision rather than a surprise.

#### The allocation seam — a deliberate deferral

`add` allocates one `Order` with `new`; `cancel` frees it with `delete`. Everywhere else, nothing allocates.

This is a **known, documented, temporary** violation of the plan's "no allocations after warmup" rule (`trading_sys_plan.md:50`), and the containment is the point:

- Every allocation happens inside `add`. Every deallocation happens inside `cancel`. Nowhere else.
- Nothing in slices 3 or 4 allocates per operation either.
- Week 5's pool allocator replaces the `new` and the `delete` and touches no other line of the book.

Isolating it this way means the pool swap is a two-line change instead of a refactor. Do not "improve" this by preallocating or pooling early — the measurement that justifies a pool does not exist yet.

---

## Slice 3 — Invariant tests

**Exit condition:** the randomized cross-check passes, and the whole suite is green under ASan+UBSan.

This is the slice that decides whether the book is *correct* rather than merely *present*. The plan's rule is that every component ships with tests under sanitizers (`trading_sys_plan.md:50`), and for an order book specifically, the reason is not formality: a book that is subtly wrong produces plausible-looking output and wrong backtests, which is the worst failure mode available.

### `tests/reference_book.hpp` — the oracle

A deliberately naive second implementation: `std::multimap<Price, Qty>` per side, linear scan for the best price, linear sum for volume. Slow, obvious, and *independently written* — the value comes from the two implementations sharing no code and no data structures.

It lives in `tests/`, not in `src/`, and never links into the shipped library. This is the reference implementation the plan asks for at `trading_sys_plan.md:222`.

If both implementations agree after every operation across a long random stream, a whole class of bug is ruled out — and because the two use different data structures, a shared conceptual error is much less likely to produce the same wrong answer twice.

### Randomized cross-check

```
for each of N operations (N = 200,000, fixed seed):
    pick a random operation: add | cancel | modify | query
    pick random parameters from a valid range
    apply it to OrderBook
    apply the same operation to ReferenceBook
    compare:  best_bid, best_ask, volume_at_level(price),
              level_count(both sides), order_count(),
              crossed(), and the full quantity at every level
    assert equality
    assert OrderBook::check_invariants() holds
```

**Fixed seed.** A failing test must reproduce. `std::mt19937` with a hardcoded seed means any failure is replayable forever; print the seed and operation index on assertion failure. This is the property-based testing the plan requires at `trading_sys_plan.md:244`.

**One operation per iteration, compared every time.** Comparing only at the end would tell you *that* the books diverged but not *where*. Per-operation comparison localises the bug to a single op, which usually makes it obvious.

**Invariants asserted alongside:**

| Invariant | Catches |
|---|---|
| `best_bid < best_ask` | Crossed book — the matching logic is trying to match against a resting order |
| No duplicate prices in a side's map | Level created twice for one price; aggregate double-counted |
| `total_qty` equals the sum of member orders | Missing or duplicated aggregate update — the cached-field bug |
| `order_count()` equals the sum of level counts | Order leaked or double-counted across levels |
| Index and level lists contain the same orders | Dangling pointer, or an order added to the list but not the index |
| FIFO order within a level matches insertion order | Priority violated — the order that should trade first is not first |
| Quantity conservation against an independent tally of the op stream | Quantity appearing or vanishing; catches signed-overflow wraparound |

### Targeted cases

The random stream proves the invariants hold; these prove the *specific behaviours* are right, which random testing alone will not hit reliably:

- add several orders at one price, confirm FIFO order by inspecting the level
- partial cancel, confirm the level aggregate decreases by exactly that quantity
- full cancel of the last order at a level, confirm the level is removed **and the best price advances**
- full cancel of the best level with more levels behind it, confirm the cached best updates
- modify quantity at the same price, confirm priority kept and aggregate adjusted
- modify across prices, confirm priority **lost** and the order is at the new level's tail
- cancel an unknown id, confirm it is a no-op and does not corrupt state
- add to an empty book, confirm both sides report the correct sentinel when empty
- sequence: best bid resting, best ask added below it, confirm the book reports crossed and `check_invariants()` fires

Run the whole suite under the existing ASan+UBSan Debug configuration. Any use-after-free from the intrusive list surgery in `cancel` shows up here.

---

## Slice 4 — Runnable `main`

**Exit condition:** `./build/trading_system` prints a top-of-book summary generated from a synthetic message stream, and exits zero.

The point of this slice is having a program. Not a library with tests — a thing you run.

### `src/feed/message_generator.hpp`

```cpp
namespace ts::feed {

enum class MessageType : std::uint8_t { Add, Cancel, Execute };

struct FeedMessage {
    MessageType type;
    OrderId     order_id;
    Side        side;
    Price       price;
    Qty         qty;
};

class MessageGenerator {
public:
    explicit MessageGenerator(std::uint64_t seed);
    FeedMessage next();          // next message in the stream
    std::uint64_t position() const noexcept;
};

}
```

`next()` returns a plausible market-data message: mostly adds at prices near a slowly drifting mid, occasional cancels, occasional executes. It is not random noise — a book fed pure noise collapses into thousands of meaningless price levels, and the printed output stops being readable enough to eyeball. The drift is what makes the book look like a book.

`position()` reports how many messages have been generated, used by `main` to label its progress.

**Where this lives matters.** It is in `src/feed/`, not `src/book/`, because it is the *source* of the data, not part of the book. The real UDP feed generator replaces it in week 6 and the `main` loop does not change.

### `src/main.cpp`

```
create a MessageGenerator with a fixed seed
create an OrderBook
for N messages (N = 1,000):
    pull a message
    dispatch it to the book:
        Add    -> book.add(id, side, price, qty)
        Cancel -> book.cancel(id)
        Execute-> book.cancel(id)   // an execute removes the filled quantity
                                    // from the resting book; the matching
                                    // engine arrives in week 8
print: best bid / best ask / spread / level counts / order count
print: total messages applied, and a self-check line
```

**The boundary that matters here.** The book's public API stays primitive — `add`, `cancel`, `modify`. `main` does the unpacking from `FeedMessage` into those calls. The book never sees a `FeedMessage` and never knows a feed exists.

This is deliberate, and it is what keeps the book reusable. In week 8 the matching engine calls the same three methods. In week 9 the feed handler thread calls the same three methods after decoding a binary wire message. In week 10 the gateway indirectly drives the same three. If the book had taken a `FeedMessage`, every one of those would need a conversion or a fake message.

`Execute` is mapped to `cancel` on purpose, with a comment saying so. It is a simplification valid for the first cut — an execute fully removes a resting order, which is what a cancel does. Partial fills, where a resting order is only partly consumed and remains on the book, arrive with the matching engine. Calling this out prevents it from looking like an oversight.

### Out of scope for this slice

No UDP sockets, no threads, no binary protocol decoding, no strategy, no PnL. Just: synthesise messages, apply them, print the result.

---

## Slice 5 — Rough latency numbers (optional)

**Exit condition:** a printed p50/p99 for `add` and `best_bid`, with a comment marking the code as a placeholder.

Ten minutes of work, and it answers the only question that matters at this stage: *is the data structure in the right ballpark?* A book whose `add` is microseconds rather than nanoseconds means the structure is wrong, and that is worth knowing before investing in the rest.

```cpp
// bench/bench_order_book.cpp  — PLACEHOLDER, replaced by the real
// harness in week 4 (trading_sys_plan.md:151).
// Prints p50/p99 for add and best_bid. Not a rigorous benchmark:
// no warmup control, no run pinning, no repetition, no max.
```

Targets from the plan (`trading_sys_plan.md:229`, scalable to the hardware): `add` p50 under 300ns, best-quote p50 under 100ns.

**Do not put these numbers in `docs/benchmarks.md`.** That file is interview ammunition and the plan demands methodology notes with every figure (`trading_sys_plan.md:155`). A placeholder measurement has neither pinned CPU, nor disabled turbo, nor repetition count, and citing it would be dishonest in exactly the way the plan is trying to prevent. The real harness in week 4 produces the citable numbers.

This slice is genuinely optional. Skip it if time is short; nothing later depends on it.

---

## Out of scope

Explicitly not in this plan, and not to be started until slices 0–4 are green:

- SPSC queue and the thread split (week 3 / week 9)
- Pool and arena allocators (week 5) — the seam is already in place for this
- UDP sockets, binary protocol, gap detection (week 6)
- Matching engine and order state machine (week 8)
- Gateway, order manager, strategy, Python analytics (week 10)

Slice 4 is the last thing that touches `src/main.cpp` before the threading step.

---

## Verify

```bash
# configure (ASan+UBSan turn on automatically for Debug)
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug

# build
cmake --build build

# tests
ctest --test-dir build --output-on-failure
# or run the binary directly for doctest's own reporting:
./build/unit_tests

# program
./build/trading_system
```

The existing `build/` cache is already `Debug` + Ninja, so the first command is only needed if it is ever cleared.

---

## Immediately after

Two pieces of work, in this order:

1. **`docs/design-book.md`** — the design doc. The plan requires a design doc *before* coding each component (`trading_sys_plan.md:165`), and for the book specifically it should record the decisions made here and why: integer prices, the two-level map-plus-FIFO-list structure, the cached aggregates and best prices, the modify-priority rule, and the allocation seam. It is cheap to write now while the reasoning is fresh, and it is the artifact an interviewer reads.

2. **The SPSC queue** — the next component, and the one thing that genuinely cannot be skipped. Not for the learning exercise, but because the entire architecture the plan describes at week 9 (one component per thread, single-writer ownership, feed handler producing and book consuming) is built on it. Write it as a component, not as a week-long study: a fixed-capacity power-of-two ring buffer with acquire/release ordering, which is a well-understood design with a known correct implementation shape.

The thread split then becomes mechanical — feed thread produces, book thread consumes, one queue between them — and the slice-3 invariant tests still apply unchanged to the book itself.
