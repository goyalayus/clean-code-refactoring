# clean-code-refactoring

`clean-code-refactoring` is a small public skill repo for Codex and Claude Code that teaches one thing well: how to make Python code easier to read and easier to change.

It pushes toward direct call paths, fewer unnecessary wrappers, visible data shapes, early returns, and less cleverness. The point is not abstract clean-code talk. The point is practical refactoring you can apply in a real codebase without turning it into a pattern museum.

## What is in here

- `SKILL.md`: the skill itself

## Quick install

Clone this repo into the skill directory you use locally.

### Codex

```bash
git clone https://github.com/goyalayus/clean-code-refactoring.git ~/.codex/skills/clean-code-refactoring
```

### Claude Code

```bash
git clone https://github.com/goyalayus/clean-code-refactoring.git ~/.claude/skills/clean-code-refactoring
```

## Good prompts

- `Refactor this for readability.`
- `This module has too many wrapper functions. Simplify it.`
- `Clean up this call chain and remove unnecessary indirection.`
- `Make this code easier to follow without changing behavior.`

## Design philosophy

- Prefer directness over indirection.
- Prefer names that earn their keep.
- Show data shape when type annotations are not enough.
- Break long functions into real named phases, not ceremony.

If you want clean code that still feels like a human wrote it, this is aimed at that.
