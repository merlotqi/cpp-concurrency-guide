# Spec: 04-cpp-memory-model-intro

## Metadata

- **File**: `no-concurrency/04-cpp-memory-model-intro.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: 03-data-race-vs-race-condition
- **Estimated sections**: 5

## Learning Objectives

- Understand the C++ **abstract machine** and why it matters for concurrency
- Know what a **modification order** is — every object has one
- Grasp the "as-if" rule and why the compiler/CPU can reorder your code
- Recognize the gap between source code order and execution order

## Outline

### 1. The C++ Abstract Machine

- C++ code does not run as-is; the compiler translates to an abstract machine
- The "as-if" rule: any transformation is valid as long as observable behavior is preserved
- Observable behavior: volatile I/O, file I/O — but not memory accesses (by default)
- Implication for concurrency: the compiler *will* reorder your memory accesses

### 2. Modification Order

- Every object (including scalar variables) has a **modification order** — a total order of all writes from all threads
- Different threads may observe this order at different times (relaxed atomics)
- `std::atomic` guarantees a single, coherent modification order per object
- Non-atomic variables: no such guarantee — UB on concurrent write+read

### 3. Why Reordering Happens

- **Compiler reordering**: register allocation, common subexpression elimination, loop-invariant code motion
- **CPU reordering**: store buffers, load speculation, out-of-order execution
- Concrete example: `x = 1; y = 2;` may be stored as `y = 2; x = 1;` on another core's view
- The Dekker's algorithm counterexample — fails without proper barriers

### 4. The Problem with Single-Threaded Intuition

- In a single-threaded world, "my writes appear in program order" — always true
- In a multi-threaded world, this is only true with proper synchronization
- The mental model shift: from "the memory" to "my thread's view of memory"

### 5. Bridging to happens-before

- This chapter sets up the formal framework of happens-before (next chapter)
- The C++ memory model is the contract between programmer and compiler
- Without it, reasoning about concurrent code is impossible

## Key Concepts

| Concept | Depth |
|---------|-------|
| Abstract machine / as-if rule | Full |
| Modification order | Full |
| Compiler reordering | Medium |
| CPU reordering (store buffer etc.) | Light — just enough intuition |
| Memory location (C++ definition) | Medium |

## Pitfalls

- "It works on my machine" — the most dangerous phrase; reordering is architecture-dependent
- Assuming `volatile` helps (it doesn't — `volatile` is for MMIO, not threads)
