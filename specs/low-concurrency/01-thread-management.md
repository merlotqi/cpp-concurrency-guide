# Spec: 01-thread-management

## Metadata

- **File**: `low-concurrency/01-thread-management.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: no-concurrency/07-immutability
- **Estimated sections**: 6

## Learning Objectives

- Create, run, and destroy `std::thread` correctly
- Choose between `join()` and `detach()` — and understand the dangers of `detach()`
- Write exception-safe thread management (RAII thread wrapper)
- Use `std::jthread` (C++20) for automatic joining and stop tokens
- Determine the right number of threads for a workload

## Outline

### 1. Creating and Running Threads

- `std::thread` constructor: function pointer, lambda, `std::bind`, member function
- Thread starts immediately on construction — no separate "start" method
- The callable's arguments are copied (decay-copied) into the thread's storage
- Passing references: `std::ref` / `std::cref` — and the lifetime responsibility

### 2. join() vs detach()

- `join()`: block until thread finishes, clean up resources — the safe default
- `detach()`: thread runs independently, resources reclaimed on termination — use rarely
- Calling neither → `std::terminate` in destructor — the single biggest beginner mistake
- Valid use cases for `detach()`: daemon threads, fire-and-forget tasks

### 3. Exception-Safe Thread Management

- If an exception is thrown between thread creation and join → `std::terminate`
- **RAII thread wrapper**: join in destructor
- The `joining_thread` class (Anthony Williams' pattern)
- Now built into the standard as `std::jthread` (C++20)

### 4. std::jthread (C++20)

- Automatic join in destructor — no more `std::terminate` surprises
- **Stop tokens**: cooperative thread cancellation
- `std::stop_token` / `std::stop_source` / `std::stop_callback`
- Pattern: `while (!token.stop_requested()) { ... }`

### 5. How Many Threads?

- `std::thread::hardware_concurrency()` — hint, not a guarantee
- CPU-bound: N threads ≈ N cores (or N-1 to leave one for the OS)
- IO-bound: can oversubscribe — threads spend time waiting
- The danger of oversubscription: context switch overhead dominates

### 6. Thread Identity and Handles

- `std::this_thread::get_id()` — unique thread identifier
- `std::thread::native_handle()` — platform-specific handle (for pthread/Windows API)
- Thread-local random seed, thread naming (pthread_setname_np / SetThreadDescription)

## Key Concepts

| Concept | Depth |
|---------|-------|
| std::thread lifecycle | Full |
| join vs detach | Full |
| Exception safety (RAII) | Full |
| std::jthread + stop_token | Full |
| Thread count selection | Medium |

## Code Examples

- Basic thread creation with lambda capture
- RAII `joining_thread` class (before C++20)
- `std::jthread` with cooperative cancellation via stop_token
- Hardware concurrency heuristic

## Pitfalls

- ⚠️ Forgetting to join or detach → `std::terminate` (silent crash)
- ⚠️ Passing reference to local variable → dangling reference in thread
- ⚠️ `detach()` with access to stack variables → use-after-free
- ⚠️ `hardware_concurrency()` returns 0 when indeterminable
