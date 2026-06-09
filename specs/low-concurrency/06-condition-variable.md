# Spec: 06-condition-variable

## Metadata

- **File**: `low-concurrency/06-condition-variable.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 03-lock-management
- **Estimated sections**: 7

## Learning Objectives

- Use `std::condition_variable` for thread-to-thread signaling
- Understand **spurious wakeup** and write the correct wait loop
- Distinguish `notify_one` from `notify_all` — and when each is correct
- Recognize the **lost wakeup** problem and how the mutex+predicate pattern prevents it
- Apply the standard wait paradigm: `cv.wait(lock, predicate)`

## Outline

### 1. Why Condition Variables?

- Mutex alone can't express "wait until X is true" — you'd have to spin
- Condition variable: sleep until notified + atomically release mutex + reacquire on wake
- The two roles: **waiting thread** and **notifying thread**
- Analogy: a pager/beeper — you sleep until someone pages you

### 2. The Basic API

- `wait(lock)`: atomically unlocks mutex and blocks; reacquires on wake
- `wait(lock, predicate)`: the recommended form — loops internally
- `notify_one()`: wakes one waiting thread (unspecified which)
- `notify_all()`: wakes all waiting threads
- `wait_for(lock, duration, predicate)` / `wait_until(lock, time_point, predicate)`: timed waits

### 3. Spurious Wakeup

- The thread may wake up even though **no thread called notify**
- Hardware/OS scheduling artifacts
- The fix: always re-check the condition; `wait(lock, predicate)` handles this automatically
- This is NOT a bug — it's part of the POSIX/C++ contract

### 4. The Correct Wait Pattern

```
std::unique_lock lock(mtx);
cv.wait(lock, [] { return ready; });
// At this point: lock is held AND ready == true
```

- Why we need `unique_lock` not `lock_guard`: `wait` must unlock/re-lock
- The predicate is checked under the lock — no race between check and wait
- This pattern prevents **lost wakeup**: if notify happens before wait, predicate is already true → wait returns immediately

### 5. notify_one vs notify_all

- `notify_one`: wake one waiter — correct when all waiters are equivalent (e.g., thread pool)
- `notify_all`: wake all waiters — correct when different waiters wait for different conditions
- The "thundering herd" problem with notify_all: N threads wake, N-1 immediately go back to sleep
- Rule: if only one thread can make progress, use `notify_one`

### 6. Condition Variable + Shared Data Pattern

- The data being waited on is a separate variable (protected by the same mutex)
- CV doesn't carry data — it only signals
- The full pattern: modify data under lock → unlock → notify (or notify under lock, both valid)
- Notify under lock: no chance of "late notification" but potentially slower (waiter wakes immediately, finds lock held, blocks)
- Notify after unlock: better performance (waiter can grab lock directly)

### 7. std::condition_variable_any

- Works with any lock type, not just `unique_lock<mutex>`
- Slightly more overhead than `std::condition_variable`
- Use when you need `shared_lock` or custom lock types

## Key Concepts

| Concept | Depth |
|---------|-------|
| CV wait/notify mechanics | Full |
| Spurious wakeup | Full |
| Lost wakeup | Full |
| notify_one vs notify_all | Full |
| CV + predicate pattern | Full |

## Code Examples

- Simple flag-based signaling: worker thread waits for `ready` flag
- Bounded queue with CV: `push` notifies consumer, `pop` notifies producer (preview of producer-consumer)
- Timed wait: `wait_for` with timeout — "process or give up after 1 second"
- Broken: `if (predicate)` instead of `while (predicate)` → spurious wakeup bug
- Broken: waiting without mutex → lost wakeup

## Pitfalls

- ⚠️ Using `if` instead of `while` for the condition check → spurious wakeup bug
- ⚠️ Notifying without holding the mutex that protects the condition → lost wakeup
- ⚠️ Forgetting to re-check the predicate after wake → acting on stale state
- ⚠️ `notify_one` when multiple waiters wait for different predicates → some threads never wake
