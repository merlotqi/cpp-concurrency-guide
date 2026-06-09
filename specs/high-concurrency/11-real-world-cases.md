# Spec: 11-real-world-cases

## Metadata

- **File**: `high-concurrency/11-real-world-cases.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: All previous high-concurrency chapters
- **Estimated sections**: 5

## Learning Objectives

- Read and understand production-grade lock-free code
- Compare design decisions across three major implementations: DPDK rte_ring, LMAX Disruptor, Facebook Folly MPMCQueue
- Extract reusable patterns from industrial codebases
- Know how to evaluate and adapt a real-world concurrent component for your own needs

## Outline

### 1. DPDK rte_ring

**Context**: Data Plane Development Kit — high-performance packet processing, millions of packets per second.

- **Design**: Bounded MP/MC ring buffer (supports SP/SC/MP/MC in a single implementation)
- **Key innovation 1**: CAS-free for SPSC, SCMP, MPSC (uses load/store only for single-producer or single-consumer modes)
- **Key innovation 2**: `rte_smp_wmb()` memory barriers instead of C11 atomics (DPDK predates widespread C11 atomic adoption)
- **Key innovation 3**: Watermark — enqueue/dequeue can specify "all-or-nothing" semantics (useful for burst packet processing)
- **Header structure**: `struct rte_ring` has producer/consumer `head` and `tail` indices in separate cache lines
- **Burst API**: `rte_ring_enqueue_burst(ring, objs, n)` — enqueue up to N objects in one call (batching!)
- **Code walkthrough**: `__rte_ring_do_enqueue` — the CAS loop for multi-producer mode
- **Why it works**: DPDK runs in busy-poll mode (no blocking), so pure lock-free spin is correct and optimal

### 2. LMAX Disruptor

**Context**: Financial trading exchange — ultra-low-latency (< 1µs end-to-end), millions of events per second.

- **Design**: Ring buffer + sequence numbers + event processors
- **Key innovation 1**: Pre-allocate all event slots — zero GC/allocation in hot path (designed for Java; C++ benefits from same principle)
- **Key innovation 2**: Sequence barriers — each consumer tracks its own sequence number; producer checks all consumer sequences before overwriting
- **Key innovation 3**: No contention between consumers — each consumer reads its own slots; no CAS on consumer side
- **Key innovation 4**: Multi-cast/broadcast — multiple consumers can read the same events (unlike queues where consumers compete)
- **Sequence barrier**: Producer → Barrier → Consumer. The barrier aggregates consumer progress; producer blocks if any consumer is too slow.
- **C++ adaptation**: use `std::atomic<int64_t>` for sequence numbers, `wait` for backpressure, pre-allocated `std::array<Event, Size>`
- **Why it works**: No write contention (SPMC-like), no read contention (each consumer independent), no allocation (pre-allocated)

### 3. Facebook Folly MPMCQueue

**Context**: General-purpose C++ library — needs to work well across diverse workloads.

- **Design**: Turn-based bounded MPMC queue (the algorithm from Chapter 06)
- **Key innovation 1**: Per-slot turn counters — no shared CAS on write/read indices → contention is per-slot, not per-queue
- **Key innovation 2**: Ticket-based turn assignment — `writeTicket` increments by 1 per slot → wait for `turn == writeTicket`
- **Key innovation 3**: `is_lock_free()` checks — the queue adapts behavior based on whether `std::atomic<T>` is lock-free
- **Key innovation 4**: Supports move-only types and non-trivial destructors
- **Code walkthrough**: `MPMCQueue<T>::write(T&&)` — claim ticket, wait for turn, write data, advance turn
- **Why it works**: Turn-based algorithm distributes contention to per-slot granularity → minimal cache-line ping-pong

### 4. Design Comparison Matrix

| Aspect | DPDK rte_ring | LMAX Disruptor | Folly MPMCQueue |
|--------|--------------|----------------|-----------------|
| **Language** | C + macros | Java (design), C++ (adaptation) | C++17 |
| **Bounded/Unbounded** | Bounded | Bounded | Bounded |
| **Producer model** | SP/MP | SP (usually) | MP |
| **Consumer model** | SC/MC | MC (independent) | MC |
| **Contention model** | CAS on head/tail | Per-consumer sequence CAS | Per-slot turn counter |
| **Blocking** | No (busy-poll) | Yes (backpressure) | No (returns false) |
| **Batching** | Burst API (1-32) | Batch processing | Not built-in |
| **Memory model** | Custom barriers | Java/C++ atomics | C++11 atomics |
| **Cache line awareness** | Manual padding | Explicit padding | `hardware_destructive_interference_size` |
| **Best for** | Packet processing, HFT | Event processing, streaming | General-purpose, library use |

### 5. Lessons and Patterns

- **Batching is a universal win**: all three designs batch in some way (burst enqueue, batch processing, ticket claim)
- **Separate hot from cold data**: indices/sequence numbers on separate cache lines; data in contiguous array
- **Piggyback information on existing atomics**: DPDK uses `head`/`tail` for both position and synchronization; Folly's `turn` serves double duty (slot state + memory ordering)
- **Design for the common case**: SPSC code path in DPDK is free of CAS; Disruptor consumers never contend; Folly's MPMC turns typically advance without waiting
- **Know your workload**: DPDK expects always-busy (no blocking); Disruptor expects bursty (backpressure needed); Folly is general (no blocking, try-and-fail interface)
- **C++ pattern**: Use `std::atomic<T>` + `alignas` + `static_assert` for portable, correct lock-free code
- **The non-blocking spectrum**: DPDK (never blocks) → Folly (never blocks, caller decides) → Disruptor (blocks on backpressure)
- **Open question for the reader**: How would you combine Disruptor's broadcast semantics with Folly's turn-based MPMC? (Hint: each consumer gets its own turn, producer publishes N turns ahead)

## Key Concepts

| Concept | Depth |
|---------|-------|
| DPDK rte_ring design | Deep |
| LMAX Disruptor pattern | Deep |
| Folly MPMCQueue design | Deep |
| Three-way comparison | Full |
| Reusable patterns | Full |

## Code Examples

- Annotated `rte_ring` enqueue walkthrough (extracted core logic in C++)
- Simplified Disruptor in C++ (~200 lines)
- Folly `MPMCQueue<T>` usage examples + internal walkthrough
- Comparison benchmark: all three (simplified) on 4P4C 1M ops

## Cross-References

- SPSC → MPMC implementations: `high-concurrency/03` through `06`
- OS wait (for backpressure): `high-concurrency/01-os-sync-primitives`
- Memory ordering: `low-concurrency/08-memory-ordering`
- Cache line / false sharing: `high-concurrency/02-lockfree-design-space`
- Performance tuning: `high-concurrency/10-performance-tuning`

## Pitfalls

- ⚠️ Copying DPDK's custom barrier approach → use `std::atomic` with standard memory_orders instead; it's portable and better understood by compilers
- ⚠️ Disruptor's ring buffer MUST be power-of-2 → modulo by bitmask; non-power-of-2 kills latency
- ⚠️ Folly MPMCQueue assumes `atomic<T>::is_lock_free()` → if false, it falls back to mutex; instantiating with large structs silently degrades
- ⚠️ All three implementations use `uint64_t` for indices to avoid practical wrap-around issues; don't use `uint32_t`
