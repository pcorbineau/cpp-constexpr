---
name: cpp-constexpr
description: Use when refactoring or writing C++ code and considering constexpr/consteval. Also when a function calls runtime operations (I/O, third-party, syscall) and callers need to remain constexpr. Also when tests should run at compile time.
---

# cpp-constexpr

## Overview

Enforce a "constexpr/consteval all the things" philosophy: **everything is constexpr by default**; runtime operations are explicit exceptions isolated via `if not consteval` (C++23) or `std::is_constant_evaluated()` (C++20).

Target: C++ experts. Standard-adaptive via `__cplusplus`. Multi-platform.

## Core Principles

1. **Constexpr by default** — every function starts as `constexpr`. Prove why it CAN'T be constexpr, not why it should.
2. **Runtime ops are explicit** — I/O, third-party calls, syscalls go in `if not consteval { ... }` / `if not std::is_constant_evaluated()`. No wrapper functions that break constexpr chains.
3. **Assert in constexpr path** — assertions stay in the direct code path. Failure at compile time = compilation error.
4. **No std assumptions** — verify std function constexpr status by testing or cppreference. No static tables.
5. **Third-party APIs** — wrap in a constexpr interface with `if not consteval { third_party_call(); }` / `std::is_constant_evaluated()` delegation.
6. **Version-adaptive** — use `CONSTEXPR20`/`CONSTEXPR23` macros with fallback.
7. **Tests are constexpr** — every constexpr function gets a `static_assert` or lambda test in `namespace tests`.

## Phase 1: Analysis

Goal: evaluate code for constexpr eligibility.

### Decision Flow

```
┌─ Principle: everything is constexpr until proven otherwise ─┐
│                                                              │
│  - No assumptions about types — test directly                │
│  - Runtime operations isolated with                          │
│    if not consteval / std::is_constant_evaluated()           │
│  - Only compile-time callers? → promote to consteval         │
└──────────────────────────────────────────────────────────────┘
```

### Checklist

- [ ] Function starts as `constexpr`
- [ ] Std calls verified (local test or cppreference)
- [ ] Third-party → `if not consteval` / `std::is_constant_evaluated()` interface
- [ ] I/O/syscall → `if not consteval` / `std::is_constant_evaluated()`
- [ ] Assert/check → direct path (no wrapper)
- [ ] C++ standard version checked

## Phase 2: Refactoring

Goal: transform existing code to constexpr step by step.

### Decision Flow

```
  constexpr on signature
         │
         ▼
      Compile
      │       │
     OK      FAIL
      │       │
      │   ┌───┴──────────────┐
      │   │                  │
      │  Runtime op      Feature too
      │  (I/O, third-    recent for
      │   party)         target std
      │   │                  │
      │   ▼                  ▼
      │  if not          Versioned macro
      │  consteval /      + fallback
      │  is_constant_
      │  evaluated()
      │   │                  │
      │   └──────┬───────────┘
      │          ▼
      │     Re-compile
      │
      ▼
  constexpr tests + namespace tests
      │
      ▼
  100% compile-time → consteval
```

### Checklist

- [ ] `constexpr` on signature, compile
- [ ] Assert/check in constexpr path (not behind `if not consteval` / `std::is_constant_evaluated()`)
- [ ] I/O, third-party, syscall → `if not consteval` / `std::is_constant_evaluated()`
- [ ] Version mismatch → macro `CONSTEXPR20`/`CONSTEXPR23` + fallback
- [ ] constexpr test in `namespace tests`

## Phase 3: Code Generation

Goal: produce new code with constexpr from the start.

### Decision Flow

```
  constexpr by default on every new function
         │
         ▼
  I/O? → if not consteval / is_constant_evaluated()
  assert? → direct path
  third-party? → constexpr interface
         │
         ▼
  constexpr tests (namespace tests)
         │
         ▼
  100% compile-time usage? → consteval
```

### Patterns

