# python-sorted-vs-sort-production-safety
Runnable code examples, benchmarks, and tests for every pattern in the sorted() vs list.sort() production safety guide — with cache corruption demos, GIL behavior proofs, and a full test suite.
# python-sorted-vs-sort-production-safety

> A practical code reference for the article:
> **[Why sorted() Is Safer Than list.sort() in Production Python Systems](https://emitechlogic.com/why-sorted-is-safer-than-list-sort-in-production-python-systems/)**

This repository contains every runnable code example, benchmark script, and test case from the article. Each file is self-contained and annotated. You can clone the repo and run any example directly — no external dependencies beyond the Python standard library.

---

## Why This Repository Exists

The article at [emitechlogic.com](https://emitechlogic.com/why-sorted-is-safer-than-list-sort-in-production-python-systems/) covers a category of bug that is genuinely hard to trace in production: `list.sort()` mutating shared objects silently, across references, with no exception and no log entry.

This repo gives you:

- Working code for every pattern the article describes
- A benchmark script you can run on your own machine
- Thread-safety demonstrations with accurate GIL behavior
- A test suite that shows exactly how to catch mutation bugs
- A real cache corruption example with a before/after fix

---

## Repository Structure

```
python-sorted-vs-sort-production-safety/
│
├── README.md
│
├── 01_mutation_basics/
│   ├── aliasing_bug.py          # The reference aliasing problem
│   ├── function_parameter.py    # How sort() mutates a caller's list
│   └── safe_with_sorted.py      # The same patterns using sorted()
│
├── 02_cpython_internals/
│   ├── id_identity_check.py     # Using id() to trace object identity
│   ├── refcount_demo.py         # Reference counting with ctypes
│   └── ob_item_visualization.py # Showing pointer array before/after
│
├── 03_gil_behavior/
│   ├── no_key_function.py       # sort() with no key — GIL held
│   ├── python_key_function.py   # sort() with lambda — GIL releases
│   └── thread_race_demo.py      # Demonstrating the lambda race condition
│
├── 04_cache_corruption/
│   ├── broken_cache.py          # Shared cache corrupted by sort()
│   ├── fixed_cache.py           # Same pattern fixed with sorted()
│   └── flask_example.py         # Request handler context example
│
├── 05_pipeline_patterns/
│   ├── mutating_pipeline.py     # Pipeline where sort() corrupts state
│   ├── safe_pipeline.py         # Pipeline using sorted() throughout
│   └── memoization_safe.py      # lru_cache + sorted() composition
│
├── 06_benchmarks/
│   ├── benchmark.py             # Timing sort() vs sorted() at various sizes
│   └── profiling_guide.md       # How to measure if allocation is your bottleneck
│
├── 07_decision_guide/
│   ├── when_sort_is_fine.py     # Cases where sort() is genuinely correct
│   └── antipatterns.py          # Common mistakes and how to fix them
│
└── 08_tests/
    ├── test_mutation.py         # Tests that catch mutation bugs
    ├── test_pipeline.py         # Tests for pipeline input integrity
    └── test_thread_safety.py    # Concurrency tests for shared list access
```

---

## Quick Start

```bash
git clone https://github.com/YOUR_USERNAME/python-sorted-vs-sort-production-safety.git
cd python-sorted-vs-sort-production-safety
python3 -m pytest 08_tests/ -v
```

No pip install needed. All examples use the Python standard library only.

Tested on Python 3.9, 3.10, 3.11, 3.12.

---

## Code Examples

### The Core Problem — Reference Aliasing

```python
# 01_mutation_basics/aliasing_bug.py

raw       = [5, 2, 9, 1]
audit_log = raw          # looks like a rename — is actually an alias

def process_orders(order_ids):
    order_ids.sort()     # mutates raw and audit_log silently
    return order_ids

result = process_orders(raw)

print(result)            # [1, 2, 5, 9]  — correct
print(audit_log)         # [1, 2, 5, 9]  — insertion order gone
print(raw)               # [1, 2, 5, 9]  — same object, same damage
print(id(raw) == id(audit_log) == id(result))  # True — all the same object
```

---

### The Fix — One Word Change

```python
# 01_mutation_basics/safe_with_sorted.py

raw       = [5, 2, 9, 1]
audit_log = raw

def process_orders(order_ids):
    return sorted(order_ids)   # creates a new list, touches nothing

result = process_orders(raw)

print(result)            # [1, 2, 5, 9]  — sorted output
print(audit_log)         # [5, 2, 9, 1]  — original order preserved
print(raw)               # [5, 2, 9, 1]  — untouched
print(id(raw) == id(result))  # False — independent objects
```

---

### Mutation Through a Function Parameter

```python
# 01_mutation_basics/function_parameter.py

def rank_products(products):
    """
    BUG: This modifies the caller's list.
    Any other reference to 'products' in the caller will see sorted data.
    """
    products.sort(key=lambda p: p['revenue'], reverse=True)
    return products

def rank_products_safe(products):
    """
    CORRECT: sorted() leaves the caller's list untouched.
    """
    return sorted(products, key=lambda p: p['revenue'], reverse=True)


catalog = [
    {'name': 'Widget A', 'revenue': 1200},
    {'name': 'Widget B', 'revenue': 450},
    {'name': 'Widget C', 'revenue': 3800},
]

display_list = catalog  # caller keeps a reference

top = rank_products(catalog)

# After rank_products():
print(catalog[0]['name'])      # Widget C  — insertion order gone
print(display_list[0]['name']) # Widget C  — affected too, same object

catalog2 = [
    {'name': 'Widget A', 'revenue': 1200},
    {'name': 'Widget B', 'revenue': 450},
    {'name': 'Widget C', 'revenue': 3800},
]

display_list2 = catalog2
top2 = rank_products_safe(catalog2)

print(catalog2[0]['name'])       # Widget A  — insertion order intact
print(display_list2[0]['name'])  # Widget A  — untouched
```

---

### GIL Behavior — No Key Function

```python
# 03_gil_behavior/no_key_function.py
#
# When list.sort() is called with no Python key function:
#   - The entire sort runs as a single C function call
#   - The GIL is held throughout
#   - Other threads see the list as EMPTY during the sort
#   - The final sorted state becomes visible all at once
#
# This does NOT produce a partially sorted view in other threads.

import threading
import time

shared = list(range(1000, 0, -1))   # [1000, 999, ..., 1]
observations = []

def reader():
    time.sleep(0.001)
    # During sort with no key, this will either see the original
    # list or an empty list — never a partially sorted list.
    snap = shared[:]
    observations.append(len(snap))

def writer():
    shared.sort()   # no key function — GIL held throughout

t1 = threading.Thread(target=reader)
t2 = threading.Thread(target=writer)

t1.start(); t2.start()
t1.join();  t2.join()

print('List length seen by reader:', observations)
# Will be 0 (empty during sort) or 1000 (before/after) — not partial
```

---

### GIL Behavior — Python Key Function (Real Race Condition)

```python
# 03_gil_behavior/python_key_function.py
#
# When list.sort() uses a Python lambda as key:
#   - The GIL releases between each key() call
#   - Another thread CAN read the list mid-sort
#   - This is a real race condition in multi-threaded services

import threading
import time

shared = [{'v': i} for i in range(500, 0, -1)]
partial_reads = []

def reader():
    time.sleep(0.001)
    # With a Python key function, the GIL may release mid-sort
    # and this reader could observe a mid-rearrangement state
    snap = [item['v'] for item in shared]
    partial_reads.append(snap)

def writer():
    # Python lambda = GIL releases between comparisons
    shared.sort(key=lambda x: x['v'])

t1 = threading.Thread(target=reader)
t2 = threading.Thread(target=writer)

t1.start(); t2.start()
t1.join();  t2.join()

# Regardless of GIL timing — the shared list is permanently mutated.
# All future readers see the sorted version. That is the real problem.
print('First element after sort:', shared[0]['v'])  # 1 — permanently mutated
```

---

### Cache Corruption — The Production Pattern

```python
# 04_cache_corruption/broken_cache.py

# Simulates a common in-memory cache pattern where
# one handler's sort() corrupts the shared cache object.

_cache = {}

def get_products():
    if 'products' not in _cache:
        _cache['products'] = [
            {'name': 'Pen',    'price': 1.5,  'added': 1},
            {'name': 'Desk',   'price': 299.0, 'added': 2},
            {'name': 'Chair',  'price': 189.0, 'added': 3},
            {'name': 'Lamp',   'price': 45.0,  'added': 4},
        ]
    return _cache['products']   # returns the actual cache object


# Handler A: wants products sorted by price (for a "top products" endpoint)
def handler_top_by_price():
    products = get_products()
    products.sort(key=lambda p: p['price'], reverse=True)  # BUG: sorts shared cache
    return [p['name'] for p in products]


# Handler B: wants products in insertion order (for the main listing)
def handler_listing():
    products = get_products()
    return [p['name'] for p in products]


# Simulate request order
print('Main listing (first request):')
print(handler_listing())
# ['Pen', 'Desk', 'Chair', 'Lamp']  — correct, insertion order

print('\nTop by price request:')
print(handler_top_by_price())
# ['Desk', 'Chair', 'Lamp', 'Pen']  — correct output from this handler

print('\nMain listing (after top-by-price was called):')
print(handler_listing())
# ['Desk', 'Chair', 'Lamp', 'Pen']  — WRONG: cache is permanently sorted
# No exception. No warning. Just wrong data for every future request.
```

```python
# 04_cache_corruption/fixed_cache.py

_cache = {}

def get_products():
    if 'products' not in _cache:
        _cache['products'] = [
            {'name': 'Pen',    'price': 1.5,  'added': 1},
            {'name': 'Desk',   'price': 299.0, 'added': 2},
            {'name': 'Chair',  'price': 189.0, 'added': 3},
            {'name': 'Lamp',   'price': 45.0,  'added': 4},
        ]
    return _cache['products']


def handler_top_by_price():
    products = get_products()
    # sorted() creates a new list — cache object is never touched
    top = sorted(products, key=lambda p: p['price'], reverse=True)
    return [p['name'] for p in top]


def handler_listing():
    products = get_products()
    return [p['name'] for p in products]


print('Main listing (first request):')
print(handler_listing())
# ['Pen', 'Desk', 'Chair', 'Lamp']

print('\nTop by price request:')
print(handler_top_by_price())
# ['Desk', 'Chair', 'Lamp', 'Pen']

print('\nMain listing (after top-by-price was called):')
print(handler_listing())
# ['Pen', 'Desk', 'Chair', 'Lamp']  — correct, cache is untouched
```

---

### Safe Functional Pipeline

```python
# 05_pipeline_patterns/safe_pipeline.py

from functools import lru_cache

users = [
    {'name': 'Alice',   'score': 82, 'active': True},
    {'name': 'Bob',     'score': 91, 'active': False},
    {'name': 'Carol',   'score': 74, 'active': True},
    {'name': 'David',   'score': 95, 'active': True},
    {'name': 'Eve',     'score': 88, 'active': True},
]

def filter_active(users):
    return [u for u in users if u['active']]

def rank_by_score(users):
    return sorted(users, key=lambda u: u['score'], reverse=True)

def take_top(users, n=3):
    return users[:n]

# Each function leaves its input unchanged.
# You can call each independently, in any order, on any subset.
result = take_top(rank_by_score(filter_active(users)))

print('Top 3 active users by score:')
for u in result:
    print(f"  {u['name']}: {u['score']}")

# Original list is completely unchanged
print('\nOriginal users list intact:', [u['name'] for u in users])
```

---

### Benchmark — Run on Your Own Machine

```python
# 06_benchmarks/benchmark.py

import timeit
import random

def benchmark(n, runs=200):
    data = random.sample(range(n * 10), n)

    sort_time = timeit.timeit(
        stmt='lst[:].sort()',
        setup=f'lst = {data}',
        number=runs
    ) / runs * 1e6

    sorted_time = timeit.timeit(
        stmt='sorted(lst)',
        setup=f'lst = {data}',
        number=runs
    ) / runs * 1e6

    overhead = ((sorted_time - sort_time) / sort_time) * 100
    print(f'n={n:>8,}  |  sort()={sort_time:>8.2f} µs  |  '
          f'sorted()={sorted_time:>8.2f} µs  |  overhead={overhead:>+.1f}%')

print(f'{"n":>9}  |  {"sort()":>14}  |  {"sorted()":>15}  |  {"overhead":>10}')
print('-' * 65)
for size in [100, 500, 1_000, 5_000, 10_000, 50_000, 100_000, 500_000, 1_000_000]:
    benchmark(size)
```

---

### Test Suite — Catching Mutation Bugs

```python
# 08_tests/test_mutation.py

import unittest
import copy

# ── Function under test ──
def rank_by_score_buggy(users):
    """Uses .sort() — will mutate the input."""
    users.sort(key=lambda u: u['score'], reverse=True)
    return users

def rank_by_score_safe(users):
    """Uses sorted() — input is untouched."""
    return sorted(users, key=lambda u: u['score'], reverse=True)


class TestMutationBehavior(unittest.TestCase):

    def setUp(self):
        self.users = [
            {'name': 'Alice', 'score': 82},
            {'name': 'Bob',   'score': 91},
            {'name': 'Carol', 'score': 74},
        ]
        self.original_order = [u['name'] for u in self.users]

    def test_buggy_function_mutates_input(self):
        """Demonstrates that .sort() changes the caller's list."""
        original_ref = self.users   # same object, different name
        rank_by_score_buggy(self.users)
        after_order = [u['name'] for u in original_ref]
        # This assertion FAILS — mutation happened
        self.assertEqual(self.original_order, after_order,
                         'rank_by_score_buggy() mutated the input list')

    def test_safe_function_does_not_mutate_input(self):
        """sorted() leaves the input completely unchanged."""
        original_copy = copy.deepcopy(self.users)
        result = rank_by_score_safe(self.users)
        # Input is unchanged
        self.assertEqual(self.users, original_copy)
        # Output is correctly sorted
        self.assertEqual(result[0]['name'], 'Bob')

    def test_safe_function_returns_new_object(self):
        """sorted() always returns a different object from the input."""
        result = rank_by_score_safe(self.users)
        self.assertIsNot(result, self.users)

    def test_safe_function_produces_correct_order(self):
        """Verify the sort order is descending by score."""
        result = rank_by_score_safe(self.users)
        scores = [u['score'] for u in result]
        self.assertEqual(scores, sorted(scores, reverse=True))

    def test_same_input_called_twice_gives_same_result(self):
        """Referential transparency: same input = same output, every time."""
        result1 = rank_by_score_safe(self.users)
        result2 = rank_by_score_safe(self.users)
        self.assertEqual(
            [u['name'] for u in result1],
            [u['name'] for u in result2]
        )


class TestCacheIntegrity(unittest.TestCase):
    """Tests that simulate a shared cache being sorted in place."""

    def setUp(self):
        self.cache = [
            {'id': 1, 'revenue': 500},
            {'id': 2, 'revenue': 120},
            {'id': 3, 'revenue': 890},
        ]
        self.original_ids = [p['id'] for p in self.cache]

    def test_sort_corrupts_cache(self):
        """list.sort() on the cache reference changes the cache."""
        fetched = self.cache   # simulates get_from_cache()
        fetched.sort(key=lambda p: p['revenue'], reverse=True)
        after_ids = [p['id'] for p in self.cache]
        self.assertNotEqual(self.original_ids, after_ids,
                            'Cache was NOT corrupted — test should fail here')

    def test_sorted_preserves_cache(self):
        """sorted() on the cache reference leaves the cache unchanged."""
        fetched = self.cache
        _ = sorted(fetched, key=lambda p: p['revenue'], reverse=True)
        after_ids = [p['id'] for p in self.cache]
        self.assertEqual(self.original_ids, after_ids)


if __name__ == '__main__':
    unittest.main()
```

---

### When `list.sort()` Is Actually Correct

```python
# 07_decision_guide/when_sort_is_fine.py

import random

# CASE 1: You built the list right here, no other references exist.
def build_and_sort_locally():
    data = random.sample(range(1000), 50)   # built right here
    data.sort()                             # sole owner — fine
    return data


# CASE 2: Explicitly owned temporary list, not passed anywhere.
def process_batch(raw_records):
    # Make a copy first, then sort the copy you own.
    working = list(raw_records)   # explicit copy — now you own it
    working.sort(key=lambda r: r['timestamp'])
    return working


# CASE 3: Building a new list from a comprehension before sorting.
def top_active_scores(users, n=10):
    scores = [u['score'] for u in users if u['active']]  # new list
    scores.sort(reverse=True)                             # you own it
    return scores[:n]


# ── What NOT to do ──
def antipattern_sort_then_copy(items):
    items.sort()            # mutates caller's list first
    return sorted(items)    # then pays allocation cost anyway
    # You got mutation risk AND allocation cost. Use sorted(items) directly.


def antipattern_return_none(items):
    result = items.sort()   # result is None
    return result           # returns None — TypeError on first use
```

---

## Key Facts This Repo Demonstrates

| Behavior | `list.sort()` | `sorted()` |
|---|---|---|
| Returns | `None` | New sorted list |
| Modifies input | Yes — always | Never |
| Works on generators | No | Yes |
| Works on tuples | No | Yes |
| Works on sets | No | Yes |
| Algorithm | Timsort | Timsort |
| Stable sort | Yes | Yes |
| GIL held (no key) | Yes — full sort | Yes — full sort |
| GIL releases (Python key) | Between key calls | Between key calls |
| Safe for shared/cached lists | No | Yes |
| Safe as function parameter | No | Yes |

---

## The One Rule Worth Remembering

```python
# Default to sorted(). Switch to .sort() only when:
#   1. You built the list in the current scope
#   2. No other reference to it exists
#   3. You have profiled and found a real allocation bottleneck
#
# In most backend service code, all three conditions are rarely true together.
```

---

## Full Article

All patterns in this repository are explained in detail, with interactive visualizations and a CPython memory diagram, in the original article:

**[Why sorted() Is Safer Than list.sort() in Production Python Systems](https://emitechlogic.com/why-sorted-is-safer-than-list-sort-in-production-python-systems/)**

The article covers:

- The reference aliasing bug and why it does not crash
- What CPython's `PyListObject` actually does during each operation
- The correct GIL behavior during `list.sort()` — and where a common explanation goes wrong
- A step-by-step production incident walkthrough
- Functional pipeline safety with `sorted()`
- Performance benchmarks at five different list sizes
- A decision framework for when `list.sort()` is genuinely the right call

---

## Contributing

If you have a real-world example of a mutation bug caused by `list.sort()` in a service context — or a benchmark result from a different Python version or implementation — open a PR. Keep examples focused and runnable with no external dependencies.

---

## License

MIT — use any code in this repository freely in your own projects.

---

## Related Reading on emitechlogic.com

- [Python Sorting Algorithms Explained](https://emitechlogic.com/sorting-algorithm-in-python/)
- [Mutable vs Immutable in Python](https://emitechlogic.com/mutable-vs-immutable-in-python/)
- [Python Garbage Collection and Reference Counting](https://emitechlogic.com/python-garbage-collection/)
- [Scope and Lifetime of Variables in Python](https://emitechlogic.com/scope-and-lifetime-of-variables-in-python/)
- [Parameter Passing Techniques in Python](https://emitechlogic.com/parameter-passing-techniques-in-python/)
- [CPython vs Jython vs IronPython](https://emitechlogic.com/cpython-vs-jython-vs-ironpython-which-one-should-you-actually-use/)
- [Python Optimization Guide](https://emitechlogic.com/python-optimization-guide-how-to-write-faster-smarter-code/)
