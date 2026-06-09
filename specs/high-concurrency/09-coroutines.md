# Spec: 09-coroutines

## Metadata

- **File**: `high-concurrency/09-coroutines.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 08-thread-pool, low-concurrency/09-future-promise-async
- **Estimated sections**: 7

## Learning Objectives

- Understand the C++20 coroutine model: **stackless**, **suspend/resume**, **promise type**
- Use `co_await`, `co_yield`, and `co_return` effectively
- Build a simple `Generator<T>` for lazy sequences
- Build a `Task<T>` for asynchronous computation (like Rust's Future or JavaScript's Promise)
- Integrate coroutines with the thread pool from Chapter 08 for async execution
- Recognize when coroutines help concurrency (IO-bound) vs when they don't (CPU-bound)

## Outline

### 1. Stackless Coroutines: What and Why

- Not threads — no preemption, no stack per coroutine
- A coroutine is a function that can **suspend** and **resume**
- The compiler transforms the function body into a state machine
- Key benefit for concurrency: IO-bound code reads like synchronous code but doesn't block a thread
- Comparison: threads (preemptive, costly), callbacks (efficient, ugly), coroutines (efficient, readable)
- C++20 coroutines are **stackless**: only the coroutine body is suspended, not the entire call stack
- Stackful coroutines (boost::coroutine2, boost::fiber) have their own stack — heavier but can suspend from nested calls

### 2. The Three Keywords

| Keyword | Meaning |
|---------|---------|
| `co_await expr` | Suspend until `expr` is ready |
| `co_yield value` | Suspend and produce a value (generator) |
| `co_return` / `co_return value` | Complete the coroutine |

- A function becomes a coroutine if its body contains any of the three
- The return type must be a "promise type" compatible type (the compiler instantiates the promise via `std::coroutine_traits`)

### 3. The Coroutine Machinery (What the Compiler Generates)

- **Promise type**: controls coroutine behavior (what happens on suspend, resume, return, exception)
- **Awaitable**: any type with `await_ready()`, `await_suspend(handle)`, `await_resume()`
- **Coroutine handle**: `std::coroutine_handle<P>` — non-owning pointer to the coroutine frame
- Typical lifecycle:
  1. Compiler allocates coroutine frame on heap
  2. Calls `promise.get_return_object()` → the return value the caller sees
  3. Runs coroutine body until `co_await` → calls `awaitable.await_suspend(handle)` → suspends
  4. Later: `handle.resume()` → calls `awaitable.await_resume()` → continues from `co_await`
  5. `co_return` → calls `promise.return_value(val)` or `promise.return_void()` → destroys frame

### 4. Building a Generator<T>

```
Generator<int> range(int n) {
    for (int i = 0; i < n; ++i) {
        co_yield i;
    }
}

// Usage:
for (int v : range(10)) { std::cout << v << '\n'; }
```

- Generator: simplest coroutine type — co_yield only, no co_await
- The promise type holds the current value; co_yield suspends; caller reads value; caller resumes
- Memory: one heap allocation per generator (HALO — Heap Allocation eLision Optimization — may eliminate it)
- Walk through the state machine: range(10) generates states for i=0, i=1, ..., co_return

### 5. Building a Task<T> (Async Computation)

```
Task<int> compute_async() {
    int result = co_await async_read_file("data.txt");
    co_return result;
}
```

- `Task<T>`: a coroutine that can `co_await` other awaitables, eventually `co_return` a value
- The `.await_suspend()` is where integration with thread pool happens:
  ```
  void await_suspend(coroutine_handle<> h) {
      thread_pool.submit([h] { h.resume(); });
  }
  ```
- `co_await thread_pool.schedule()` → suspends current coroutine, submits it to thread pool → resumes on a worker thread
- This enables: N concurrent IO operations on a single thread → each `co_await` suspends, the thread picks up another coroutine

### 6. Coroutines and Concurrency

| Pattern | Use Case | Concurrency Benefit |
|---------|----------|-------------------|
| Generator | Lazy sequences, ranges | None (single-threaded) |
| Task + IO | File IO, network, DB queries | Many pending IO ops on few threads |
| Task + Thread Pool | CPU work with async dispatch | Cooperative multitasking, work scheduling |
| Async streams (C++23 `std::generator`) | Data pipelines | Producer-consumer without buffer management |

- Coroutines do NOT make CPU-bound code faster — they make IO-bound code use fewer threads
- Thread pool + coroutines: submit tasks that `co_await` → thread pool workers execute and resume coroutines
- Example: a web server with 1000 concurrent connections on 8 threads using coroutines

### 7. Current Limitations (C++20) and Future (C++23/26)

- C++20: language support only → no standard library coroutine types (Generator, Task, etc.)
- C++23: `std::generator<T>` for generators, but no `std::task<T>` yet
- C++26: expected additions — `std::task<T>`, `std::lazy<T>`, async algorithms
- Third-party for now: cppcoro, libunifex, ASIO coroutines
- The ecosystem is catching up; understanding the low-level machinery prepares you for the library versions

## Key Concepts

| Concept | Depth |
|---------|-------|
| Stackless coroutine model | Full |
| co_await / co_yield / co_return | Full |
| Promise type + awaitable | Full |
| Generator implementation | Full |
| Task + thread pool integration | Full |
| HALO (heap allocation elision) | Light |
| C++23/26 outlook | Light |

## Code Examples

- `Generator<int>` full implementation (~60 lines)
- `Task<T>` with thread pool integration (~120 lines)
- IO simulation: `co_await async_read("file.txt")` with `sleep_for` as IO stand-in
- Web request simulation: 1000 concurrent "requests" on 4 threads with coroutines
- Comparison: 1000 threads (heavy) vs 1000 callbacks (ugly) vs coroutines on thread pool (elegant)

## Cross-References

- Thread pool: `high-concurrency/08-thread-pool`
- Future/promise: `low-concurrency/09-future-promise-async`
- OS wait: `high-concurrency/01-os-sync-primitives` (for IO completion notification)

## Pitfalls

- ⚠️ Coroutine frame is heap-allocated by default → can be a hidden cost; use HALO (co_yield in a scope where the coroutine is immediately consumed) when possible
- ⚠️ Dangling references in coroutine captures → the coroutine outlives the captured reference → UB. Always capture by value or shared_ptr.
- ⚠️ Forgetting to `co_await` a Task → it never executes → silent no-op (some libraries warn)
- ⚠️ Recursive `co_await` without symmetric transfer → stack overflow → use `await_suspend` that returns the next coroutine handle (symmetric transfer, C++20 feature)
- ⚠️ Coroutines are NOT a replacement for threads for CPU work — they don't magically parallelize
