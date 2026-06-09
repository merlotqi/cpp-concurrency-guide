# Spec: 11-producer-consumer-basic

## Metadata

- **File**: `low-concurrency/11-producer-consumer-basic.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 06-condition-variable, 03-lock-management
- **Estimated sections**: 5

## Learning Objectives

- Implement a **bounded thread-safe queue** using mutex + condition variable
- Understand the two-condition-variable pattern (not_full + not_empty)
- Write a benchmark harness for producer-consumer throughput
- Establish the **baseline** that lock-free queues (Phase III) will be compared against

## Outline

### 1. The Bounded Queue Problem

- Fixed-size ring buffer, multiple producers, multiple consumers
- Producers wait when queue is full; consumers wait when queue is empty
- This is the "hello world" of concurrent data structures
- We implement it with the tools from Phase II so we can later see why lock-free is faster

### 2. Implementation: Mutex + Two Condition Variables

```
template<typename T>
class BlockingQueue {
    std::mutex mtx;
    std::condition_variable not_full;
    std::condition_variable not_empty;
    std::vector<T> buffer;
    size_t head = 0, tail = 0, count = 0;
    
    void push(T item) {
        std::unique_lock lock(mtx);
        not_full.wait(lock, [this] { return count < capacity; });
        buffer[tail] = std::move(item);
        tail = (tail + 1) % capacity;
        ++count;
        not_empty.notify_one();
    }
    
    T pop() {
        std::unique_lock lock(mtx);
        not_empty.wait(lock, [this] { return count > 0; });
        T item = std::move(buffer[head]);
        head = (head + 1) % capacity;
        --count;
        not_full.notify_one();
        return item;
    }
};
```

### 3. Why Two Condition Variables?

- One CV for "queue not empty" → consumers wait
- One CV for "queue not full" → producers wait
- If you use one CV and `notify_all`: thundering herd — all threads wake, N-1 go back to sleep
- Two CVs + `notify_one`: only a single waiter of the right type wakes
- Performance difference: significant at scale (10+ threads)

### 4. Benchmark Harness

- N producers, M consumers, queue capacity C, total items I
- Measure: throughput (ops/sec), latency (per op time)
- Variables to test: thread count, queue capacity, item size
- This harness will be reused for every lock-free queue in Phase III
- Tools: `std::chrono`, Google Benchmark (optional)

### 5. Baseline Results & Analysis

- Expected throughput for mutex+CV bounded queue on a typical x86-64 machine
- Contention analysis: where time is spent (kernel entry for futex, context switches)
- Why this implementation has a ceiling: mutex contention + condition variable syscalls
- Preview: lock-free queue eliminates the mutex (contention) and optionally the CV syscalls (blocking)

## Key Concepts

| Concept | Depth |
|---------|-------|
| Bounded queue with mutex+CV | Full |
| Two-CV pattern | Full |
| Benchmark methodology | Full |
| Contention analysis | Medium |

## Code Examples

- Full `BlockingQueue` implementation (~50 lines)
- Benchmark: `push/pop` throughput with 1P1C, 4P4C, 8P8C
- Benchmark: latency distribution (p50, p99, p999)
- Single-CV version (for comparison — shows the thundering herd problem)

## Cross-References

- **Direct comparison target** for:
  - `high-concurrency/03-spsc-ring-buffer`
  - `high-concurrency/04-spmc-queue`
  - `high-concurrency/05-mpsc-queue`
  - `high-concurrency/06-mpmc-ring-buffer`
  - `high-concurrency/07-blocking-lockfree-queue`

## Pitfalls

- ⚠️ Using `notify_all` with one CV for both conditions → thundering herd
- ⚠️ Notify outside the lock: can be faster but risks "lost wakeup" in subtle cases (use the predicate pattern)
- ⚠️ Using `notify_one` when N producers are waiting for not_full and only 1 consumer pops → other N-1 producers miss the signal. In this case, notify_one is correct because only one slot opened — but only if the waiter processes exactly one slot
