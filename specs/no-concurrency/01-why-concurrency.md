# Spec: 01-why-concurrency

## Metadata

- **File**: `no-concurrency/01-why-concurrency.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: None (entry point)
- **Estimated sections**: 5–6

## Learning Objectives

After reading this chapter, the reader should:

- Distinguish **concurrency** from **parallelism** with concrete examples
- Understand the hardware trends (CPU frequency wall, multi-core) that made concurrency essential
- Know Amdahl's Law and its implication: why adding cores doesn't scale linearly
- Recognize the major categories of difficulty in concurrent programming (correctness, performance, debuggability)

## Outline

### 1. Opening: The Free Lunch Is Over

- Herb Sutter's 2005 article as historical context
- Single-core frequency plateau → multi-core era
- Why every C++ programmer must now care about concurrency

### 2. Concurrency vs. Parallelism

- Precise definitions with everyday analogies
- Concurrency = dealing with multiple things at once (structure)
- Parallelism = doing multiple things at once (execution)
- Venn diagram: overlapping but distinct concerns

### 3. Why Concurrency Matters

- Responsiveness (UI threads)
- Throughput (servers, data processing)
- Resource utilization (keeping all cores busy)
- Real-world examples: web server, game engine, database

### 4. Amdahl's Law

- Formula: S = 1 / ((1 - P) + P / N)
- The serial bottleneck — one slow part limits everything
- Interactive chart/table showing diminishing returns
- Implication: you must reduce serial portions, not just add threads

### 5. The Challenges Ahead (Preview)

- Correctness: data races, race conditions, deadlocks
- Performance: contention, false sharing, cache coherence
- Debugging: non-determinism, Heisenbugs
- This sets up the motivation for the entire guide

### 6. Chapter Map

- Visual diagram showing the three-phase journey ahead
- What each phase teaches and how they connect

## Key Concepts to Explain

| Concept | Depth | Notes |
|---------|-------|-------|
| Concurrency | Full | Use interleaved-task diagram |
| Parallelism | Full | Use simultaneous-execution diagram |
| Amdahl's Law | Medium | Formula + interactive intuition |
| Multi-core architecture | Light | Brief context, not a hardware deep-dive |

## Pitfalls to Flag

- "Just add more threads" fallacy — concurrency is not free performance
- Confusing concurrency with parallelism (pervasive mistake)

## Cross-References

- Serves as entry point → all subsequent chapters
- Reinforced by: `high-concurrency/10-performance-tuning` (Amdahl's Law in profiling)
