# Spec: 09-future-promise-async

## Metadata

- **File**: `low-concurrency/09-future-promise-async.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 01-thread-management
- **Estimated sections**: 6

## Learning Objectives

- Use `std::async` to launch asynchronous tasks with minimal ceremony
- Understand `std::future` and `std::promise` as a one-shot channel between threads
- Know `std::packaged_task` for wrapping callables into futures
- Use `std::shared_future` when multiple consumers need the same result
- Choose between `std::launch::async` and `std::launch::deferred`

## Outline

### 1. std::async — Fire and (Eventually) Forget

- `auto future = std::async(std::launch::async, func, args...)` — runs on a new thread
- `auto future = std::async(std::launch::deferred, func, args...)` — runs lazily on `future.get()`
- Default: `std::launch::async | std::launch::deferred` — implementation chooses (surprise!)
- **Always specify the launch policy explicitly** — the default is ambiguous

### 2. std::future<T> — The Receiving End

- `future.get()`: blocks until result is ready, returns T (or rethrows exception)
- One-shot: can only call `get()` once — second call is UB
- `future.wait()`: block without consuming the result
- `future.wait_for(duration)` / `wait_until(time_point)`: timed wait → `future_status::ready/timeout/deferred`
- `future.valid()`: has a shared state? (false after get, or if default-constructed)

### 3. std::promise<T> — The Sending End

- `promise.set_value(val)`: send result to the paired future
- `promise.set_exception(ptr)`: send an exception instead
- `promise.get_future()`: get the associated future (call once)
- Promise-future pair = one-shot thread-safe channel
- Use case: thread returns multiple values, or return value + exception

### 4. std::packaged_task — Callable → Future

- Wraps any callable and associates it with a future
- `packaged_task<R(Args...)> task(func); future = task.get_future(); task(args...);`
- Can be moved into `std::thread` for deferred execution
- Use case: queue of tasks for a thread pool

### 5. std::shared_future — Multiple Consumers

- `future.share()` → returns `shared_future` (invalidates the future)
- Multiple threads can call `get()` on the same `shared_future`
- All callers receive the same result (copies)
- Use case: broadcast a one-time result to multiple threads

### 6. Exception Propagation

- Exceptions thrown in async/task are captured and rethrown on `future.get()`
- `promise.set_exception()` with `std::current_exception()`
- The future's destructor does NOT join — if the future is the last reference, the shared state persists until the async task completes (fire-and-forget is OK)
- But: a future from `std::async` with `std::launch::async` will block in destructor (join-like behavior)

## Key Concepts

| Concept | Depth |
|---------|-------|
| std::async + launch policies | Full |
| future / promise pair | Full |
| packaged_task | Medium |
| shared_future | Medium |
| Exception propagation | Medium |

## Code Examples

- `std::async`: parallel sum of array halves
- `promise/future`: worker thread computes result, main thread gets it
- `packaged_task`: task queue with GUI thread picking up results
- `shared_future`: notify multiple threads when initialization is complete
- Broken: calling `future.get()` twice → UB
- Broken: not specifying launch policy → surprising synchronous execution

## Pitfalls

- ⚠️ Default launch policy is `async|deferred` — the implementation chooses; you may get synchronous execution unexpectedly
- ⚠️ `future.get()` called twice → UB (use `shared_future` if needed)
- ⚠️ Future from `std::async` blocks in destructor (wait for completion) — other futures don't
- ⚠️ Promise destroyed without `set_value`/`set_exception` → `future_error` (broken_promise) on `get()`
