---
name: clean-code-refactoring
version: 1.0.0
description: Apply practical refactoring rules to Python codebases. Use this skill when the user asks to refactor code, clean up a module, reduce indirection, simplify a call chain, remove unnecessary abstractions, or improve code readability. This skill focuses on direct data flow, earned abstractions, visible data shapes, and small named steps.
---

# Clean Code Refactoring

This skill is for making Python code easier to read and easier to change.

It is deliberately narrow. It does not try to teach architecture, design patterns, or abstract "clean code" philosophy. It focuses on the four problems that make day-to-day Python code annoying to work with:

- values getting laundered through config dicts and wrapper layers
- functions that exist only to sound organized
- data structures whose real shape is hidden behind `Dict[str, Any]`
- long functions that do several meaningful things in sequence

Use these rules mechanically. The goal is not to make code "more elegant." The goal is to make a future reader understand the code faster.

## Default posture

- Prefer directness over indirection.
- Prefer fewer moving parts.
- Prefer early returns.
- Prefer names that help the reader at the call site.
- Do not extract helper functions unless they earn their name.
- Do not preserve abstractions just because they already exist.

## Rule 1: Trust the call chain

**Test:** when function A already knows a value and function B needs it, does A pass it directly, or does it hide that value inside a dict or override object and make B unpack it again?

If A has the value, pass it.

### Signs of violation

- A function receives `config` and immediately unpacks the same 2 or 3 keys every time.
- One function computes a value, stores it in a config override, and the next function reads it back out.
- A wrapper exists only to extract parameters and forward them unchanged.
- Callers keep building bags of optional state just to thread one field through.

### Before

```python
def get_rl_dataset(config, *, word_source=None):
    task_cfg = config.get("task", {})
    n_samples = task_cfg.get("synthetic_samples", 1024)
    seed = config.get("training", {}).get("seed", 3407)
    return generate_synthetic_wordle_dataset(
        config,
        seed,
        n_samples,
        word_source=word_source,
    )
```

### After

```python
def generate_synthetic_wordle_dataset(
    config,
    *,
    seed=None,
    n_samples=None,
    word_source=None,
):
    task_cfg = config.get("task", {})
    resolved_seed = seed if seed is not None else config.get("training", {}).get("seed", 3407)
    resolved_n_samples = (
        n_samples if n_samples is not None else task_cfg.get("synthetic_samples", 1024)
    )
```

The function that needs the values should usually resolve them itself. That removes the middleman and makes the call site easier to follow.

## Rule 2: A function must earn its name

**Test:** before extracting a block into a helper, make sure it passes at least one of these:

1. It is called from two or more places with the same logic.
2. The logic is not obvious and the name genuinely explains something.
3. It does real work beyond what the function name already says.

If reading the function name tells you the entire body, that is usually indirection, not abstraction.

### Signs of violation

- The function is called from exactly one place.
- The body is shorter than the signature.
- The body wraps a single call with no added logic.
- Inlining it would make the caller easier to read, not harder.

### Before

```python
def _config_with_overrides(config, *overrides):
    return merge_config_overrides(config, *overrides)
```

That helper does not earn its name. It should be deleted.

### After

```python
merged_config = merge_config_overrides(config, *overrides)
```

### Keep a shim only when it protects a real external contract

```python
def score_completion(...):
    """Backward-compat shim used by the dashboard."""
    return _score_constraint_completion(...)
```

If something outside the file imports the function, the wrapper may still be worth keeping. Say why in a short comment or docstring.

## Rule 3: Show the shape, not just the type

**Test:** if a reader sees a variable, dataclass field, parameter, or return type, can they picture a real value?

`Dict[str, Any]` is often technically correct and still useless.

When the type does not show the actual structure, add a concrete shape comment.

### Before

```python
@dataclass
class AccumulatedConstraints:
    green_positions: Dict[int, str]
    yellow_banned_positions: Dict[str, set]
    yellow_letters: set
    min_counts: Dict[str, int]
    max_counts: Dict[str, int]
```

