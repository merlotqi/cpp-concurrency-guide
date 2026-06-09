# Spec: 03-spsc-ring-buffer

## Metadata

- **File**: `high-concurrency/03-spsc-ring-buffer.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 02-lockfree-design-space, low-concurrency/08-memory-ordering
- **Estimated sections**: 7

## Learning Objectives

- Implement a **correct and performant SPSC ring buffer** from scratch
- Understand why SPSC is the simplest lock-free model: no CAS needed, just atomic load/store
- Apply cache-line padding to eliminate false sharing between producer and consumer
- Benchmark against `std::mutex` + `std::queue` and the Phase II `BlockingQueue`
- Recognize real-world SPSC use cases (audio pipeline, log buffer, event loop)

## Outline

### 1. Why SPSC Is Special

- One writer, one reader → no contention on the same index
- Producer only writes `write_idx`; consumer only writes `read_idx`
- Producer reads `read_idx` to check if full; consumer reads `write_idx` to check if empty
- **No CAS loop** — just `load` and `store` — near-zero synchronization overhead
- This is the "hello world" of lock-free programming

### 2. The Ring Buffer Mathematics

- Fixed array of capacity N
- Two indices: `write_pos` (producer) and `read_pos` (consumer)
- `size() = write_pos - read_pos` (monotonic indices, no modulo for position tracking)
- `enqueue(T)`: write to `buffer[write_pos % N]`, then `write_pos++`
- `dequeue(T)`: read from `buffer[read_pos % N]`, then `read_pos++`
- Monotonic indices (never reset) → no ambiguity between "full" and "empty"
- Indices are `uint64_t` — wrap at 2^64 is not a practical concern (centuries at 1GHz increment)

### 3. Memory Order Selection (The Heart of the Matter)

| Operation | Variable | Order | Why |
|-----------|----------|-------|-----|
| Producer writes data | buffer slot | (non-atomic) | |
| Producer publishes | `write_pos.store(val, release)` | release | Makes data visible to consumer |
| Consumer reads `write_pos` | `write_pos.load(acquire)` | acquire | Sees the data that producer published |
| Consumer reads data | buffer slot | (non-atomic) | Now safe thanks to acquire |
| Consumer publishes progress | `read_pos.store(val, release)` | release | Tells producer slot is free |
| Producer reads `read_pos` | `read_pos.load(acquire)` | acquire | Sees that consumer freed the slot |

- Key insight: the atomic write_pos and read_pos act as **one-way doors** for data visibility
- Diagram: timeline showing data, write_pos, read_pos across two threads

### 4. Full Implementation (~80 lines)

```cpp
template<typename T, size_t Capacity>
class SPSCQueue {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of 2");
    
    alignas(64) std::atomic<uint64_t> write_pos_{0};
    alignas(64) std::atomic<uint64_t> read_pos_{0};   // separate cache lines!
    alignas(64) std::array<T, Capacity> buffer_;       // separate cache line
    
    uint64_t cached_read_pos_{0};  // producer-local cache
    uint64_t cached_write_pos_{0}; // consumer-local cache
    
public:
    bool try_push(T item) {
        const uint64_t pos = write_pos_.load(relaxed);
        // Check full: use cached read_pos, refresh if needed
        if (pos - cached_read_pos_ >= Capacity) {
            cached_read_pos_ = read_pos_.load(acquire);
            if (pos - cached_read_pos_ >= Capacity) return false;
        }
        buffer_[pos % Capacity] = std::move(item);
        write_pos_.store(pos + 1, release);
        return true;
    }
    
    bool try_pop(T& item) {
        // Mirror of try_push
    }
};
```

- Explanation of the **cached index** pattern: each thread keeps a local copy of the other's index to avoid frequent atomic loads
- Why capacity must be power of 2: `pos % Capacity` → `pos & (Capacity - 1)` (free modulo!)

### 5. False Sharing Prevention

- Producer touches `write_pos_` frequently; consumer touches `read_pos_` frequently
- If on the same cache line: every `write_pos_.store` invalidates consumer's cache → consumer re-fetches → 10-50x slowdown
- Solution: separate cache lines with `alignas(64)` padding
- Also pad the buffer from the indices to avoid false sharing between indices and data

### 6. Benchmark

| Implementation | 1P1C throughput | Notes |
|----------------|---------------|-------|
| `std::mutex` + `std::queue` | ~5M ops/s | Mutex overhead dominates |
| `BlockingQueue` (Phase II ch11) | ~6M ops/s | CV adds syscall cost |
| SPSC (no padding) | ~20M ops/s | False sharing penalty |
| SPSC (padded) | ~80M ops/s | Near-memory-bandwidth limited |

- Graph: throughput vs message size (16B, 64B, 256B, 1KB)
- CPU utilization comparison: mutex version pegs CPU at 100% (spinning); SPSC at ~30% for same throughput

### 7. Real-World Applications

- **Audio/video pipelines**: fixed-rate data flow, one source, one sink → perfect SPSC fit
- **Logging**: application threads → log collector thread (often SPSC per-thread)
- **Network packet processing**: NIC ring → application (hardware SPSC)
- **Game engine event loop**: input thread → game logic thread

## Key Concepts

| Concept | Depth |
|---------|-------|
| Monotonic index pattern | Full |
| acquire/release for data visibility | Full |
| Cached remote index for batching | Full |
| False sharing elimination | Full |
| Power-of-2 capacity | Practical |

## Code Examples

- Full SPSCQueue implementation (header-only)
- Simple benchmark: 10M push/pop pairs, measure ops/sec
- TSan verification: push/pop on same thread → no races; concurrent push/pop by same thread → intentional
- Bug demo: SPSC without padding → TSan false sharing? No, but `perf stat` shows 3x cache misses
- Comparison table: SPSC with and without cached remote index (batch-read optimization)

## Cross-References

- Comparison baseline: `low-concurrency/11-producer-consumer-basic`
- Scales up to: `high-concurrency/04-spmc-queue` (add CAS for multiple consumers)
- Used in: `high-concurrency/07-blocking-lockfree-queue` (add futex blocking to SPSC)
- Cache line knowledge from: `high-concurrency/02-lockfree-design-space`

## Pitfalls

- ⚠️ Forgetting padding → false sharing kills performance (and you won't see it without profiling)
- ⚠️ Using modulo (%) instead of bitmask (&) — `%` is ~10x slower on hot path
- ⚠️ `uint32_t` indices → wrap at 4B items; use `uint64_t`
- ⚠️ Non-power-of-2 capacity → expensive modulo; Capacity & (Capacity-1) != 0 static_assert saves this
- ⚠️ Producer stores before write_pos → data not visible; release store fixes this
