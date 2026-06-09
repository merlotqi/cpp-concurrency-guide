# Spec: 05-mpsc-queue

## Metadata

- **File**: `high-concurrency/05-mpsc-queue.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: 03-spsc-ring-buffer, 04-spmc-queue
- **Estimated sections**: 6

## Learning Objectives

- Implement an **MPSC (Multiple Producer, Single Consumer)** ring buffer
- Master the producer CAS race for write_idx — the mirror image of SPSC consumer CAS
- Understand why MPSC is the natural model for **task schedulers** and **work queues**
- Compare throughput with SPSC, SPMC, and mutex-based queue

## Outline

### 1. MPSC: The Task Queue Model

- Multiple producers → they CAS `write_pos_` to claim a write slot
- Single consumer → no read contention → consumer code is simple (load/acquire)
- Mirror of SPMC: the asymmetry is reversed
- Natural fit for: one worker thread processing tasks submitted by many threads

### 2. Producer Side — CAS for Write Slot

```
bool try_push(T item) {
    uint64_t pos = write_pos_.load(relaxed);
    while (true) {
        // Check full
        if (pos - cached_read_pos_ >= Capacity) {
            cached_read_pos_ = read_pos_.load(acquire);
            if (pos - cached_read_pos_ >= Capacity) return false;
        }
        // CAS to claim this write slot
        if (write_pos_.compare_exchange_weak(pos, pos + 1, acq_rel, relaxed)) {
            buffer_[pos % Capacity] = std::move(item);
            return true;
        }
        // CAS failed → another producer got this slot → pos updated, retry
    }
}
```

- Notice: data is written **after** claiming the slot. Other producers have different slots → no data race
- `acq_rel` on CAS: acquire to see `read_pos` update from consumer, release to make data visible
- Actually, the `acquire` part is for `cached_read_pos_` refresh to be correct — producer needs to see consumer's release

### 3. Consumer Side — Simpler than SPSC? No.

- Single consumer → no CAS needed for reading
- But: the consumer sees slots that may not yet have data!
- Producer CAS claims slot → but data write happens *after* CAS
- Consumer must wait for slot to be "committed" — a slot is valid when `write_pos_ > slot_index`
- Actually, with the monotonic index approach, the consumer just reads `write_pos_` to know which slots are ready
- Consumer reads `write_pos_` (acquire), then reads all slots from its current position up to write_pos
- This naturally batches: consumer processes a batch of ready items at once

### 4. Batching in MPSC

- The consumer can read `write_pos_` once, then process many items:
  ```
  uint64_t ready = write_pos_.load(acquire);
  while (local_read < ready) {
      result.push(buffer_[local_read % Capacity]);
      local_read++;
  }
  read_pos_.store(local_read, release); // publish all at once
  ```
- Batching reduces: atomic store frequency for `read_pos_`, producer cache invalidations
- Throughput can approach SPSC levels with large batches

### 5. Benchmark & Comparison

| Configuration | Mutex+Queue | SPSC | MPSC (1P) | MPSC (2P) | MPSC (4P) | MPSC (8P) |
|---------------|-------------|------|-----------|-----------|-----------|-----------|
| Throughput | 5M | 80M | 75M | 55M | 40M | 28M |
| Dominant cost | Mutex | — | Store buffer | CAS contention | CAS contention | CAS contention |

- Single-producer MPSC ≈ SPSC (consumer side is the same)
- Multi-producer: CAS contention degrades gracefully compared to mutex

### 6. Real-World MPSC

- **Task scheduler**: N worker threads submit follow-up tasks → 1 scheduler thread
- **Network server**: N connection handler threads → 1 serialization/DB thread
- **UI framework**: Multiple event sources → 1 UI thread (Qt, WPF pattern)
- **Memory allocator**: N threads freeing memory → 1 reclamation thread
- MPSC is arguably the most useful asymmetric model in practice

## Key Concepts

| Concept | Depth |
|---------|-------|
| MPSC asymmetry | Full |
| Producer CAS for write claim | Full |
| Consumer batching | Full |
| Slot commitment timing | Full |
| Scalability limit (producer CAS) | Medium |

## Code Examples

- Full `MPSCQueue` implementation (~100 lines)
- Consumer batching: process N items per read_pos update
- Benchmark: MPSC vs mutex+queue, varying producer count
- Benchmark: effect of consumer batch size on throughput

## Cross-References

- Producer CAS pattern → reused in: `high-concurrency/06-mpmc-ring-buffer` (both sides)
- Consumer side pattern → from: `high-concurrency/03-spsc-ring-buffer` (single consumer)
- Used in: `high-concurrency/08-thread-pool` (task submission = MPSC)

## Pitfalls

- ⚠️ Slot not yet committed: producer wins CAS but hasn't written data yet — consumer can't read until write_pos advances
- ⚠️ With batching, large batches increase latency → trade batch size for latency budget
- ⚠️ Producer starvation: consumer that never updates read_pos quickly → producers spin on CAS (mitigate with timely read_pos updates)
- ⚠️ Single consumer is a bottleneck — if it can't keep up, queue fills and producers block/spin