### After

```python
@dataclass
class AccumulatedConstraints:
    green_positions: Dict[int, str]          # {0: "a", 3: "e"} - position -> confirmed letter
    yellow_banned_positions: Dict[str, set]  # {"r": {1, 4}} - letter -> banned positions
    yellow_letters: set                      # {"r", "s"} - present but not green
    min_counts: Dict[str, int]               # {"e": 2} - minimum required count
    max_counts: Dict[str, int]               # {"t": 0, "r": 2} - max allowed count
```

### Also apply this to config dicts and return dicts

```python
def _score_constraint_completion(
    prompt_text: str,
    completion_text: str,
    valid_set: set,
    task_cfg: Dict[str, Any],  # {"rewards": {"format": 0.2, "dict": 0.2}, "reward_profile": "constraint"}
    tokenizer: Any,
) -> Dict[str, Any]:
    # Returns: {"strict_ok": True, "parsed_guess": "crane", "reward_total": 0.85, ...}
```

### When to add shape comments

- Always for `Dict[str, Any]`, `Optional[Dict[str, Any]]`, `set`, tuple-heavy types, or nested containers.
- Usually for dataclass fields whose meaning is not obvious from the name alone.
- Usually for untyped or loosely typed return dicts.

### When not to

- `name: str`
- `count: int`
- `is_valid: bool`
- Any variable whose structure is obvious from both the name and the code directly below it

## Rule 4: Decompose into named steps

**Test:** if a function does several meaningful things in sequence, can each phase be described in a short phrase that represents real work?

If yes, extract those phases so the parent function reads like a plan.

This is the counterpart to Rule 2.

- Rule 2 says: do not extract trivial wrappers.
- Rule 4 says: do extract real phases inside long functions.

### Signs you should decompose

- The function is long enough that readers have to scroll to understand the flow.
- The code has natural phases separated by comments or blank lines.
- A block introduces local variables that do not matter to the rest of the function.
- Parsing, validation, scoring, formatting, and publishing are all mixed together.

### Before

```python
def _score_constraint_completion(...):
    # read config
    # parse completion
    # validate output length
    # parse prompt history
    # evaluate constraints
    # compute rewards
    # build result dict
```

### After

```python
def _score_constraint_completion(...):
    weights = _read_constraint_reward_weights(task_cfg)
    parse_result = _parse_and_validate_guess(completion_text, tokenizer, max_output_tokens)
    if not parse_result.strict_ok:
        return _build_failed_parse_result(parse_result, weights)

    history_rows = _parse_history_from_prompt(prompt_text)
    constraint_score = _evaluate_constraints(parse_result.guess, history_rows, valid_set)
    return _build_scored_result(parse_result, constraint_score, weights, history_rows)
```

The top-level function now reads like a short plan. Each helper earns its existence because it contains a full phase of work.

## How to apply this skill in practice

When refactoring a file:

1. Map the call chain.
2. Mark single-caller helpers as Rule 2 candidates.
3. Mark config encode-decode flows as Rule 1 candidates.
4. Mark `Dict[str, Any]` boundaries as Rule 3 candidates.
5. Mark long sequential functions as Rule 4 candidates.
6. Inline bottom-up.
7. Re-run syntax and tests after each meaningful change.

## Safe refactor checklist

- Check whether a helper is imported outside the file before deleting it.
- Preserve public APIs unless you update every consumer.
- Do not remove framework-required wrappers just because they look thin.
- If a name exists for backward compatibility, keep it and document why.
- Make one class of change at a time so regressions are easy to spot.

## What this skill should optimize for

- code that reads top to bottom without mental stack overflow
- direct parameter flow
- fewer wrappers
- fewer vague containers
- smaller surfaces for bugs
- easier reviews

## What this skill should not do

- introduce abstractions "for consistency"
- split every function into five tiny helpers
- replace readable code with patterns
- preserve unnecessary indirection because it feels organized
- chase style points at the expense of clarity

If two versions are both correct, prefer the one a tired engineer can understand on the first read.
