# Spec: 06-const-and-thread-safety

## Metadata

- **File**: `no-concurrency/06-const-and-thread-safety.md`
- **Phase**: I — Prerequisites
- **Prerequisites**: 02-thread-safety, 05-happens-before
- **Estimated sections**: 4

## Learning Objectives

- Understand why `const`-correctness is a **thread-safety tool**, not just a style preference
- Know the pitfalls of `mutable` members in concurrent contexts
- Recognize the `const` member function contract from the STL's thread-safety guarantees
- Learn to use `std::atomic` members inside `const` methods for caching/lazy evaluation

## Outline

### 1. const as a Thread-Safety Contract

- A `const` member function promises not to modify observable state
- If all threads only call `const` methods, no data race can occur (per the STL guarantee)
- This is a *structural* guarantee — the compiler helps enforce it
- Counterexample: a non-const `getter` that modifies internal cache → invites races

### 2. The mutable Trap

- `mutable` lets a `const` method modify a member — it explicitly opts out of the `const` contract
- Classic use: caching computed values (e.g., `mutable std::map<K,V> cache`)
- The thread-safety implication: `mutable` members turn `const` calls into potential data races
- Solutions: wrap `mutable` members in `std::atomic` or `std::mutex`

### 3. Patterns for Thread-Safe const Methods

- **Pattern 1**: Use `std::atomic<T>` for mutable members → lock-free reads
- **Pattern 2**: Use `mutable std::mutex` + `mutable T` → pay lock cost only when cache miss
- **Pattern 3**: Eager initialization — compute in constructor, then truly const
- **Pattern 4**: `std::call_once` for lazy one-time initialization

### 4. const and the Standard Library

- STL containers: const member functions are safe to call concurrently
- `std::shared_ptr`: `use_count()` is const and thread-safe; dereferencing is not
- `std::string`: `c_str()` and `size()` are const — safe to call concurrently
- Guideline: follow the STL's lead — make your own classes behave the same way

## Key Concepts

| Concept | Depth |
|---------|-------|
| const as thread-safety guarantee | Full |
| mutable + thread safety | Full |
| Pattern catalog for const methods | Full |
| STL const contract | Medium |

## Code Examples

- Broken: mutable counter incremented in const method → TSan fires
- Fixed: `mutable std::atomic<int>` counter
- Fixed: `mutable std::mutex` + `mutable std::optional<Expensive> cache`
- Lazy singleton with `std::call_once`

## Pitfalls

- "It's const so it must be thread-safe" — false if `mutable` is involved
- Using `const_cast` to bypass const — don't
- Over-synchronization: wrapping everything in mutex, killing read scalability
