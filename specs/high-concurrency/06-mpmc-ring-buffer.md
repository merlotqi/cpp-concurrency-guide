# Spec: 06-mpmc-ring-buffer

## Metadata

- **File**: `high-concurrency/06-mpmc-ring-buffer.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 04-spmc-queue, 05-mpsc-queue
- **Estimated sections**: 8

## Learning Objectives

- Implement a **fully general MPMC ring buffer** with CAS on both read and write indices
- Understand the **ticket-based** approach: separate "claim" from "commit" to reduce contention
- Compare three MPMC algorithms: single-CAS, ticket-based, and turn-based (Folly style)
- Learn the memory reclamation strategy — when and how to safely overwrite slots
- Benchmark against mutex+queue and all simpler lock-free variants

## Outline

### 1. The Full Contention Problem

- Multiple producers CAS `write_pos_`, multiple consumers CAS `read_pos_`
- Both ends contend → this is the hardest lock-free queue to get right and fast
- The naive approach: one CAS per push/pop, like SPMC producer-side + MPSC consumer-side
- Problem: the CAS on the shared index becomes a serial bottleneck at high thread counts

### 2. Naive MPMC Implementation

- Producer: CAS `write_pos_` to claim slot (same as MPSC producer)
- Consumer: CAS `read_pos_` to claim slot (same as SPMC consumer)
- Works correctly, but CAS on `write_pos_` and `read_pos_` are contented hot spots
- Throughput drops sharply above 4 producers or 4 consumers
- This is the "simple but slow" baseline

### 3. Ticket-Based MPMC (Improved)

- Separate **claiming** a position from **committing** the data:
  - Producer 1: CAS `write_claim_` → gets ticket T
  - Producer 1: writes data to `buffer_[T % Capacity]`
  - Producer 1: spins until `write_commit_ == T` (previous producer committed)
  - Producer 1: `write_commit_.store(T + 1, release)` → commits
- The ticket system serializes commits → guarantees order → consumers see data in order
- Benefit: CAS only on `write_claim_` (fast ticket allocation), then a wait-free commit step
- Same pattern on consumer side

### 4. Turn-Based MPMC (Folly MPMCQueue Style)

- Pre-allocated array of **turn counters**, one per slot:
  ```
  struct Slot { T data; std::atomic<size_t> turn; };
  Slot buffer_[Capacity];
  // turn[i] == i → slot is ready for producer (empty)
  // turn[i] == i + 1 → slot is ready for consumer (full)
  ```
- Producer: `while (buffer_[pos].turn.load(acquire) != pos); buffer_[pos].data = item; buffer_[pos].turn.store(pos + 1, release);`
- Consumer: `while (buffer_[pos].turn.load(acquire) != pos + 1); item = buffer_[pos].data; buffer_[pos].turn.store(pos, release);`
- No shared CAS on write_pos/read_pos → each slot has its own turn atomic
- Congestion is per-slot, not per-queue → scales much better
- This is the algorithm used by Facebook Folly's `MPMCQueue`

### 5. Memory Reclamation in MPMC

- In ring buffers, "reclamation" means detecting when a slot is safe to overwrite
- With turn-based: the turn counter acts as a per-slot ready flag → no separate reclamation
- With ticket-based: write_pos and read_pos implicitly define safe slots → `pos < read_pos + Capacity`
- No dynamic allocation in ring buffer designs → no ABA, no hazard pointers needed
- This is a key advantage of bounded (ring buffer) queues for lock-free programming

### 6. Algorithm Comparison

| Algorithm | 4P4C throughput | Main cost | Complexity |
|-----------|----------------|-----------|------------|
| Mutex+queue | ~3M ops/s | Mutex serialization | Trivial |
| Naive MPMC | ~15M ops/s | CAS contention on indices | Low |
| Ticket MPMC | ~30M ops/s | CAS on claim + spin on commit | Medium |
| Turn-based (Folly) | ~50M ops/s | Per-slot atomic load | Medium |

### 7. Cache Line Layout for MPMC

- Write claim/commit: separate cache lines
- Read claim/commit: separate cache lines
- Slot array: each slot padded to cache line? Expensive. At minimum, separate slot array from index variables
- Turn-based naturally distributes contention: each slot's turn is independent → cache lines naturally partition
- Hardware cache-coherence protocol (MESI) handles this efficiently as long as different threads access different slots

### 8. Benchmark Suite

- Vary producer count: 1, 2, 4, 8, 16
- Vary consumer count: 1, 2, 4, 8, 16
- Vary queue capacity: 64, 256, 1024, 4096
- Vary item size: 8B (int), 64B (cache line), 256B, 1KB
- Metrics: throughput, latency percentiles, CPU utilization, cache misses (`perf stat`)
- Comparison table with all previous queue implementations

## Key Concepts

| Concept | Depth |
|---------|-------|
| Ticket-based MPMC | Full |
| Turn-based MPMC (Folly) | Full |
| Naive MPMC (baseline) | Full |
| Slot-level vs queue-level contention | Full |
| Per-slot turn counters | Full |
| Algorithm comparison matrix | Full |

## Code Examples

- Naive MPMC: CAS on write_pos + CAS on read_pos (~60 lines)
- Ticket MPMC: claim + commit pattern (~90 lines)
- Turn-based MPMC: Folly-style per-slot turns (~100 lines)
- Benchmark harness: parameterized producer/consumer/capacity
- Benchmark CSV output → matplotlib/pgfplots visualization

## Cross-References

- Combines: `high-concurrency/04-spmc-queue` + `high-concurrency/05-mpsc-queue`
- Foundation: `high-concurrency/02-lockfree-design-space`
- Reference analysis: `high-concurrency/11-real-world-cases` (Folly MPMCQueue deep-dive)
- Comparison baseline: `low-concurrency/11-producer-consumer-basic`

## Pitfalls

- ⚠️ Ticket MPMC: commit spin-loop → if a producer is preempted after claiming but before committing, all producers stall → not truly lock-free (courtesy of ticket order)
- ⚠️ Turn-based: slot count must equal capacity, and capacity must be power of 2 for `pos % Capacity` optimization
- ⚠️ Turn overflow: `turn` starts at 0 and increments by 2 per full cycle (empty→full→empty). `uint64_t` → practical infinity. But verify the wrap condition mathematically.
- ⚠️ Naive MPMC with single read_pos/write_pos: consumer CAS on `read_pos_` means consumers fight for the *same cache line* → false sharing between consumers. Consider padding between index variables.
