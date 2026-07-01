# cpp-constexpr

A multi-platform agent skill that enforces a **constexpr/consteval all the things** philosophy for C++ code.

## Description

This skill transforms how AI agents approach C++: **everything is constexpr by default**. Runtime operations (I/O, third-party calls, syscalls) are explicit exceptions isolated via `if not consteval` / `std::is_constant_evaluated()`.

The skill provides a 3-phase pipeline:
1. **Analysis** — evaluate existing code for constexpr eligibility
2. **Refactoring** — transform code to constexpr step by step
3. **Code Generation** — produce new code with constexpr from the start

## Installation

Skills are loaded by the agent at runtime. Place the skill in your platform's skills directory:

| Platform | Path |
|----------|------|
| [opencode](https://opencode.ai) | `~/.config/opencode/skills/cpp-constexpr/SKILL.md` |
| Claude Code | `~/.claude/skills/cpp-constexpr/SKILL.md` |
| Cursor | `~/.cursor/agent/skills/cpp-constexpr/SKILL.md` |
| Copilot CLI | `~/.config/github-copilot/skills/cpp-constexpr/SKILL.md` |

Or clone this repo and symlink/copy:

```bash
git clone https://github.com/pcorbineau/cpp-constexpr.git
cp -r cpp-constexpr/skills/cpp-constexpr ~/.config/opencode/skills/
```
