# Spec: 08-thread-pool

## Metadata

- **File**: `high-concurrency/08-thread-pool.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 07-blocking-lockfree-queue, low-concurrency/09-future-promise-async
- **Estimated sections**: 7

## Learning Objectives

- Design and implement a **work-stealing thread pool** from scratch
- Understand task queue design: one shared queue vs per-worker queues + work-stealing
- Use `std::packaged_task` + `std::future` for submitting tasks with return values
- Handle thread pool shutdown correctly: drain pending tasks, join all workers
- Know the design space: fixed-size, cached, fork-join, and the C++17 parallel algorithms relationship

## Outline

### 1. The Thread Pool Abstraction

- A pool of pre-created worker threads that execute submitted tasks
- Avoids thread creation/destruction overhead (can be ~50µs per thread)
- Controls concurrency: N threads = at most N concurrent tasks
- The fundamental abstraction for CPU-bound parallel work

### 2. Simple Thread Pool (Shared Queue)

- One MPSC (or MPMC) queue shared by all workers
- `submit(task)`: push to queue, notify one worker
- Worker loop:
  ```
  while (!stop) {
      if (try_pop(task)) {
          task();
      } else {
          wait(); // blocking when idle
      }
  }
  ```
- Simple, correct, but: all workers contend on the same queue → bottleneck

### 3. Work-Stealing Thread Pool

- Each worker has its own SPSC/MPSC queue
- `submit(task)`: push to current thread's local queue (or random worker's queue)
- Worker loop:
  ```
  while (!stop) {
      if (try_pop_local(task)) { task(); }
      else if (try_steal_from_random(task)) { task(); }
      else { wait(); }
  }
  ```
- Stealing: take from the **opposite end** of another worker's queue
  - Owner pushes/pops from the **tail** (LIFO — cache-hot tasks)
  - Thieves steal from the **head** (FIFO — fairness)
- This is the classic work-stealing deque (double-ended queue)
- Source: Cilk, Java ForkJoinPool, TBB, .NET Task Parallel Library

### 4. Work-Stealing Deque Implementation

- The deque (double-ended queue) supports:
  - `push_bottom(task)`: owner adds task (fast path, no CAS if not empty)
  - `pop_bottom()`: owner takes task (fast path, no CAS if at least 2 items)
  - `steal()`: thief takes from top (CAS on top index)
- Key: owner operations are lock-free and don't contend with each other
- Only contention: thief vs owner on the same queue's boundary case
- Full implementation uses a ring buffer + two indices (top, bottom)

### 5. Task Submission and Futures

- `submit(func) -> std::future<decltype(func())>`
- Internally: wrap `func` in `std::packaged_task<R()>`, get `future`, push task, return `future`
- Support for: `submit([] { return 42; })` → `std::future<int>`
- Support for: `submit([](int a, int b) { return a + b; }, 3, 4)` → bind arguments
- Exception propagation: task exception → future.get() rethrows

### 6. Shutdown Protocol

- Problem: how to stop workers that are blocking on `wait()`?
- Solution 1: `stop_source` → `request_stop()` → workers check `token.stop_requested()`, process remaining tasks, exit
- Solution 2: Poison pill — push a special "stop" task for each worker
- `ThreadPool::~ThreadPool()`:
  1. Set stop flag
  2. Notify all workers
  3. Join all threads
- Drain policy: finish pending tasks or discard? (offer a parameter)

### 7. Performance and Tuning

- Worker count: `std::thread::hardware_concurrency()` — good default
- Over-subscription: IO-bound tasks can use more threads than cores
- Task granularity: too fine (1µs tasks) → overhead dominates; too coarse (1s tasks) → load imbalance
- Rule of thumb: task should be >10µs for thread pool to beat synchronous execution
- Work-stealing overhead: ~20-50ns for push/pop on uncontended deque; ~100-200ns for steal

## Key Concepts

| Concept | Depth |
|---------|-------|
| Shared queue pool design | Full |
| Work-stealing deque | Full |
| Task submit + future | Full |
| Shutdown protocol | Full |
| Granularity considerations | Medium |

## Code Examples

- Simple `ThreadPool` with shared MPSC queue (~80 lines)
- Work-stealing `ThreadPool` with per-worker deques (~200 lines)
- Deque class: push_bottom, pop_bottom, steal (~80 lines)
- Benchmark: sum of array with thread pool, varying task size
- Benchmark: thread pool vs `std::async` (creation overhead)

## Cross-References

- Queue implementations: `high-concurrency/03` through `07`
- Future/promise: `low-concurrency/09-future-promise-async`
- OS wait: `high-concurrency/01-os-sync-primitives`
- Task parallelism: `high-concurrency/09-coroutines`

## Pitfalls

- ⚠️ Shared queue bottleneck → use work-stealing for CPU-bound, shared queue OK for IO-bound
- ⚠️ Shutdown deadlock: worker waits on queue, destructor waits on worker → need notify or stop_token
- ⚠️ Task capturing dangling references: `submit([&x] { ... })` → `x` goes out of scope before task runs
- ⚠️ Exception in task → must store and rethrow in future.get(), not crash the worker
- ⚠️ Starvation in work-stealing: if all tasks are long-running, new tasks pile up → consider LIFO for owner, FIFO for thieves
