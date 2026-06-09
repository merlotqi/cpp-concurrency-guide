# Spec: 10-thread-local-and-once

## Metadata

- **File**: `low-concurrency/10-thread-local-and-once.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 01-thread-management
- **Estimated sections**: 5

## Learning Objectives

- Use `thread_local` to give each thread its own copy of a variable — no synchronization needed
- Apply `std::call_once` + `std::once_flag` for thread-safe one-time initialization
- Implement the thread-safe singleton correctly in C++ (Meyer's singleton)
- Understand the performance characteristics of `thread_local` access

## Outline

### 1. thread_local Storage Duration

- Each thread gets its own independent instance
- Lifetime: starts on first access from a thread, destroyed on thread exit
- No synchronization needed between threads — each sees only its own copy
- Syntax: `thread_local int counter = 0;` (at namespace scope, block scope, or static member)
- Use case: per-thread caches, per-thread random number generators, per-thread allocators

### 2. thread_local Performance

- Access is typically via TLS (Thread-Local Storage) — an indirection through a segment register (FS on x86-64)
- Faster than mutex, slower than stack variable
- Dynamic TLS model (dlopen): may involve function call overhead
- Don't overuse: if you have 1000 `thread_local` variables × 100 threads = overhead
- Pattern: aggregate multiple thread_local items into a single `thread_local struct`

### 3. std::call_once + std::once_flag

- Guarantees exactly-once execution across all threads — regardless of contention
- `std::call_once(flag, func, args...)` — thread-safe, efficient
- If `func` throws, the flag is NOT set → next call will retry
- Better than "double-checked locking" (which is subtly broken without atomics)
- Use case: lazy initialization of shared resources

### 4. The Thread-Safe Singleton

- **Meyer's Singleton** (C++11+): `static T& get() { static T instance; return instance; }`
- C++11 guarantees thread-safe initialization of function-local statics — the compiler inserts equivalent of `call_once`
- This is the correct, simple, performant way to make a singleton
- No need for double-checked locking, no need for mutex

### 5. constinit (C++20)

- `constinit` guarantees that a variable is initialized at compile time or static init time
- Prevents the "static initialization order fiasco" across translation units
- `constinit thread_local int x = 42;` — compile-time init, no runtime overhead
- Use case: global constants that must be available before `main()`

## Key Concepts

| Concept | Depth |
|---------|-------|
| thread_local semantics and lifetime | Full |
| TLS performance model | Medium |
| call_once / once_flag | Full |
| Meyer's Singleton | Full |
| constinit | Light |

## Code Examples

- Per-thread random engine (avoiding contention on one global RNG)
- Per-thread allocation cache (mimalloc/tcmalloc style)
- `call_once` for lazy database connection initialization
- Meyer's Singleton: the one true C++ singleton
- Broken: double-checked locking (before C++11 atomics) — show why it's broken

## Pitfalls

- ⚠️ Overusing `thread_local` → memory bloat (N threads × M variables)
- ⚠️ Accessing `thread_local` for the first time may allocate — not signal-safe
- ⚠️ Double-checked locking pattern — use Meyer's Singleton instead
- ⚠️ `thread_local` in dynamically loaded libraries: complex destruction ordering
