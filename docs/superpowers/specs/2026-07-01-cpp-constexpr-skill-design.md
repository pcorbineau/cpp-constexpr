# Skill: cpp-constexpr — Design Document

**Date:** 2026-07-01
**Status:** Draft

## 1. Overview

A multi-platform skill (opencode, Claude Code, Cursor, Copilot) for C++ experts that enforces a "constexpr/consteval all the things" philosophy. The skill transforms how agents approach C++ code: **everything is constexpr by default**; runtime operations are explicit exceptions isolated via `if not consteval` / `std::is_constant_evaluated()`.

## 2. Core Principles

1. **Constexpr by default** — every function starts as `constexpr`. The burden of proof is on why it CAN'T be constexpr, not on why it should.
2. **`if not consteval` / `if not std::is_constant_evaluated()` for runtime** — I/O, third-party calls, syscalls are encapsulated in `if not consteval { ... }` (or `if not std::is_constant_evaluated()`) blocks, not hidden in wrapper functions that break constexpr chains. Prefer `if not consteval` (C++23) when available; use `std::is_constant_evaluated()` (C++20) as fallback.
3. **Assert in constexpr path** — assertions stay in the direct code path. A failing assertion at compile time = compilation error.
4. **No assumptions about standard library** — verify whether a std function is constexpr by testing locally or fetching cppreference. No static tables.
5. **Third-party APIs** — wrap in a constexpr interface with `if not consteval { third_party_call(...); }` / `if not std::is_constant_evaluated()` delegation.
6. **Version-adaptive** — use `CONSTEXPR20`/`CONSTEXPR23` macros with fallback for code targeting older C++ standards.
7. **Tests are constexpr** — every constexpr function gets a constexpr test (`static_assert` or lambda in `namespace tests`).

## 3. Architecture: 3-Phase Pipeline

```
  [ANALYSIS] ──▶ [REFACTORING] ──▶ [CODE GENERATION]
```

Each phase is a flowchart-based decision tree with a checklist.

### 3.1 Phase 1: Analysis

Goal: evaluate existing code for constexpr eligibility.

```
  ┌─ Principle: everything is constexpr until proven otherwise ─┐
  │                                                              │
  │  - No assumptions about types (std::string, std::vector,     │
  │    third-party, etc.) — test directly                        │
  │  - Runtime operations are isolated with                      │
  │    if not consteval / std::is_constant_evaluated() — no      │
  │    wrapper                                                    │
  │  - If a function is only called in compile-time contexts     │
  │    → promote to consteval                                    │
  └──────────────────────────────────────────────────────────────┘
```

Checklist:
- [ ] Function starts as `constexpr`
- [ ] Std calls verified (local test or cppreference)
- [ ] Third-party → `if not consteval` / `std::is_constant_evaluated()` interface
- [ ] I/O/syscall → `if not consteval` / `std::is_constant_evaluated()` direct or interface
- [ ] Assert/check → direct path (no wrapper)
- [ ] Standard version checked

### 3.2 Phase 2: Refactoring

Goal: transform existing code to constexpr step by step.

```
  Add `constexpr` to signature
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
      │   party)         target standard
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

Checklist:
- [ ] `constexpr` on signature, compile
- [ ] Assert/check in constexpr path (not in `if not consteval` / `std::is_constant_evaluated()`)
- [ ] I/O, third-party, syscall → `if not consteval` / `std::is_constant_evaluated()`
- [ ] Version mismatch → macro `CONSTEXPR20`/`CONSTEXPR23` + fallback
- [ ] constexpr test in `namespace tests` (static_assert or lambda)

### 3.3 Phase 3: Code Generation

Goal: produce new code with constexpr from the start.

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

Code generation patterns:
- **constexpr wrapper** for runtime API (if not consteval / is_constant_evaluated + call)
- **constexpr-first API** (constexpr signature, direct assertions)
- **Pure compile-time helper** (consteval + source_location)
- **Version-adaptive** via macros

## 4. Key Patterns

```cpp
// Pattern: assert in constexpr path
constexpr auto check(bool cond) {
  assert(cond && "check failed");
}

// Pattern: runtime op wrapper
constexpr auto log(auto const& msg) {
  if not consteval {          // or: if not std::is_constant_evaluated()
    std::println("LOG: {}", msg);
  }
}

// Pattern: constexpr test
namespace tests {
  constexpr auto test_foo = []() {
    auto r = foo(42);
    assert(r == expected);
    return true;
  };
  static_assert(test_foo());
}

// Pattern: version-adaptive macro
#if __cplusplus >= 202002L
  #define CONSTEXPR20 constexpr
#else
  #define CONSTEXPR20 inline
#endif
```

## 5. Multi-Platform Adaptation

The skill is written as a SKILL.md with YAML frontmatter (name, description) and uses platform-agnostic action descriptions. Each platform (opencode, Claude Code, Cursor, Copilot) maps actions to their native tools.

## 6. Testing Strategy

Each skill iteration follows writing-skills TDD:
- Pressure scenarios (subagents) WITHOUT skill → document baseline failures
- Write minimal skill → re-test WITH skill
- Refactor: identify rationalizations, add explicit counters, red flags table
