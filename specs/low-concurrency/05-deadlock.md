# Spec: 05-deadlock

## Metadata

- **File**: `low-concurrency/05-deadlock.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 03-lock-management
- **Estimated sections**: 6

## Learning Objectives

- Recognize the **four necessary conditions** for deadlock
- Use `std::lock` / `std::scoped_lock` to avoid deadlock when locking multiple mutexes
- Apply **hierarchical locking** as a design pattern
- Understand **livelock** and **starvation** as related problems
- Use runtime tools to detect deadlocks

## Outline

### 1. What Is Deadlock?

- Two (or more) threads each holding a resource the other needs
- The classic Dining Philosophers problem
- Simple C++ example: thread A locks mtx1 then mtx2; thread B locks mtx2 then mtx1
- Deadlock is deterministic given the right interleaving — not a "rare bug"

### 2. The Four Necessary Conditions

- **Mutual exclusion**: resources cannot be shared
- **Hold and wait**: thread holds one resource while waiting for another
- **No preemption**: resources cannot be forcibly taken away
- **Circular wait**: cycle in the wait-for graph
- Break any one → deadlock impossible

### 3. Avoidance Strategy 1: Consistent Lock Ordering

- All threads acquire mutexes in the same global order
- Practical: assign each mutex an ID, always lock lower ID first
- Limitation: doesn't compose — two independently correct modules can deadlock when combined

### 4. Avoidance Strategy 2: std::lock and std::scoped_lock

- `std::lock(m1, m2, m3)` atomically locks all or none — deadlock-free algorithm
- Internally uses try_lock with back-off
- `std::scoped_lock` is the RAII wrapper that calls `std::lock`
- This is the **recommended approach** in modern C++

### 5. Avoidance Strategy 3: Hierarchical Locking

- Assign numeric levels to mutexes
- A thread may only lock mutexes with levels **higher** (or lower, by convention) than currently held
- Runtime assertion: `assert(level > current_level)` → catches violation immediately
- The `hierarchical_mutex` class (from C++ Concurrency in Action)
- Catches ordering bugs at development time

### 6. Related Problems

- **Livelock**: threads keep responding to each other but make no progress (like two people in a corridor)
- **Starvation**: a thread never gets the resource because others keep taking it
- Lock-free algorithms (Phase III) are one solution, but bring their own complexity

## Key Concepts

| Concept | Depth |
|---------|-------|
| Four deadlock conditions | Full |
| std::lock / scoped_lock | Full |
| Consistent lock ordering | Full |
| Hierarchical mutex | Full |
| Livelock / starvation | Medium |

## Code Examples

- Classic deadlock: two mutexes, two threads, opposite order
- Fixed with `std::scoped_lock`
- Hierarchical mutex implementation (~40 lines) + demo
- Livelock simulation (two threads constantly yielding)

## Pitfalls

- ⚠️ Nested locks are the #1 cause — always question "do I need to hold lock A while acquiring lock B?"
- ⚠️ Callbacks / virtual functions while holding a lock — unknown code may lock
- ⚠️ `std::lock` doesn't compose across modules — consistent ordering at application level still needed
