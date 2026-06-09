# Spec: 08-memory-ordering

## Metadata

- **File**: `low-concurrency/08-memory-ordering.md`
- **Phase**: II — Concurrency Primitives
- **Prerequisites**: 07-atomic-fundamentals, no-concurrency/05-happens-before
- **Estimated sections**: 8

## Learning Objectives

- Know all six `std::memory_order` values and their semantics
- Map each ordering to a concrete guarantee about visibility and reordering
- Choose the **weakest** ordering that satisfies correctness — performance depends on it
- Understand `std::atomic_thread_fence` as distinct from operation-level ordering
- Reason about inter-thread happens-before using acquire/release pairs

## Outline

### 1. The Six Memory Orders

| Order | Meaning | Cost (x86) | Cost (ARM) |
|-------|---------|------------|------------|
| `relaxed` | No ordering guarantees | Cheapest | Cheapest |
| `consume` | Data dependency ordering (broken, avoid) | Like relaxed | Like relaxed |
| `acquire` | No reads/writes after can be reordered before | Free (HW does this) | Light |
| `release` | No reads/writes before can be reordered after | Some barrier | Medium |
| `acq_rel` | Both acquire and release | One mfence | DMB |
| `seq_cst` | Total global order (default) | mfence | DMB + stronger |

### 2. relaxed — Atomicity Only, No Ordering

- Guarantees: atomicity and modification-order coherence per variable
- Does NOT guarantee: visibility of other variables across threads
- Use case: simple counters where you only care about the final count
- Example: per-thread statistics counter, aggregating at the end

### 3. acquire / release — Pair for Inter-Thread Ordering

- **release** store + **acquire** load = synchronizes-with edge
- The classic pattern: thread 1 writes data (non-atomic), does release store; thread 2 does acquire load, reads data
- All writes before release are visible after acquire
- Analogy: release = "my work is done"; acquire = "show me what's done"
- Use case: mutex implementation, producer-consumer flag

### 4. seq_cst — The Default, The Strongest

- Total order: all seq_cst operations across all threads appear in a single global order
- Easier to reason about — when in doubt, use this
- But more expensive on ARM (requires DMB + additional fencing)
- Use case: when multiple atomics must be globally ordered relative to each other

### 5. consume — Avoid It

- Intended to be a cheaper acquire that only orders data-dependent loads
- Compilers treat it as acquire (pessimistic but correct)
- Standard committee encourages avoiding it — may be removed or redefined
- tl;dr: use acquire instead

### 6. acq_rel — For Read-Modify-Write Operations

- `exchange`, `compare_exchange_xxx` that need both acquire and release semantics
- Acquire the old value (as acquire), release the new value (as release)
- Use case: the head of a lock-free queue (read old head, write new head)

### 7. Fences (std::atomic_thread_fence)

- A standalone barrier, not tied to a specific atomic operation
- `std::atomic_thread_fence(acq)` acts like a load-acquire of all preceding atomics
- `std::atomic_thread_fence(rel)` acts like a store-release of all subsequent atomics
- Used in more complex lock-free patterns (e.g., Dekker-like algorithms)

### 8. Choosing the Right Order

- Decision flowchart: "Do I need inter-thread ordering?" → No → relaxed; Yes → "Do I need a global total order?" → No → acquire/release; Yes → seq_cst
- Default to seq_cst → profile → weaken to acquire/release → benchmark → iterate
- Don't prematurely optimize to relaxed — correctness first

## Key Concepts

| Concept | Depth |
|---------|-------|
| All 6 memory_order values | Full |
| acquire/release pairing | Full |
| seq_cst total order | Full |
| Fence vs operation ordering | Medium |
| x86 vs ARM cost model | Light |

## Code Examples

- relaxed counter: `fetch_add(1, relaxed)` — wrong if used for synchronization, right for stats
- acquire/release flag: thread 1 writes data, then `flag.store(true, release)`; thread 2 `flag.load(acquire)`, then reads data
- Broken: relaxed flag → data may not be visible
- Fence example: Peterson-like mutual exclusion
- Benchmark: throughput for each order on x86 and ARM

## Pitfalls

- ⚠️ Defaulting to relaxed because "it's faster" — without proving correctness
- ⚠️ Using consume — it compiles, but gives no ordering guarantee in practice
- ⚠️ Forgetting that relaxed only orders the same atomic variable — all other variables are unordered
- ⚠️ Thinking `seq_cst` means "all threads see everything at the same instant" — it doesn't
