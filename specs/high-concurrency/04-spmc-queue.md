# Spec: 04-spmc-queue

## Metadata

- **File**: `high-concurrency/04-spmc-queue.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 03-spsc-ring-buffer
- **Estimated sections**: 6

## Learning Objectives

- Implement an **SPMC (Single Producer, Multiple Consumer)** ring buffer
- Understand the asymmetry: producer stays simple (SPSC-like), consumers need CAS on read_idx
- Design the consumer coordination: one shared read_idx, CAS to claim a slot
- Recognize SPMC use cases: log consumption by multiple parsers, event dispatch, broadcast (with fan-out)

## Outline

### 1. SPMC: The Asymmetric Model

- Single producer → no write contention → producer code is identical to SPSC
- Multiple consumers → they compete for read slots → need CAS on `read_idx`
- This is why SPMC comes before MPSC: the producer side is a direct copy from SPSC
- The new complexity: consumer CAS loop to claim the next read slot

### 2. Producer Side — Copy from SPSC

- Exactly the same as SPSC producer: check space, write data, store write_pos with release
- No contention from other producers → no atomic RMW needed
- The only atomics: `write_pos_.load(relaxed)` (own position), `read_pos_.load(acquire)` (consumer progress)
- This is the beauty of the asymmetric model: keep one side simple

### 3. Consumer Side — CAS the read_idx

```
bool try_pop(T& item) {
    uint64_t pos = read_pos_.load(relaxed);
    while (true) {
        // Check empty
        if (pos >= cached_write_pos_) {
            cached_write_pos_ = write_pos_.load(acquire);
            if (pos >= cached_write_pos_) return false;
        }
        // CAS to claim this slot
        if (read_pos_.compare_exchange_weak(pos, pos + 1, acq_rel, relaxed)) {
            item = std::move(buffer_[pos % Capacity]);
            return true;
        }
        // CAS failed → another consumer took this slot → pos updated by CAS, try next
    }
}
```

- The CAS loop: multiple consumers race; winner gets `pos`, losers get new `pos` value and retry
- `acq_rel` on CAS: acquire to see the data (release from producer), release to ensure data read completes before next consumer claims this slot
- Wait, we don't need release on the consumer CAS actually... The consumer only updates `read_pos`, and the producer only reads it. Let's analyze:
  - Consumer reads buffer[pos] (not atomic)
  - Consumer CAS read_pos to pos+1
  - Producer loads read_pos (acquire) to free slots
  - So CAS needs release to ensure buffer read happens before read_pos update
- Yes, `acq_rel` is correct: acquire for seeing producer's data, release for producer to see our progress

### 4. Cache Line Layout

- Producer writes: `write_pos_` — needs its own cache line
- Consumers CAS: `read_pos_` — needs its own cache line (separate from `write_pos_`)
- Buffer: separate cache line
- Producer also caches `read_pos_` locally → its own `cached_read_pos_`
- Each consumer caches `write_pos_` locally → its own `cached_write_pos_`
- Total: minimal cross-thread cache invalidation

### 5. Benchmark vs SPSC

| Configuration | SPSC | SPMC (2C) | SPMC (4C) | SPMC (8C) |
|---------------|------|-----------|-----------|-----------|
| 1P, N consumers | 80M | 55M | 40M | 25M |
| Notes | No CAS | CAS contention starts | Moderate contention | High contention |

- Key observation: SPMC throughput drops as consumer count increases — CAS contention
- But still much faster than mutex+queue at any consumer count
- CPU utilization: consumers that lose CAS spin briefly → some wasted cycles

### 6. Real-World SPMC

- **Log aggregation**: one logger thread → multiple processing threads (metrics, alerting, archiving)
- **Event dispatch**: main thread posts events → worker threads process them (each event processed once)
- **Packet fan-out**: NIC receives → multiple analysis threads each grab a packet
- Pattern: one source, many sinks, each item consumed by exactly one sink
- If you need every consumer to see every item → that's broadcast, not SPMC (use separate SPSC per consumer)

## Key Concepts

| Concept | Depth |
|---------|-------|
| SPMC asymmetry | Full |
| Consumer CAS loop | Full |
| Memory ordering for consumer CAS | Full |
| Scalability limit (CAS contention) | Medium |
| SPMC vs broadcast distinction | Medium |

## Code Examples

- Full `SPMCQueue` implementation (~100 lines)
- Consumer CAS loop with backoff (try yielding after N failures)
- Benchmark: 1P2C, 1P4C, 1P8C throughput comparison
- Comparison: SPMC vs N separate SPSC queues + producer round-robin

## Cross-References

- Producer side reuses: `high-concurrency/03-spsc-ring-buffer`
- Counterpart: `high-concurrency/05-mpsc-queue` (asymmetry reversed)
- Generalizes to: `high-concurrency/06-mpmc-ring-buffer` (both sides use CAS)

## Pitfalls

- ⚠️ Consumer CAS contention at scale → consider batching (each consumer grabs N items at once)
- ⚠️ Starvation: one consumer may always lose the CAS race → use backoff or round-robin
- ⚠️ `acq_rel` is easy to get wrong — if a consumer uses `release` only, it may not see producer's data
- ⚠️ The producer can be a straight copy-paste from SPSC → verify the memory orders still make sense
