# Spec: 07-immutability

## Metadata

- **File**: `no-concurrency/07-immutability.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: 06-const-and-thread-safety
- **Estimated sections**: 5

## Learning Objectives

- Understand that **immutability is the strongest thread-safety guarantee** — zero synchronization cost
- Distinguish **shallow** vs **deep** immutability
- Apply functional-programming patterns in C++ (value semantics, `const` propagation)
- Recognize when immutability is practical and when it hits performance limits

## Outline

### 1. The Zero-Cost Safety Guarantee

- Immutable data: created once, never modified, shared freely across threads
- No locks, no atomics, no happens-before reasoning needed
- The cost: you must create new objects instead of mutating — sometimes expensive, often not
- Where it shines: configuration, look-up tables, shared metadata, event data

### 2. Shallow vs Deep Immutability

- **Shallow**: the top-level pointer/reference doesn't change, but the pointed-to data might
- **Deep**: the entire object graph is immutable (transitive)
- Example: `const std::string* const` — shallow (string itself could be modified if not const)
- Example: `const std::string` — deep (assuming string contents are const)
- C++ provides tools (`const`, `constexpr`, `constinit`) but no language-level deep immutability

### 3. Functional Patterns in C++

- **Value semantics**: pass by value, return by value (move semantics make this cheap)
- **Builder pattern**: accumulate mutations in a builder, then construct the immutable object
- **Copy-on-write**: share until mutation, then deep-copy (but watch thread safety on the ref count)
- **Persistent data structures**: structural sharing (brief mention, not C++ strengths)

### 4. constexpr and constinit

- `constexpr`: compile-time evaluation → truly immutable, zero runtime cost
- `constinit` (C++20): guarantees static initialization at compile time, prevents the static initialization order fiasco
- Use case: global lookup tables, configuration constants

### 5. When NOT to Use Immutability

- Hot-path data that changes frequently → copy cost dominates
- Large objects → memory pressure from copies
- Cyclic data structures → hard to make immutable without GC
- Practical guideline: default to immutable, profile before abandoning

## Key Concepts

| Concept | Depth |
|---------|-------|
| Immutability = free thread safety | Full |
| Shallow vs deep const | Full |
| Value semantics for concurrency | Medium |
| constexpr / constinit | Medium |

## Code Examples

- Before: mutable shared config struct with mutex
- After: immutable config struct built once, shared by `shared_ptr<const Config>`
- constexpr lookup table for a parser (no runtime init, no synchronization)
- Copy-on-write string class (and why it's tricky with threads)

## Pitfalls

- Shallow const gives a false sense of safety: `const std::unique_ptr<T>` — pointer is const, but `*ptr` is not
- Copying large objects on every update without profiling
- constexpr limits (no dynamic allocation until C++20 `constexpr new`)
