# Spec: 07-atomic-fundamentals

## Metadata

- **File**: `low-concurrency/07-atomic-fundamentals.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 02-mutex-basics, no-concurrency/05-happens-before
- **Estimated sections**: 7

## Learning Objectives

- Understand what `std::atomic<T>` guarantees: indivisibility, coherence, and (with ordering) visibility
- Master the core operations: `load`, `store`, `exchange`, `compare_exchange_strong`, `compare_exchange_weak`
- Know `is_lock_free()` — and why it matters for performance and signal safety
- Use `std::atomic_flag` as a simple spinlock
- Distinguish when to use atomics vs mutexes

## Outline

### 1. What Does Atomic Mean?

- Indivisible: no thread can observe a partially-completed operation
- The operations are the synchronization points — not the type itself
- `std::atomic<int>` is a different type from `int` — no implicit conversion
- Specializations: integral, pointer, `atomic<bool>`, `atomic_flag`, and `atomic<T>` (user-defined, with restrictions)

### 2. Core Operations

| Operation | Description | Returns |
|-----------|-------------|---------|
| `load()` | Read value | T |
| `store(val)` | Write value | void |
| `exchange(val)` | Swap, return old | T |
| `compare_exchange_weak(expected, desired)` | CAS — conditional swap | bool |
| `compare_exchange_strong(expected, desired)` | CAS with strong guarantee | bool |
| `fetch_add(val)` | Add, return old (integral/pointer only) | T |
| `fetch_sub(val)` | Subtract, return old | T |

### 3. Compare-and-Swap (CAS) — The Universal Primitive

- `compare_exchange_weak(expected, desired)`: if `*this == expected`, set to `desired`; otherwise set `expected = *this`
- Spurious failure: `weak` version may fail even when `*this == expected` (cheaper on some architectures, e.g., ARM LDREX/STREX)
- `strong` version: guaranteed to succeed iff `*this == expected` (more expensive)
- **Always put CAS in a loop with `weak`** unless you have a reason not to
- Example: atomic increment with CAS loop

### 4. std::atomic_flag — The Simplest Atomic

- Always lock-free (guaranteed by the standard)
- `test_and_set()`: set to true, return old value
- `clear()`: set to false
- `test()`: read without modifying (C++20)
- Building a spinlock with `atomic_flag` (~10 lines)

### 5. is_lock_free()

- `atomic<T>::is_lock_free()`: does this atomic use hardware instructions or internal mutex?
- Lock-free is needed for: signal handlers, real-time code, avoiding contention
- Platform dependence: `atomic<int>` is always lock-free; `atomic<BigStruct>` probably not
- `is_always_lock_free` (C++17): compile-time constant

### 6. Atomics vs Mutexes

| Criterion | Atomic | Mutex |
|-----------|--------|-------|
| Overhead | Very low (ns) | Higher (ns to µs) |
| Blocking | No (spin or fail) | Yes (OS schedules) |
| Complexity | High (memory ordering) | Low |
| Composability | Single word only | Arbitrary data |
| Use case | Counters, flags, state machines | Complex data structures |

### 7. The fetch_xxx Family (Integral & Pointer)

- `fetch_add`, `fetch_sub`, `fetch_and`, `fetch_or`, `fetch_xor`
- Return the **old** value (pre-increment semantics)
- Operator overloads: `++`, `--`, `+=`, `-=`, `&=`, `|=`, `^=` — but these return new value, different from fetch_xxx!
- Memory order can be specified: `counter.fetch_add(1, std::memory_order_relaxed)`

## Key Concepts

| Concept | Depth |
|---------|-------|
| load / store | Full |
| exchange | Full |
| compare_exchange_weak/strong | Full |
| CAS loop pattern | Full |
| atomic_flag spinlock | Full |
| is_lock_free | Medium |

## Code Examples

- Atomic counter with `fetch_add`
- CAS loop: `do { old = x.load(); new = f(old); } while (!x.compare_exchange_weak(old, new));`
- Spinlock with `atomic_flag`
- Broken: non-atomic increment → demonstrate with TSan
- `is_lock_free` check for portability

## Pitfalls

- ⚠️ Forgetting the CAS loop — a single CAS may spuriously fail (weak) or genuinely fail
- ⚠️ Using `exchange` when you wanted `fetch_add` (exchange overwrites, doesn't add)
- ⚠️ Assuming `atomic<T>` is always lock-free — check `is_lock_free()`
- ⚠️ `atomic<T>::operator=` is equivalent to `store()`, not `exchange()`
