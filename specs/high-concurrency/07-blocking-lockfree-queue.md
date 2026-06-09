# Spec: 07-blocking-lockfree-queue

## Metadata

- **File**: `high-concurrency/07-blocking-lockfree-queue.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 01-os-sync-primitives, 03-spsc-ring-buffer, 06-mpmc-ring-buffer
- **Estimated sections**: 6

## Learning Objectives

- Combine **OS-level wait/notify** (Chapter 01) with **lock-free ring buffers** (Chapters 03-06)
- Implement blocking push/pop that sleep when queue is full/empty instead of spinning
- Understand the hybrid fast-path/slow-path design: spin first, block after N attempts
- Build a queue that matches `std::mutex` + `std::condition_variable` API but with lock-free fast path
- Benchmark: blocking lock-free vs mutex+CV vs pure spin

## Outline

### 1. Why Blocking Matters

- Pure lock-free queues spin on CAS failure or full/empty → wastes CPU
- In low-load scenarios, spinning is fine (and fast)
- In high-load or bursty scenarios: producer spins while queue is full → 100% CPU for nothing
- The solution: **adaptive wait** — try lock-free N times, then block on OS primitive
- This is how real systems work: fast path userspace, slow path kernel

### 2. Adding Blocking to SPSC

- Start simple: add blocking to the SPSC ring buffer from Chapter 03
- `push_blocking(item)`: try `try_push` → if full, `write_pos_.wait(pos, relaxed)` — block until consumer notifies
- `pop_blocking(item)`: try `try_pop` → if empty, `read_pos_.wait(pos, relaxed)` — block until producer notifies
- Producer's `try_push` after a successful push: `write_pos_.notify_one()` to wake a consumer
- Consumer's `try_pop` after a successful pop: `read_pos_.notify_one()` to wake a producer
- This is the simplest hybrid design

### 3. Adding Blocking to MPMC

- MPMC: multiple waiters on both ends → `notify_one` wakes one; `notify_all` wakes all
- Challenge: a producer wakes on `read_pos_` notification, but by the time it runs, the queue may be full again (another producer stole the slot)
- This is the same problem as condition variable spurious wakeup → always re-check condition in a loop
- Pattern: `while (!try_push(item)) { write_pos_.wait(old_pos, relaxed); }`
- `notify_all` on full/empty transitions to avoid missed wakeups

### 4. Adaptive Spinning (Hybrid Design)

- Pure blocking: every empty/full → syscall → ~1-10µs. Good for long waits, bad for short waits.
- Pure spinning: every empty/full → burn CPU. Good for microsecond-scale waits, bad for long waits.
- **Adaptive**: spin N times (e.g., 16), then block. Covers both cases:
  ```
  for (int spin = 0; spin < SPIN_LIMIT; ++spin) {
      if (try_push(item)) { notify_one(); return; }
  }
  while (!try_push(item)) { write_pos_.wait(old_pos, relaxed); }
  notify_one();
  ```
- SPIN_LIMIT tuning: depends on (context switch cost) / (try_push cost). Typically 8-64.
- Some implementations use exponential backoff: spin 1, 2, 4, 8, 16... then block

### 5. Comparison with std::mutex + std::condition_variable

| Feature | mutex+CV | Blocking Lock-Free |
|---------|----------|-------------------|
| Uncontended push | ~50ns (lock+unlock) | ~10ns (atomic store) |
| Contended push | Blocks via futex | Spin then block via futex |
| Uncontended pop | ~50ns | ~10ns (atomic load) |
| Empty pop | Blocks via CV/futex | Spin then block via atomic::wait |
| Wakeup mechanism | CV notify | atomic::notify |
| Memory overhead | mutex(~40B) + CV(~48B) | 2 atomic<uint64_t> (~16B) |

### 6. Real-World Usage

- When your queue is empty 99% of the time: use blocking (don't spin 99%)
- When your queue is empty 1% of the time: use pure lock-free (spinning 1% is cheap)
- When you don't know: use adaptive
- Production examples: Linux io_uring SQ/CQ (spinning + `io_uring_enter` syscall for blocking), DPDK rte_ring (pure spin, designed for always-busy scenarios)

## Key Concepts

| Concept | Depth |
|---------|-------|
| Hybrid spin-then-block | Full |
| atomic::wait as CV replacement | Full |
| Adaptive waiting strategy | Full |
| notify_one vs notify_all in MPMC | Full |
| Fast-path/slow-path design | Full |

## Code Examples

- `BlockingSPSCQueue`: SPSC + atomic::wait (~120 lines)
- `BlockingMPMCQueue`: MPMC turn-based + atomic::wait (~150 lines)
- Adaptive spin: spin limit 16, then block
- Benchmark: empty-queue scenario (producer slow, consumer fast) → compare CPU utilization
- Benchmark: full-queue scenario (producer fast, consumer slow) → compare latency

## Cross-References

- OS layer: `high-concurrency/01-os-sync-primitives`
- Lock-free core from: `high-concurrency/03-spsc-ring-buffer` through `06-mpmc-ring-buffer`
- CV comparison: `low-concurrency/06-condition-variable`, `low-concurrency/11-producer-consumer-basic`
- Used in: `high-concurrency/08-thread-pool` (worker blocking on task queue)

## Pitfalls

- ⚠️ `notify_one` may wake a thread that can't make progress (slot already taken by another) → always loop and re-check
- ⚠️ `atomic::wait` may spuriously wake → always check predicate in a loop
- ⚠️ Notifying after push/pop is critical — forgetting it causes permanent blocking
- ⚠️ Notify inside vs outside the "locked" code: `atomic::notify` doesn't have the same trade-off as CV notify_under_lock vs notify_after_unlock — it's always safe to notify after the store
- ⚠️ SPIN_LIMIT too high → burning CPU; too low → unnecessary syscalls
