# Spec: 05-happens-before

## Metadata

- **File**: `no-concurrency/05-happens-before.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: 04-cpp-memory-model-intro
- **Estimated sections**: 5

## Learning Objectives

- Define **sequenced-before**, **happens-before**, and **synchronizes-with** as formal relations
- Understand happens-before as a partial order, not a total order
- Apply the transitive closure rule: if A happens-before B and B happens-before C, then A happens-before C
- Use these relations to reason about data-race freedom without running code

## Outline

### 1. The Three Relations

- **Sequenced-before**: intra-thread ordering — what the source code says
- **Synchronizes-with**: inter-thread ordering — established by atomic operations and fences
- **Happens-before**: the transitive closure of sequenced-before ∪ synchronizes-with
- Diagram: two threads with arrows showing each relation

### 2. Sequenced-Before in Detail

- Full-expressions, statement order within a single thread
- The subtle cases: function argument evaluation order (unspecified! `f(a++, a++)` is UB)
- Expression evaluation order reform in C++17

### 3. Synchronizes-With in Detail

- Atomic store-release synchronizes-with atomic load-acquire (on the same atomic object)
- Mutex unlock synchronizes-with subsequent lock
- Thread creation synchronizes-with the start of the new thread function
- Thread join synchronizes-with the completion of the thread function
- Table: all synchronization points in the C++ standard

### 4. The Transitive Magic

- If A sequenced-before B, and B synchronizes-with C, and C sequenced-before D → A happens-before D
- This is how synchronization propagates across threads
- Example: thread 1 writes data, releases mutex; thread 2 acquires mutex, reads data → happens-before guarantees visibility

### 5. Applying Happens-Before

- The data race definition revisited: a program has a data race iff there are two conflicting actions not ordered by happens-before
- Walk through a mutex-protected counter — prove it's race-free using happens-before
- Walk through a broken example — find where happens-before is missing

## Key Concepts

| Concept | Depth |
|---------|-------|
| Sequenced-before | Full |
| Synchronizes-with | Full |
| Happens-before (transitive closure) | Full |
| Partial vs total order | Medium |
| Data race definition (formal) | Full |

## Pitfalls

- Confusing sequenced-before with happens-before (intra vs inter thread)
- Forgetting that argument evaluation is unsequenced before C++17
- Thinking "the code comes first in the file so it runs first" (reordering!)
