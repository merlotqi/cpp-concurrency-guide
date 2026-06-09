# C++ Concurrency Programming Guide

[English](./README.md) | [简体中文](./doc/README_zh.md)


## 📖 About

A systematic guide to **C++ concurrent programming**, organized into **three progressive phases**:

| Phase | Directory | Focus | Keywords |
|:---:|------|------|--------|
| I | [no-concurrency/](./no-concurrency/) | **Prerequisites** | Thread safety, data races, memory model, immutability |
| II | [low-concurrency/](./low-concurrency/) | **Concurrency Primitives** | `std::mutex`, `std::atomic`, condition variables, `std::lock_guard` |
| III | [high-concurrency/](./high-concurrency/) | **Advanced Architectures** | Thread pools, lock-free structures, coroutines, performance tuning, case studies |

> Each chapter follows a consistent pattern: **Motivation** → **Concepts** → **Code Examples** → **Common Pitfalls** → **Exercises**. Suitable for both self-study and team training.

---

## 🎯 Audience

- Developers with **basic C++ knowledge** (C++17/20)
- Engineers seeking **systematic concurrency education** rather than ad-hoc googling
- Anyone needing **end-to-end guidance** from theory to production
- Interview candidates preparing for concurrency/multithreading questions

---

## 🗺️ Roadmap

```
┌──────────────────────────────────────────────────────────────┐
│                     Phase I: Prerequisites                    │
│  no-concurrency/                                              │
│  ├── 01-basic-thread-safety          Thread safety basics     │
├──────────────────────────────────────────────────────────────┤
│                  Phase II: Concurrency Primitives             │
│  low-concurrency/                                             │
├──────────────────────────────────────────────────────────────┤
│                Phase III: Advanced Architectures              │
│  high-concurrency/                                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 🚀 Quick Start

### Prerequisites

- **Compiler**: GCC 11+ / Clang 14+ / MSVC 2022+ (C++20 support)
- **Build system**: CMake 3.20+
- **Optional tools**: ThreadSanitizer, perf, Valgrind (for debugging chapters)

### Build & Run

```bash
git clone https://github.com/your-org/cpp-concurrency-guide.git
cd cpp-concurrency-guide
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)

# Run all examples
ctest --output-on-failure

# Detect data races with ThreadSanitizer
cmake .. -DCMAKE_BUILD_TYPE=Debug -DENABLE_TSAN=ON
make -j$(nproc)
```

---

## 🤝 Contributing

Issues and pull requests are welcome!

- New chapters should follow the existing chapter template
- Ensure code samples compile under both GCC and Clang
- Run `clang-format` before committing
- Write documentation in English (Chinese translations in `*-zhCN.md` files)

See [CONTRIBUTING.md](./CONTRIBUTING.md) (coming soon).

---

## 📚 References

| Resource | Description |
|----------|-------------|
| [C++ Concurrency in Action (2nd Ed.)](https://www.manning.com/books/c-plus-plus-concurrency-in-action-second-edition) | The definitive guide by Anthony Williams |
| [CppCon Talks](https://www.youtube.com/@CppCon) | Annual concurrency & parallelism tracks |
| [cppreference.com](https://en.cppreference.com/w/cpp/thread) | Standard library concurrency reference |
| [C++ Core Guidelines: Concurrency](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-concurrency) | Official concurrency guidelines |

---

## 📄 License

This project is open-sourced under the [GNU General Public License v3.0](./LICENSE).

---

<p align="center">
  <sub>If this guide helps you, please consider giving it a ⭐!</sub>
</p>
