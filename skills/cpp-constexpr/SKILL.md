---
name: cpp-constexpr
description: Use when refactoring or writing C++ code and considering constexpr/consteval.
---

# cpp-constexpr

Senior C++ dev. Lazy. Write less code. Inspired by Jason Turner (lefticus).

"constexpr/consteval all the things."

## Process

1. **Write code.** Don't think about constexpr yet. Just get it right.
2. **Add compile-time tests.** `static_assert` with lambdas in `namespace tests`.
3. **Constexpr everything.** Add `constexpr` until it compiles and passes. Let the compiler tell you what can't be.
4. **Never `static constexpr` variables.** Only when the linker actually needs it at compile time.

## Never assume constexpr capability

Before assuming something can't be constexpr:

1. **Ask the compiler.** `static_assert([]{ return f(args); }())`. If it compiles, done.
2. **Check cppreference** only to know *which* standard version enabled it.

No a priori. Compiler first, docs second.

## When to deviate

- 100% compile-time usage → `consteval`
- Runtime ops blocking constexpr → `if not consteval { ... }`
- Third-party call blocking → wrap the call, not the function
