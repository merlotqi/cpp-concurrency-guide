# Spec: 10-performance-tuning

## Metadata

- **File**: `high-concurrency/10-performance-tuning.md`
- **Phase**: III — Advanced Architectures
- **Prerequisites**: All previous chapters (uses them as tuning targets)
- **Estimated sections**: 7

## Learning Objectives

- Profile concurrent C++ code with `perf`, VTune, and ThreadSanitizer
- Identify and fix the top 5 sources of concurrency performance loss
- Read a `perf stat` output: cache misses, branch mispredictions, IPC
- Generate and interpret flame graphs for multi-threaded programs
- Apply the **measure-don't-guess** methodology to concurrency optimization

## Outline

### 1. The Performance Tuning Mindset

- **Measure first, optimize second** — the golden rule
- Concurrency makes it harder: non-determinism, Heisenbugs, measurement perturbation
- The optimization workflow: profile → find bottleneck → hypothesize → test → verify
- Tools you need: `perf` (Linux), VTune (cross-platform), ThreadSanitizer, Google Benchmark
- What "fast" means: define your metric (throughput? latency p99? tail latency p999?)

### 2. The Five Horsemen of Concurrency Performance Loss

| # | Problem | Symptom | Tool |
|---|---------|---------|------|
| 1 | **Contention** | High CPU, low throughput, spinning on mutex/CAS | `perf record` + flame graph |
| 2 | **False sharing** | High cache misses, scaling drops at 2+ threads | `perf stat -e cache-misses` |
| 3 | **Context switch overhead** | High `%system`, futex syscalls dominate | `perf stat -e context-switches` |
| 4 | **Memory allocation contention** | `malloc`/`free` in hot path → arena lock | `perf record -e malloc` or heaptrack |
| 5 | **Imbalanced work distribution** | Some threads idle, others 100% → sub-linear scaling | Timeline visualization |

### 3. perf stat — The First Look

```
perf stat -e cycles,instructions,cache-misses,branch-misses,\
  context-switches,cpu-migrations,L1-dcache-load-misses,\
  LLC-load-misses ./my_benchmark
```

- **IPC** (instructions per cycle): < 1.0 → CPU is stalling (memory or dependencies); > 2.0 → well-optimized
- **Cache miss rate**: > 3% LLC misses → data layout problem
- **Context switches**: > 1000/s for a CPU-bound program → too many threads or contention
- **CPU migrations**: high number → threads bouncing between cores → use CPU affinity (`taskset`, `pthread_setaffinity`)

### 4. perf record + Flame Graphs

```
perf record -g -F 99 --call-graph dwarf ./my_benchmark
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

- Flame graph: width = time spent, top-to-bottom = call stack
- In concurrent code: look for wide `__lll_lock_wait` (mutex contention), `pthread_cond_wait` (CV), spin loops
- Identify: is time spent in your code (logic) or in synchronization (overhead)?
- Off-CPU flame graphs: what are threads doing when NOT on CPU? (blocked on mutex, CV, IO, futex)

### 5. ThreadSanitizer (TSan) for Performance

- TSan is not just for correctness — data races cause: volatile-like compiler barriers, torn values, missed optimizations
- Fixing races sometimes unlocks compiler optimizations (e.g., the compiler can hoist loads out of loops)
- TSan can also detect lock order inversions (potential deadlocks)
- Performance cost of TSan: 5-10x slowdown → use only in testing, not in production profiling

### 6. Concurrency-Specific Optimizations

| Technique | When to Apply | Expected Gain |
|-----------|---------------|---------------|
| Batching | High per-op overhead (mutex, allocation) | 2-10x throughput |
| Cache line padding | Two frequently-written atomics nearby | 2-5x throughput |
| Per-thread data (TLS) | Global counter/heap contention | Up to N-thread linear scaling |
| Read-copy-update (RCU) | Read-heavy, rarely-updated structure | Near-zero read overhead |
| Lock elision (TSX/HLE) | Contended mutex with rare conflicts | 2-5x under low contention |

### 7. Case Study: Tuning an MPMC Queue

Walk through a real optimization session:

1. **Baseline**: Naive MPMC queue, 4P4C, 20M ops/s
2. **perf stat**: IPC 0.6, cache-misses 15%, 40k context-switches/s
3. **perf record + flame graph**: 60% time in `compare_exchange_weak` spin loop
4. **Hypothesis 1**: false sharing between write_pos and read_pos
5. **Fix 1**: add `alignas(64)` → 30M ops/s, IPC 0.8
6. **Hypothesis 2**: CAS retries burning cycles → batch operations
7. **Fix 2**: each producer claims 8 slots per CAS → 55M ops/s, IPC 1.5
8. **Hypothesis 3**: allocation overhead → pre-allocated buffer
9. **Fix 3**: use ring buffer (already done) → no change
10. **Final**: 55M ops/s, 2.75x baseline. Bottleneck shifts to memory bandwidth.

## Key Concepts

| Concept | Depth |
|---------|-------|
| perf stat / record | Full |
| Flame graph interpretation | Full |
| IPC as performance metric | Full |
| Five horsemen diagnosis | Full |
| Optimization technique selection | Full |
| Case study walkthrough | Full |

## Code Examples

- Benchmark harness: Google Benchmark with `->Threads(1)->Threads(2)->...`
- `perf stat` one-liners for each bottleneck type
- Thread pinning: `cpu_set_t` + `pthread_setaffinity_np` for reproducible benchmarks
- TSan CMake integration: `-fsanitize=thread` + suppressions file
- Script: automated profiling run (benchmark → perf stat → flame graph)

## Cross-References

- All queue implementations from chapters 03-07 serve as tuning targets
- Thread pool from chapter 08
- const/immutability (free thread safety) from `no-concurrency/06` and `07`

## Pitfalls

- ⚠️ Optimizing without profiling → "premature optimization is the root of all evil" (Knuth)
- ⚠️ Microbenchmarking in isolation → real workloads have different contention patterns
- ⚠️ `perf` on virtualized/cloud instances: PMU may be incomplete → use software events
- ⚠️ Optimization for one platform (x86) may hurt another (ARM) → measure on target platform
- ⚠️ TSan false positives with custom synchronization → annotate with `__tsan_acquire`/`__tsan_release`
