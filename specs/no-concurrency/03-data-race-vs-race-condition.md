# Spec: 03-data-race-vs-race-condition

## Metadata

- **File**: `no-concurrency/03-data-race-vs-race-condition.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: 02-thread-safety
- **Estimated sections**: 5

## Learning Objectives

- Define **data race** precisely as defined by the C++ Standard (§[intro.races])
- Define **race condition** as a higher-level logical problem
- Understand why you can have a race condition *without* a data race (and vice versa, in theory)
- Learn to use ThreadSanitizer (TSan) to detect data races in practice

## Outline

### 1. Two Concepts Everyone Confuses

- The distinction is critical: data race → undefined behavior; race condition → logical bug
- Analogy: data race is like memory corruption, race condition is like TOCTOU
- Both are bad, but they require different fixes

### 2. Data Race (C++ Standard Definition)

- §[intro.races] paragraph 2: two accesses to the same memory location, at least one is a write, at least one is not atomic, not ordered by happens-before
- UB per the standard — the compiler may do *anything*
- Example: two threads increment `int counter` without synchronization
- Compiler may optimize away the read in a loop → infinite loop, torn writes, etc.

### 3. Race Condition

- Definition: program correctness depends on relative timing of events
- No UB, but wrong result
- Classic example: bank transfer — check balance then withdraw, but balance changes in between
- Example with mutex: two threads both check `queue.empty()` before `pop()` — safe from data race, but still a race condition

### 4. Detecting Data Races with ThreadSanitizer

- Compiler flag: `-fsanitize=thread`
- What TSan detects (data races) and what it doesn't (race conditions)
- Reading TSan output: the two stack traces, which variable, which threads
- False positives and how to handle them (`__tsan_acquire`/`__tsan_release` annotations)

### 5. Preventing Both

- Data race → use atomics or mutexes
- Race condition → restructure logic, check-then-act → act-then-check
- The fundamental pattern: make compound operations atomic

## Code Examples

- `int counter` increment race → TSan output walkthrough
- Bank transfer race condition → correct version with atomic compare-and-swap
- `if (!q.empty()) q.pop()` → correct with `try_pop`

## Key Concepts

| Concept | Depth |
|---------|-------|
| Data race (standard definition) | Full |
| Race condition | Full |
| TSan usage | Practical |
| Undefined behavior in concurrency | Medium |
