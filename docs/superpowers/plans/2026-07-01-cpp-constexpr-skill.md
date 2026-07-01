# cpp-constexpr Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a working, tested `cpp-constexpr` skill at `skills/cpp-constexpr/SKILL.md` that enforces the "constexpr/consteval all the things" philosophy.

**Architecture:** Single-file skill (SKILL.md) with YAML frontmatter, three phases (analysis, refactoring, code generation), flowchart-based decision trees, and pattern reference. Developed via writing-skills TDD: pressure scenarios first, then skill, then refactor.

**Tech Stack:** Markdown, YAML frontmatter (agentskills.io spec), C++ (patterns/examples)

## Global Constraints

- Skill name: `cpp-constexpr` (letters, numbers, hyphens only)
- Description starts with "Use when..." — triggering conditions, NOT workflow summary
- Description in third person, <500 chars
- All examples in C++
- Multi-platform: opencode, Claude Code, Cursor, Copilot
- C++ standard adaptive (`__cplusplus` macros)
- Both `if not consteval` (C++23) and `std::is_constant_evaluated()` (C++20) covered
- Assert stays in constexpr path (not behind `if not consteval`)
- No static tables for std library constexpr status — test or fetch cppreference
- Target: C++ experts

---

## File Structure

```
skills/cpp-constexpr/SKILL.md          # The skill itself
docs/superpowers/plans/2026-07-01-cpp-constexpr-skill.md  # This plan
docs/superpowers/specs/2026-07-01-cpp-constexpr-skill-design.md  # Design spec
```

---

### Task 1: RED — Create pressure scenarios & run baseline

**Files:**
- Create: `skills/cpp-constexpr/SKILL.md` (placeholder — will be overwritten in Task 2)

**Interfaces:**
- Consumes: Design spec at `docs/superpowers/specs/2026-07-01-cpp-constexpr-skill-design.md`
- Produces: Baseline failure documentation (rationalizations per scenario)

- [ ] **Step 1: Create 3 pressure scenarios**

Write three scenarios that test whether an agent follows the skill's rules. Save as embedded documentation in this plan.

```markdown
### Scenario 1: Runtime log wrapper
**Setup:** Codebase with a logging function used across many constexpr callers.
**User message:** "Refactor this logging utility so callers can still be constexpr. Keep the print at runtime."
```cpp
void log(std::string_view msg) {
    std::println("LOG: {}", msg);
}
```
```

**Expected correct behavior:** Agent wraps the body in `if not consteval` and marks function `constexpr`.
**Expected baseline failure (without skill):** Agent leaves function as `void` (non-constexpr), or creates a separate wrapper that breaks the constexpr chain.

```markdown
### Scenario 2: Constexpr assert
**Setup:** A validation function with an assert.
**User message:** "Make this function constexpr. The assert must still trigger at compile time if the condition is false."
```cpp
void check_positive(int val) {
    assert(val > 0 && "value must be positive");
}
```
```

**Expected correct behavior:** Agent adds `constexpr`, leaves assert in the direct path (no `if not consteval`).
**Expected baseline failure (without skill):** Agent wraps assert in `if not consteval`, or avoids `constexpr` entirely.

```markdown
### Scenario 3: Third-party API constexpr interface
**Setup:** Function that calls a third-party JSON library (non-constexpr).
**User message:** "Make this function constexpr. The third-party call should only happen at runtime."
```cpp
void log_json(std::string_view raw) {
    auto doc = nlohmann::json::parse(raw);
    std::println("parsed: {}", doc.dump());
}
```
```

**Expected correct behavior:** Agent wraps the third-party call and print in `if not consteval`, marks function `constexpr`.
**Expected baseline failure (without skill):** Agent gives up and leaves it runtime-only.

- [ ] **Step 2: Run Scenario 1 without skill**

Dispatch a fresh subagent (general-purpose) with Scenario 1's user message. No skill loaded.

Run:
```bash
# No specific command — dispatch subagent with prompt:
# "You are a C++ developer refactoring code. Here is the user request: [Scenario 1 message]"
# Provide the code, ask them to implement.
```

**Expected:** Agent produces non-constexpr solution or breaks the constexpr chain. Document the exact rationalization used.

Document result verbatim.

- [ ] **Step 3: Run Scenario 2 without skill**

Same as Step 2, with Scenario 2 message.

**Expected:** Agent wraps assert in `if not consteval` or avoids constexpr. Document.

- [ ] **Step 4: Run Scenario 3 without skill**

Same as Step 2, with Scenario 3 message.

**Expected:** Agent leaves function runtime-only. Document.

- [ ] **Step 5: Compile rationalization table**

From the 3 baseline runs, extract the rationalizations agents use:

