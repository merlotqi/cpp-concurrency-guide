# Spec: 02-thread-safety

## Metadata

- **File**: `no-concurrency/02-thread-safety.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: 01-why-concurrency
- **Estimated sections**: 5

## Learning Objectives

- Define **thread safety** precisely — not "it works," but *under what conditions* it's safe
- Know C++ Standard Library's thread-safety guarantees (the `const` member function contract)
- Understand the three levels: basic guarantee, strong guarantee, no-throw guarantee — and how they map to thread safety
- Distinguish **value type** vs **reference type** thread-safety behavior

## Outline

### 1. What Is Thread Safety?

- Definition: a function/class is thread-safe if it behaves correctly when invoked from multiple threads
- The key insight: thread safety is about **invariants** — can concurrent access break them?
- Examples: `std::vector::size()` (safe) vs `std::vector::push_back()` (unsafe)

### 2. C++ Standard Library Guarantees

- `[res.on.data.races]` in the standard
- Different `const` member functions on the same object can be called concurrently
- If any thread modifies an object, no other thread may access it concurrently
- What this means in practice: `shared_ptr` refcount is atomic, `vector::operator[]` is not

### 3. Levels of Thread Safety

- **Thread-unsafe**: breaks with any concurrent access
- **Thread-safe (basic)**: safe to call concurrently, but callers must coordinate
- **Thread-safe (strong)**: each operation is atomic in isolation
- **Thread-compatible**: safe only when external synchronization is provided
- Table: where common STL types land

### 4. Value Semantics and Thread Safety

- Immutable objects are always thread-safe
- Copy-on-write and its thread-safety implications
- Passing by value vs by reference in concurrent contexts

### 5. Designing for Thread Safety (Preview)

- Identify shared mutable state
- Minimize it (prefer stack, thread_local, immutable)
- Synchronize what remains
- This previews the rest of the guide's approach

## Key Concepts

| Concept | Depth |
|---------|-------|
| Thread safety definition | Full |
| STL thread-safety contract | Full |
| Invariant-based reasoning | Medium |
| Value vs reference semantics | Medium |

## Pitfalls

- "const means thread-safe" — it means *potentially* thread-safe; mutable members break it
- Assuming STL containers are thread-safe (they're not, except for const reads)
- `shared_ptr` is special — its control block is atomic, but pointed-to object is not
