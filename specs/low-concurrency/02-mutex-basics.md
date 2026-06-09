# Spec: 02-mutex-basics

## Metadata

- **File**: `low-concurrency/02-mutex-basics.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 01-thread-management, no-concurrency/05-happens-before
- **Estimated sections**: 5

## Learning Objectives

- Use `std::mutex` to protect critical sections
- Understand the happens-before relationship established by lock/unlock
- Know the variants: `std::recursive_mutex`, `std::timed_mutex`, `std::recursive_timed_mutex`
- Choose appropriate lock granularity — not too coarse, not too fine

## Outline

### 1. std::mutex — The Basic Lock

- `lock()`: blocks until exclusive ownership acquired
- `unlock()`: releases ownership — only the owning thread may call this
- `try_lock()`: non-blocking attempt, returns `bool`
- The mutex is the simplest happens-before edge: unlock → subsequent lock

### 2. Critical Sections and Invariants

- Identify the invariant: what must be true between lock and unlock
- Keep critical sections **as short as possible** — lock, do the work, unlock
- Never do IO or call unknown code while holding a lock (inversion, deadlock risk)
- The mutex should protect **data**, not **code**

### 3. Mutex Variants

- `std::recursive_mutex`: same thread can lock multiple times — use sparingly (hides design problems)
- `std::timed_mutex`: `try_lock_for(duration)`, `try_lock_until(time_point)` — for timeout-based designs
- `std::recursive_timed_mutex`: both recursive and timed
- When to use each: mutex → default; timed → when you need timeouts; recursive → almost never

### 4. Lock Granularity

- **Coarse-grained**: one mutex for many data items → more contention, simpler reasoning
- **Fine-grained**: one mutex per data item → less contention, deadlock risk from multiple locks
- Case study: thread-safe linked list — one global mutex vs one per node
- The trade-off: correctness simplicity vs performance

### 5. Mutex Internals (Preview)

- Brief peek under the hood: `std::mutex` wraps `pthread_mutex_t` (Linux) or `SRWLOCK` (Windows)
- On Linux with no contention: fast-path `futex` in userspace (faster than syscall)
- On contention: falls back to `futex(FUTEX_WAIT)` → kernel schedules the thread
- Full deep-dive deferred to `high-concurrency/01-os-sync-primitives`

## Key Concepts

| Concept | Depth |
|---------|-------|
| std::mutex lock/unlock | Full |
| Critical section design | Full |
| Mutex variants | Medium |
| Lock granularity | Full |
| Internal mechanism | Light |

## Code Examples

- Simple counter protected by `std::mutex`
- `try_lock` pattern: if locked, do something else (no blocking)
- Fine-grained locking: two independent counters with two mutexes
- Broken: calling unknown code inside critical section → deadlock

## Pitfalls

- ⚠️ Forgetting to unlock (use RAII — next chapter)
- ⚠️ Locking mutex that's already locked by current thread → UB (use recursive if you must)
- ⚠️ Holding lock across IO or blocking call
- ⚠️ Double-unlock → UB