| Scenario | Rationalization |
|----------|----------------|
| 1 | (e.g.) "The function does I/O so it can't be constexpr" |
| 2 | (e.g.) "assert is a runtime-only macro, hide it in if not consteval" |
| 3 | (e.g.) "Third-party library isn't constexpr-ready, skip constexpr" |

- [ ] **Step 6: Commit**

```bash
git add docs/superpowers/plans/2026-07-01-cpp-constexpr-skill.md
git commit -m "wip: add pressure scenarios for cpp-constexpr skill"
```

---

### Task 2: GREEN — Write minimal SKILL.md

**Files:**
- Create: `skills/cpp-constexpr/SKILL.md`

**Interfaces:**
- Consumes: Design spec + rationalization table from Task 1
- Produces: A SKILL.md that an agent can follow

- [ ] **Step 1: Write YAML frontmatter**

```yaml
---
name: cpp-constexpr
description: Use when refactoring or writing C++ code and considering constexpr/consteval. Also when a function calls runtime operations (I/O, third-party, syscall) and callers need to remain constexpr. Also when tests should run at compile time.
---
```

Verify: description starts with "Use when", is third person, <500 chars, no workflow summary.

- [ ] **Step 2: Write Overview + Core Principles**

```markdown
# cpp-constexpr

## Overview

Enforce a "constexpr/consteval all the things" philosophy: **everything is constexpr by default**; runtime operations are explicit exceptions isolated via `if not consteval` (C++23) or `std::is_constant_evaluated()` (C++20).

Target: C++ experts. Standard-adaptive via `__cplusplus`. Multi-platform.

## Core Principles

1. **Constexpr by default** — every function starts as `constexpr`. Prove why it CAN'T be constexpr, not why it should.
2. **Runtime ops are explicit** — I/O, third-party calls, syscalls go in `if not consteval { ... }` / `if not std::is_constant_evaluated()`. No wrapper functions that break constexpr chains.
3. **Assert in constexpr path** — assertions stay in the direct code path. Failure at compile time = compilation error.
4. **No std assumptions** — verify std function constexpr status by testing or cppreference. No static tables.
5. **Third-party APIs** — wrap in a constexpr interface with `if not consteval { third_party_call(); }` delegation.
6. **Version-adaptive** — use `CONSTEXPR20`/`CONSTEXPR23` macros with fallback.
7. **Tests are constexpr** — every constexpr function gets a `static_assert` or lambda test in `namespace tests`.
```

- [ ] **Step 3: Write Phase 1 — Analysis flowchart + checklist**

```markdown
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
```

- [ ] **Step 4: Write Phase 2 — Refactoring flowchart + checklist**

```markdown
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
- [ ] Assert/check in constexpr path (not behind `if not consteval`)
- [ ] I/O, third-party, syscall → `if not consteval` / `std::is_constant_evaluated()`
- [ ] Version mismatch → macro `CONSTEXPR20`/`CONSTEXPR23` + fallback
- [ ] constexpr test in `namespace tests`
```

- [ ] **Step 5: Write Phase 3 — Code Generation flowchart + patterns**

```markdown
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
// constexpr wrapper for runtime API
constexpr auto log(auto const& msg) {
  if not consteval {
    std::println("LOG: {}", msg);
  }
}

// constexpr-first API
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
#if __cplusplus >= 202002L
  #define CONSTEXPR20 constexpr
#else
  #define CONSTEXPR20 inline
#endif
```
```

- [ ] **Step 6: Write When to Use / When Not + Common Mistakes + Rationalization Table**

```markdown
## When to Use

| Trigger | Action |
|---------|--------|
| Function without I/O/syscalls | Make constexpr by default |
| Function with I/O or third-party calls | constexpr + `if not consteval` wrapper |
| Function only called at compile time | Promote to `consteval` |
| Validation/check with assert | Keep assert in constexpr path |

## When NOT to Use

- C++11/14-only codebase (no constexpr support)
- Code that genuinely cannot be isolated from runtime (hardware register access, signal handlers)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Wrapping assert in `if not consteval` | Move assert to direct path — it should fire at compile time too |
| Assuming std::string/vector aren't constexpr | Test first — C++20 made them constexpr |
| Creating a separate non-constexpr wrapper | Use `if not consteval` inline instead |
| Not checking the C++ standard level | Use `__cplusplus` macros for fallback |

## Rationalizations to Avoid

| Excuse | Reality |
|--------|---------|
| "This calls std::println so it can't be constexpr" | `if not consteval` isolates the runtime call |
| "assert is runtime-only" | assert works in constexpr — keep it in the direct path |
| "Third-party library isn't constexpr" | Wrap the call, not the whole function |
| "Too complex to make constexpr" | Add constexpr, let the compiler tell you what's wrong |
| "Nobody will call this at compile time" | Make it constexpr anyway — the caller decides |
```

- [ ] **Step 7: Write Constexpr Standard Reference**

```markdown
## Version Macro Pattern

