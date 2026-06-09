# Spec: 03-lock-management

## Metadata

- **File**: `low-concurrency/03-lock-management.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 02-mutex-basics
- **Estimated sections**: 5

## Learning Objectives

- Master all RAII lock wrappers: `lock_guard`, `unique_lock`, `scoped_lock`, `shared_lock`
- Understand the flexibility vs cost trade-off of each
- Use `std::lock` and `std::scoped_lock` to avoid deadlock when acquiring multiple mutexes
- Know when to use `defer_lock`, `adopt_lock`, `try_to_lock` tags

## Outline

### 1. std::lock_guard — The Simplest RAII Lock

- Locks on construction, unlocks on destruction — zero overhead
- Non-copyable, non-movable
- The default choice: use unless you need more flexibility
- Pattern: `{ std::lock_guard lock(mtx); /* critical section */ }`

### 2. std::unique_lock — The Flexible Lock

- Deferred locking: `std::unique_lock lock(mtx, std::defer_lock)`
- Try-lock: `std::unique_lock lock(mtx, std::try_to_lock)`
- Adopt existing lock: `std::unique_lock lock(mtx, std::adopt_lock)`
- Movable — can transfer ownership between scopes
- Can unlock/lock multiple times (`.unlock()` / `.lock()`)
- Cost: slightly larger than `lock_guard` (stores a `bool` for ownership)

### 3. std::scoped_lock (C++17) — Multi-Mutex Deadlock Avoidance

- Variadic template: `std::scoped_lock lock(mtx1, mtx2, mtx3)`
- Uses `std::lock` internally — deadlock-free locking algorithm
- Replaces `lock_guard` for the general case in C++17+
- Preferred over bare `lock_guard` when locking any mutex

### 4. std::shared_lock — For Shared (Read) Access

- Works with `std::shared_mutex` (next chapter)
- Allows multiple readers to hold the lock concurrently
- `unique_lock` for writing, `shared_lock` for reading
- RAII, same interface as `unique_lock`

### 5. Lock Tags Reference

| Tag | Effect |
|-----|--------|
| `std::defer_lock` | Don't lock on construction |
| `std::try_to_lock` | Try to lock, don't block |
| `std::adopt_lock` | Assume mutex already locked |

## Key Concepts

| Concept | Depth |
|---------|-------|
| lock_guard | Full |
| unique_lock | Full |
| scoped_lock | Full |
| Deadlock-free multi-lock | Full |
| Lock tags | Reference |

## Code Examples

- `lock_guard` for simple critical section
- `unique_lock` with deferred lock + manual lock/unlock cycle
- `unique_lock` with try_to_lock — "process if available, otherwise skip"
- `scoped_lock` locking two mutexes simultaneously
- Transferring lock ownership via `unique_lock` move

## Pitfalls

- ⚠️ Using `lock_guard` when you need to lock multiple mutexes → deadlock risk
- ⚠️ `unique_lock` with `adopt_lock` on an unlocked mutex → UB
- ⚠️ `unique_lock::owns_lock()` returns false with `defer_lock` before locking