```cpp
// Runtime op wrapper: always use if not consteval (not if consteval)
constexpr auto log(auto const& msg) {
  if not consteval {          // or: if not std::is_constant_evaluated()
    std::println("LOG: {}", msg);
  }
}

// Constexpr-first API: assert stays in direct path
template <typename T>
constexpr auto clamp(T value, T min, T max) -> T {
  assert(min <= max && "clamp: min must be <= max");
  if (value < min) return min;
  if (value > max) return max;
  return value;
}

// Pure compile-time helper
template <auto V>
consteval auto type_name() -> std::string_view {
  return std::source_location::current().function_name();
}

// Version-adaptive macro
#if __cplusplus >= 202302L
  #define CONSTEXPR23 constexpr
#else
  #define CONSTEXPR23 inline
#endif
#if __cplusplus >= 202002L
  #define CONSTEXPR20 constexpr
#else
  #define CONSTEXPR20 inline
#endif
```

### Testing pattern

Every test includes a comment explaining the purpose. Single-line or multi-line as needed.

```cpp
namespace tests {
  // foo returns the expected value for positive input
  constexpr auto test_foo = []() {
    auto r = foo(42);
    assert(r == expected);
    return true;
  };
  static_assert(test_foo());

  // foo handles edge cases correctly
  constexpr auto test_foo_edge = []() -> bool {
    assert(foo(0) == 0);
    assert(foo(-1) == 0);
    return true;
  };
  static_assert(test_foo_edge());

  /*
    Verifies that foo works with constexpr std::string.
    This tests the C++20 constexpr string path.
  */
  constexpr auto test_foo_string = []() -> bool {
    auto r = foo(std::string("test"));
    assert(r > 0);
    return true;
  };
  static_assert(test_foo_string());
}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `if consteval { no-op } else { ... }` instead of `if not consteval { ... }` | Use `if not consteval` — it's the negative, focused form. Don't write empty constexpr branches. |
| Replacing `assert` with `throw` to "make it constexpr" | Keep assert in the direct path. If you need both compile-time and runtime checking, use `static_assert` for compile-time and keep assert for debug builds. |
| Creating a separate non-constexpr wrapper function | Use `if not consteval` inline instead of splitting into two functions. |
| Assuming std::string/vector aren't constexpr without testing | Test first — C++20 made them constexpr. |
| Not checking the C++ standard level | Use `__cplusplus` macros for fallback. |

## When to Use

| Trigger | Action |
|---------|--------|
| Function without I/O/syscalls | Make `constexpr` by default |
| Function with I/O or third-party calls | `constexpr` + `if not consteval` / `is_constant_evaluated()` wrapper |
| Function only called at compile time | Promote to `consteval` |
| Validation/check with assert | Keep assert in constexpr path (no wrapper) |

## When NOT to Use

- C++11/14-only codebase (no constexpr support)
- Code that genuinely cannot be isolated from runtime (hardware register access, signal handlers)

## Checking std constexpr status

```cpp
// Method 1: test locally
constexpr auto test = []{ return std::some_function(args); };
static_assert(test());

// Method 2: fetch cppreference
// webfetch("https://en.cppreference.com/w/cpp/...")
```

## Rationalizations to Avoid

| Excuse | Reality |
|--------|---------|
| "This calls std::println so it can't be constexpr" | `if not consteval` isolates the runtime call. |
| "assert is runtime-only" | assert works in constexpr — keep it in the direct path. |
| "Third-party library isn't constexpr" | Wrap the call, not the whole function. |
| "Too complex to make constexpr" | Add constexpr, let the compiler tell you what's wrong. |
| "Nobody will call this at compile time" | Make it constexpr anyway — the caller decides. |

## Red Flags — STOP and reconsider

- You're about to skip `constexpr` because the function "does I/O"
- You're wrapping an assert in `if not consteval`
- You're creating a separate non-constexpr wrapper "for runtime callers"
- You're writing `if consteval { } else { }` instead of `if not consteval { }`
- You're assuming a std type isn't constexpr without testing

**All of these mean:** apply the `constexpr` + `if not consteval` pattern instead.
