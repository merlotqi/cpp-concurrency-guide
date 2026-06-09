# Spec: 04-shared-mutex

## Metadata

- **File**: `low-concurrency/04-shared-mutex.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 03-lock-management
- **Estimated sections**: 5

## Learning Objectives

- Understand the **readers-writer** pattern and when it applies
- Use `std::shared_mutex` (C++17) for multiple-reader, single-writer synchronization
- Pair `shared_lock` with reads and `unique_lock` with writes
- Recognize the hidden cost: shared_mutex is heavier than plain mutex — don't use blindly

## Outline

### 1. The Readers-Writer Problem

- Many readers, few writers: readers don't conflict with each other
- Classic use case: a cache that's read 1000× per second but updated once per minute
- With plain mutex: all reads serialize → bottleneck
- With shared_mutex: reads proceed concurrently → throughput scales with cores

### 2. std::shared_mutex API

- `lock()` / `unlock()`: exclusive (writer) access
- `lock_shared()` / `unlock_shared()`: shared (reader) access
- `try_lock()` / `try_lock_shared()`: non-blocking variants
- C++14: `std::shared_timed_mutex` adds `try_lock_for`/`try_lock_until`

### 3. RAII Pairing

- Writer: `std::unique_lock<std::shared_mutex>` → calls `lock()`/`unlock()`
- Reader: `std::shared_lock<std::shared_mutex>` → calls `lock_shared()`/`unlock_shared()`
- They are the same lock classes, just parameterized on different mutex types

### 4. Performance Considerations

- `std::shared_mutex` is larger and slower than `std::mutex` (more internal state)
- **Break-even point**: if reads are short and frequent, shared_mutex wins. If critical section is tiny (a few ns), mutex may be faster because it's lighter
- Benchmark pattern: measure throughput at 1, 2, 4, 8, 16 readers — find the crossover
- Rule of thumb: read:write ratio > 10:1 AND read section > mutex overhead → use shared_mutex

### 5. Writer Starvation

- If readers keep arriving, a writer may wait forever
- `std::shared_mutex` does NOT guarantee writer priority (implementation-defined)
- Mitigation: use a writer-priority flag or limit read concurrency
- Brief mention of `std::osyncstream` and other standard facilities

## Key Concepts

| Concept | Depth |
|---------|-------|
| Readers-writer pattern | Full |
| shared_mutex + shared_lock | Full |
| Performance trade-off | Full |
| Writer starvation | Medium |

## Code Examples

- DNS cache: many reader threads, occasional writer update
- Before/After benchmark: `std::mutex` vs `std::shared_mutex` at various reader counts
- `shared_lock` with `try_lock_shared` — "read if available, skip if writer active"

## Pitfalls

- ⚠️ Using shared_mutex when read section is trivial — mutex may be faster
- ⚠️ Writer starvation in read-heavy workloads
- ⚠️ Upgrading from shared to exclusive lock — `shared_mutex` doesn't support upgrade; you must unlock_shared then lock (gap vulnerability)