```cpp
// Use these to adapt code to the target C++ standard
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

### Checking std library constexpr status

```cpp
// Method 1: test locally
constexpr auto test = []{ return std::some_function(args); };
static_assert(test());

// Method 2: fetch cppreference (webfetch)
// e.g. webfetch("https://en.cppreference.com/w/cpp/algorithm/find")
```
```

- [ ] **Step 8: Save and commit**

```bash
git add skills/cpp-constexpr/SKILL.md
git commit -m "feat: add cpp-constexpr skill (GREEN phase)"
```

---

### Task 3: REFACTOR — Re-test with skill & close loopholes

**Files:**
- Modify: `skills/cpp-constexpr/SKILL.md`

**Interfaces:**
- Consumes: SKILL.md from Task 2, baseline failures from Task 1
- Produces: Hardened SKILL.md with red flags table, new rationalization counters

- [ ] **Step 1: Run Scenario 1 WITH skill**

Dispatch a fresh subagent with Scenario 1's user message AND the skill loaded (instruct agent to read the skill at `skills/cpp-constexpr/SKILL.md` before implementing).

**Expected:** Agent produces `constexpr` function with `if not consteval { std::println(...); }`. Document any new rationalizations.

- [ ] **Step 2: Run Scenario 2 WITH skill**

Same as Step 1 with Scenario 2.

**Expected:** Agent produces `constexpr` with assert in direct path. Document.

- [ ] **Step 3: Run Scenario 3 WITH skill**

Same as Step 1 with Scenario 3.

**Expected:** Agent produces `constexpr` with `if not consteval { nlohmann::json::parse(...); }`.

- [ ] **Step 4: Identify new rationalizations**

If any scenario produced violations despite the skill, add explicit counters:

```markdown
| New rationalization | Counter |
|---|---|
| (from testing) | (specific counter) |
```

Add red flags section if not already present:

```markdown
## Red Flags — STOP and reconsider

- You're about to skip `constexpr` because the function "does I/O"
- You're wrapping an assert in `if not consteval`
- You're creating a separate non-constexpr wrapper "for runtime callers"
- You're assuming a std type isn't constexpr without testing

All of these mean: apply the `constexpr` + `if not consteval` pattern instead.
```

- [ ] **Step 5: Re-run all 3 scenarios to verify**

Run all 3 scenarios again with the hardened skill. All should pass.

- [ ] **Step 6: Commit**

```bash
git add skills/cpp-constexpr/SKILL.md
git commit -m "refactor: add rationalization counters and red flags"
```

---

### Task 4: Quality checks & finalize

**Files:**
- Verify: `skills/cpp-constexpr/SKILL.md`

- [ ] **Step 1: Run quality checklist**

- [ ] Skill name uses only letters, numbers, hyphens
- [ ] YAML frontmatter has `name` and `description` (max 1024 chars)
- [ ] Description starts with "Use when..."
- [ ] Description written in third person
- [ ] No workflow summary in description
- [ ] Keywords throughout for search (constexpr, consteval, compile-time, C++20, C++23)
- [ ] Clear overview with core principle
- [ ] One excellent C++ example (not multi-language)
- [ ] Flowchart only for non-obvious decisions
- [ ] Quick reference table (When to Use / Common Mistakes)
- [ ] Rationalization table + red flags
- [ ] No narrative storytelling
- [ ] No placeholders, TBD, or TODO
- [ ] < 500 words target (check with word count)

- [ ] **Step 2: Fix any issues**

Fix any quality issues inline.

- [ ] **Step 3: Final commit**

```bash
git add skills/cpp-constexpr/SKILL.md
git commit -m "chore: finalize cpp-constexpr skill with quality checks"
```

---

## Spec Coverage Verification

| Spec Requirement | Task |
|-----------------|------|
| "constexpr by default" principle | Task 2 (Core Principles) |
| `if not consteval` / `is_constant_evaluated()` alternatives | Task 2 (all phases + patterns) |
| Assert in constexpr path | Task 2 (Phase 2 checklist + pattern) |
| No std library assumptions | Task 2 (When to Use + Version Reference) |
| Third-party API wrapping | Task 2 (Phase 1 + Phase 3) |
| Version-adaptive macros | Task 2 (Phase 2 + Version Reference) |
| constexpr tests in namespace tests | Task 2 (Phase 2 checklist + pattern) |
| 3-phase pipeline | Task 2 (Phases 1, 2, 3) |
| Multi-platform | Task 2 (Overview) |
| TDD / testing | Task 1 (pressure scenarios) + Task 3 (re-test) |
