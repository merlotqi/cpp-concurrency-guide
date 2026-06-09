# Spec: 01-os-sync-primitives

## Metadata

- **File**: `high-concurrency/01-os-sync-primitives.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: low-concurrency/02-mutex-basics, low-concurrency/06-condition-variable
- **Estimated sections**: 7

## Learning Objectives

- Understand how `std::mutex` and `std::condition_variable` work under the hood
- Use the **Linux futex** syscall directly: `FUTEX_WAIT`, `FUTEX_WAKE`, `FUTEX_WAIT_BITSET`
- Use the **Windows WaitOnAddress** family: `WaitOnAddress`, `WakeByAddressSingle`, `WakeByAddressAll`
- Use **C++20 portable** `std::atomic<T>::wait` / `notify_one` / `notify_all`
- Know the macOS/BSD equivalent (`__ulock_wait` / `__ulock_wake`) for completeness
- Build a working mutex from futex to internalize the mechanism

## Outline

### 1. Why OS Primitives Matter

- `std::mutex` is not magic — it's built on OS futex (or equivalent)
- When you understand futex, you understand:
  - Why uncontended mutex is fast (no syscall — just atomic CAS in userspace)
  - Why contended mutex is slow (syscall into kernel to block)
  - Why condition variables require a mutex (the futex word must be atomically checked-and-waited)
- This knowledge is essential for building your own synchronization primitives

### 2. The Core Idea: Atomic + OS Wait = Efficient Blocking

- Fast path (no contention): pure atomic CAS in userspace → ~5-10 ns
- Slow path (contention): syscall to kernel to sleep → ~1-10 µs
- The kernel only gets involved when threads actually need to block
- This is why mutex can be both "fast when uncontended" and "block when contended"

### 3. Linux: futex(2)

- `syscall(SYS_futex, addr, op, val, timeout, addr2, val3)`
- Key operations:
  - `FUTEX_WAIT`: if `*addr == val`, sleep until `FUTEX_WAKE`
  - `FUTEX_WAKE`: wake up to `val` waiters
  - `FUTEX_WAIT_BITSET`: like WAIT but with a bitmask — enables per-CV futex words
- The atomic check-then-sleep is **atomic** from the caller's perspective (the kernel checks `*addr` while holding internal lock)
- `FUTEX_PRIVATE_FLAG`: for single-process futexes (faster, no VMA lookup)
- `FUTEX_WAKE_OP`: combined wake + atomic operation (used for CV broadcast optimization)
- `futex(7)` man page as reference

### 4. Windows: WaitOnAddress

- `WaitOnAddress(Address, CompareAddress, Size, Milliseconds)`
- `WakeByAddressSingle(Address)` — wake one waiter
- `WakeByAddressAll(Address)` — wake all waiters
- Same atomic-check-then-sleep semantics: the kernel verifies `*Address == *CompareAddress` atomically
- Simpler API than futex — fewer operations, no `BITSET` equivalent
- Available since Windows 8 / Server 2012

### 5. C++20: std::atomic<T>::wait / notify

- `atomic.wait(old_value, order)`: if value is still `old_value`, block
- `atomic.notify_one()`: wake one waiter
- `atomic.notify_all()`: wake all waiters
- Platform-agnostic — the compiler/library maps to futex/WaitOnAddress/__ulock_wait
- Constraints: only works with `atomic<T>` where T is integer or pointer (implementation may support more)
- This is how you write portable blocking without pthread or Windows API

### 6. macOS / BSD: __ulock_wait / __ulock_wake

- macOS: `__ulock_wait(operation, addr, value, timeout)` / `__ulock_wake(operation, addr, value)`
- FreeBSD: `_umtx_op(UMTX_OP_WAIT, ...)` / `_umtx_op(UMTX_OP_WAKE, ...)`
- Brief overview — C++20 `atomic::wait` wraps these for you
- Mention for completeness; detailed exploration not needed if readers use C++20

### 7. Hands-On: Build a Mutex from futex

```cpp
class FutexMutex {
    std::atomic<int> state{0}; // 0 = unlocked, 1 = locked, no waiters
                                //                 2 = locked, with waiters
    void lock() {
        // Fast path: try to CAS from 0 to 1
        int expected = 0;
        if (state.compare_exchange_strong(expected, 1, acquire)) return;
        // Slow path: CAS to 2 and futex_wait
        if (expected != 2) expected = state.exchange(2, acq_rel);
        while (expected != 0) {
            state.wait(2, relaxed); // C++20 style (or futex(FUTEX_WAIT) directly)
            expected = state.exchange(2, acq_rel);
        }
    }
    void unlock() {
        if (state.fetch_sub(1, release) != 1) {
            state.store(0, relaxed);
            state.notify_one(); // wake one waiter
        }
    }
};
```

- Benchmark: FutexMutex vs `std::mutex` — should be comparable
- Show both the raw `syscall(SYS_futex, ...)` version and the C++20 `atomic::wait` version

## Key Concepts

| Concept | Depth |
|---------|-------|
| futex(FUTEX_WAIT/WAKE) | Full |
| WaitOnAddress/WakeByAddress | Full |
| C++20 atomic::wait/notify | Full |
| Check-then-sleep atomicity | Full |
| Userspace fast-path pattern | Full |
| macOS __ulock_wait | Light |

## Code Examples

- `futex_wait(address, expected)` / `futex_wake(address, count)` wrapper functions
- Full `FutexMutex` implementation with C++20 `atomic::wait`
- Full `FutexMutex` implementation with raw `syscall(SYS_futex, ...)`
- Benchmark: 1, 2, 4, 8 threads contending on FutexMutex vs std::mutex
- Profiling: `perf stat` showing futex syscall count under contention

## Cross-References

- Used in: `high-concurrency/07-blocking-lockfree-queue` (to add blocking to lock-free queues)
- Used in: `high-concurrency/08-thread-pool` (worker thread blocking on task queue)
- Foundation for understanding: `low-concurrency/02-mutex-basics`, `low-concurrency/06-condition-variable`

## Pitfalls

- ⚠️ Futex WAIT checks `*addr == val` — not `*addr <= val` or any other comparison
- ⚠️ Futex word must be 32-bit on most platforms (64-bit futex is Linux 5.0+ with `FUTEX2`)
- ⚠️ `WaitOnAddress` may spuriously wake (like CV spurious wakeup) — always check condition in a loop
- ⚠️ `atomic::wait` is not available for all atomic types — check `is_always_lock_free`
