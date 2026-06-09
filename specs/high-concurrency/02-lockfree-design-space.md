# Spec: 02-lockfree-design-space

## Metadata

- **File**: `high-concurrency/02-lockfree-design-space.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: low-concurrency/07-atomic-fundamentals, low-concurrency/08-memory-ordering
- **Estimated sections**: 7

## Learning Objectives

- Define lock-free, wait-free, and obstruction-free precisely
- Understand the ABA problem and the standard countermeasures
- Know the memory reclamation problem and the major solutions: hazard pointers, epoch-based, RCU
- Apply cache-line awareness (false sharing, padding) to concurrent data structures
- Map the design space: bounded vs unbounded, intrusive vs non-intrusive

## Outline

### 1. The Progress Guarantee Hierarchy

| Guarantee | Definition | Example |
|-----------|-----------|---------|
| **Wait-free** | Every thread completes in bounded steps | RCU read, atomic increment |
| **Lock-free** | At least one thread makes progress in bounded steps | CAS-loop queue |
| **Obstruction-free** | A thread completes if it runs in isolation | Double-ended queue with helping |
| **Blocking** | Threads may sleep waiting for others | mutex-based queue |

- Wait-free ⊂ Lock-free ⊂ Obstruction-free
- Lock-free is the practical sweet spot: no thread can block all others
- But lock-free ≠ fast — a poorly designed lock-free queue can be slower than mutex

### 2. The ABA Problem

- Thread 1 reads A; thread 2 changes A→B→A; thread 1's CAS succeeds when it shouldn't
- Classic example: lock-free stack pop
- Solutions:
  - **Tagged pointers**: pack a counter into the upper bits of the pointer → `A#1 → B#2 → A#3` (different from `A#1`)
  - **Double-word CAS** (`cmpxchg16b` on x86): atomically CAS both pointer and counter
  - **Hazard pointers**: prevent A from being freed while a thread holds a reference
  - **RCU**: defer reclamation until no thread can hold a reference
- C++: `std::atomic<std::uintptr_t>` + manual bit packing, or `__int128` CAS

### 3. Memory Reclamation Problem

- In lock-free structures, a node might be "unlinked" but still referenced by another thread
- Freeing it immediately → use-after-free
- Solutions:
  - **Quiescent-state-based reclamation (QSBR)**: defer free until all threads have passed a quiescent point. Fastest, but threads must periodically announce quiescence.
  - **Epoch-based reclamation**: threads are in "epochs"; defer free until all active threads are in a later epoch. Good balance.
  - **Hazard pointers**: each thread publishes a list of pointers it's currently accessing. Before freeing, check if any thread holds it. Wait-free for readers.
  - **Reference counting** (`std::atomic<std::shared_ptr>` C++20): the simplest approach, but `shared_ptr` adds overhead.
- Table: which method fits which scenario

### 4. Cache Lines and False Sharing

- Cache line: 64 bytes on x86/ARM
- **False sharing**: two unrelated variables on the same cache line → one thread's write invalidates another thread's cache line → constant cache misses
- The fix: pad with `alignas(64)` or `alignas(std::hardware_destructive_interference_size)`
- C++17: `std::hardware_destructive_interference_size` — the minimum offset to avoid false sharing
- **True sharing**: legitimately contended data — the real bottleneck. Reduce by: batching, per-thread accumulation, or spreading hot data across cache lines
- Visual: two threads writing to adjacent `int` fields on the same cache line vs padded

### 5. Bounded vs Unbounded Queues

| Criterion | Bounded (Ring Buffer) | Unbounded (Linked List) |
|-----------|----------------------|------------------------|
| Memory | Fixed pre-allocation | Dynamic allocation |
| Contention | Array index CAS | Node pointer CAS + allocation |
| Backpressure | Naturally enforces | Needs external mechanism |
| Cache locality | Excellent (contiguous) | Poor (pointer chasing) |
| Real-world use | Audio/video pipelines, DPDK | Task queues, work stealing |

### 6. Intrusive vs Non-Intrusive

- **Non-intrusive**: the queue owns the nodes and copies data in/out. General, flexible, but involves copying.
- **Intrusive**: the data item embeds the queue node (e.g., `struct Item { Item* next; int data; };`). No allocation, no copying, but requires modifying the data type.
- Linux kernel: mostly intrusive (`list_head`, `hlist_node`)
- DPDK `rte_ring`: non-intrusive (object array)
- When to use each: performance-critical path + you control the type → intrusive. Otherwise → non-intrusive.

### 7. Decision Framework

- Flowchart: "Do I need blocking?" → "Bounded or unbounded?" → "All producers/consumers equal?" (SPSC vs MPMC) → "Memory reclamation strategy?"
- This chapter is the map — chapters 3-7 are the territory

## Key Concepts

| Concept | Depth |
|---------|-------|
| Lock-free / wait-free definitions | Full |
| ABA problem | Full |
| Hazard pointers / Epoch / RCU | Medium |
| False sharing + padding | Full |
| Bounded vs unbounded | Full |
| Intrusive vs non-intrusive | Medium |

## Code Examples

- ABA demo: lock-free stack that loses an element
- ABA fix: tagged pointer CAS
- False sharing demo: two counters, adjacent vs padded, 4 threads → 4x throughput difference
- `std::hardware_destructive_interference_size` usage

## Cross-References

- Used by: every subsequent chapter (03-06) as the theoretical foundation
- Prerequisite concepts from: `low-concurrency/07-atomic-fundamentals`, `low-concurrency/08-memory-ordering`
- Memory reclamation gets deeper treatment in `high-concurrency/06-mpmc-ring-buffer` (where it's most critical)
