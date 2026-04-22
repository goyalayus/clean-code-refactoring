---
name: clean-code-refactoring
description: Apply practical refactoring rules to Python codebases. Use this skill when the user asks to refactor code, clean up a module, reduce indirection, simplify a call chain, remove unnecessary abstractions, or improve code readability. Also trigger when the user says things like "this code is hard to follow", "too many wrapper functions", "simplify this", or "make this more direct". This skill provides two concrete, testable rules with real examples — not generic "clean code" platitudes.
---

# Clean Code Refactoring

Three rules for deciding what to refactor and how to make what remains readable, plus one rule for when to extract. Each rule has a test you can apply mechanically, signs that tell you when code violates it, and before/after examples.

These rules are deliberately narrow. They don't cover architecture or design patterns. They cover the four most common sources of confusion in production Python: unnecessary indirection in data flow, unnecessary indirection in function structure, opaque data shapes, and monolithic functions that do too many things in sequence.

---

## Rule 1: Trust the call chain

**The test:** When function A knows a value and function B needs it, does A pass it directly? Or does A encode it into a config dict, merge it, and B decodes it back out?

If A already has the value, pass it. Don't launder it through a bag.

### Signs of violation

- Setting a config key then `.get()`-ing it back one call later
- Computing a value, including it in a total, and the caller subtracting it out
- Returning `Dict[str, Any]` with defensive `.get()` everywhere
- `json.loads(json.dumps(config))` to deep-copy a config just to tweak one key
- A function that receives `config` and immediately unpacks the same 2-3 keys every time

### Example: parameter extraction middleman

Before — `get_rl_dataset` exists only to extract two values from config and forward them:

```python
def get_rl_dataset(config, *, word_source=None):
    task_cfg = config.get("task", {})
    n_samples = task_cfg.get("synthetic_samples", 1024)
    seed = config.get("training", {}).get("seed", 3407)
    return generate_synthetic_wordle_dataset(config, seed, n_samples, word_source=word_source)

# Caller:
build=lambda config: get_rl_dataset(config, word_source=word_source)
```

After — `generate_synthetic_wordle_dataset` reads its own parameters from config:

```python
def generate_synthetic_wordle_dataset(config, *, seed=None, n_samples=None, word_source=None):
    task_cfg = config.get("task", {})
    resolved_seed = seed if seed is not None else config.get("training", {}).get("seed", 3407)
    resolved_n_samples = n_samples if n_samples is not None else task_cfg.get("synthetic_samples", 1024)
    # ... rest of function

# Caller:
build=lambda config: generate_synthetic_wordle_dataset(config, word_source=word_source)
```

The middleman is gone. The function that needs the values reads them itself. Callers that want to override can still pass explicit `seed=` or `n_samples=`.

### Example: redundant tuple unpacking

Before — `get_wordlists` returns the same list twice, every caller immediately unions them into a set:

```python
def get_wordlists(config, *, word_source=None):
    words = _load_word_source(resolved_source)
    return words, words  # same list twice

# Every single caller:
solutions, allowed = get_wordlists(config, word_source=word_source)
valid_set = set(solutions) | set(allowed)
```

After — return what the callers actually need:

```python
def load_valid_word_set(config, *, word_source=None):
    return set(_load_word_source(resolved_source))

# Caller:
valid_set = load_valid_word_set(config, word_source=word_source)
```

### Example: config encode-decode round-trip

Before — a function builds a config override dict, another function merges it, and the target function unpacks the same keys:

```python
def _turn_window_override(min_turns, max_turns):
    return {"task": {"min_history_turns": min_turns, "max_history_turns": max_turns}}

def _config_with_overrides(config, *overrides):
    return merge_config_overrides(config, *overrides)

# Usage:
build=lambda config: get_rl_dataset(
    _config_with_overrides(config, _turn_window_override(1, 5)),
    word_source=word_source,
)
```

After — inline the dict literal, call the merge directly:

```python
build=lambda config: generate_synthetic_wordle_dataset(
    merge_config_overrides(config, {"task": {"min_history_turns": 1, "max_history_turns": 5}}),
    word_source=word_source,
)
```

Two functions deleted, zero behavior change.

---

## Rule 2: A function must earn its name

**The test:** Before extracting a block of code into a named function, it must pass at least one of these:

1. Called from 2+ places with identical logic
2. The logic is non-obvious (a reader benefits from the name as documentation)
3. It does real work beyond what the name already tells you

If the name IS the implementation — if reading the function name tells you exactly what the body does — it's indirection, not abstraction.

### Signs of violation

- Called from exactly one place
- Body is shorter than the function signature
- Wraps a single call with zero added value
- You could inline it and the call site would be *more* readable, not less
- The function name is just a verbose restatement of the one-line body

### Example: single-call wrapper with zero added value

Before:

```python
def _config_with_overrides(config, *overrides):
    return merge_config_overrides(config, *overrides)
```

This function's name is `_config_with_overrides`. Its body is `merge_config_overrides(config, *overrides)`. The name adds nothing the reader doesn't already know. Delete it, call `merge_config_overrides` directly.

### Example: trivial computation dressed up as a function

Before:

```python
def _history_len_for_turn(turn):
    resolved_turn = int(turn)
    if resolved_turn < 1 or resolved_turn > 6:
        raise ValueError(f"Wordle turn must be within 1..6, got {turn}.")
    return max(0, resolved_turn - 1)
```

Called from 2 places, but the "real work" is `turn - 1`. The validation is the only non-trivial part. Split it:

After:

```python
def _validate_turn(turn):
    if int(turn) < 1 or int(turn) > 6:
        raise ValueError(f"Wordle turn must be within 1..6, got {turn}.")

# At call sites:
_validate_turn(turn)
history_len = turn - 1
```

The validation earns its function (it's reused, prevents bugs). The arithmetic doesn't.

### Example: backward-compat dispatcher nobody calls

Before:

```python
def get_reward_funcs(config, tokenizer):
    """Backward-compatible reward dispatcher used by named runs."""
    task_cfg = config.get("task", {})
    reward_profile = _resolve_reward_profile(task_cfg)
    if reward_profile == _PRIME_NO_LENGTH_REWARD_PROFILE:
        return get_prime_reward_funcs(config, tokenizer)
    return get_constraint_reward_funcs(config, tokenizer)
```

No experiment file calls this. The template system calls `get_constraint_reward_funcs` and `get_prime_reward_funcs` directly. This function exists "in case someone needs it." Nobody does. Delete it.

**Exception:** If an external consumer (like a dashboard, CLI tool, or test suite) imports the function, keep a shim with a comment saying why:

```python
def score_completion(...):
    """Backward-compat dispatcher used by the dashboard (server.py)."""
    ...
```

The comment justifies the function's existence. Without it, the next person will try to delete it and break the dashboard.

---

## Rule 3: Show the shape, not just the type

**The test:** If a reader sees a variable, parameter, or field declaration, can they immediately picture what a real value looks like? `Dict[str, Any]` tells you almost nothing. An inline comment with one concrete example tells you everything.

This applies to dataclass fields, function parameters, return values, and any variable whose structure isn't obvious from the type annotation alone.

### Signs of violation

- `Dict[str, Any]` with no hint of what keys exist or what the values look like
- A dataclass with five fields and zero comments — the reader has to trace every constructor call to understand what the data looks like
- A function that returns a dict, and every caller does defensive `.get()` because nobody can remember what keys are guaranteed
- Config dicts that travel through 3 functions before anyone explains their contents

### Example: dataclass fields with no shape context

Before:

```python
@dataclass
class AggregatedConstraints:
    green_positions: Dict[int, str]
    yellow_banned_positions: Dict[str, set]
    yellow_letters: set
    min_counts: Dict[str, int]
    max_counts: Dict[str, int]
```

A reader sees `green_positions: Dict[int, str]` and has to guess: is the int a letter index? A turn number? Is the str a single character or a word?

After:

```python
@dataclass
class AccumulatedConstraints:
    green_positions: Dict[int, str]         # {0: "a", 3: "e"} — position -> confirmed letter
    yellow_banned_positions: Dict[str, set]  # {"r": {1, 4}} — letter -> positions where it can't be
    yellow_letters: set                      # {"r", "s"} — letters confirmed present but not green
    min_counts: Dict[str, int]               # {"r": 1, "e": 2} — letter -> minimum occurrences
    max_counts: Dict[str, int]               # {"t": 0, "r": 2} — letter -> maximum occurrences (0 = absent)
```

Every field now has a concrete example that shows the exact shape. A reader immediately knows that `green_positions` maps position index to a single character, that `max_counts` of 0 means the letter is absent, and that `yellow_banned_positions` values are sets of integer positions.

### Example: config dict with unknown keys

Before:

```python
def _score_constraint_completion(
    prompt_text: str,
    completion_text: str,
    valid_set: set,
    task_cfg: Dict[str, Any],
    tokenizer: Any,
) -> Dict[str, Any]:
```

What's in `task_cfg`? What keys does the return dict have? You'd have to read the entire function body to find out.

After — add a shape comment at the definition or first meaningful use:

```python
def _score_constraint_completion(
    prompt_text: str,
    completion_text: str,
    valid_set: set,
    task_cfg: Dict[str, Any],
    # task_cfg example: {"rewards": {"format": 0.2, "dict": 0.2, "constraint": 0.5}, "reward_profile": "constraint"}
    tokenizer: Any,
) -> Dict[str, Any]:
    # Returns: {"strict_ok": True, "parsed_guess": "crane", "reward_total": 0.85, "sat_count": 4, ...}
```

### Example: intermediate variables in a computation

Before:

```python
totals = {"green": 0, "yellow": 0, "absent": 0, "maxcap": 0}
sats = {"green": 0, "yellow": 0, "absent": 0, "maxcap": 0}
```

What do `totals` and `sats` represent? Are they counts? Ratios? Booleans packed in a dict?

After:

```python
totals = {"green": 0, "yellow": 0, "absent": 0, "maxcap": 0}  # count of constraints by type
sats = {"green": 0, "yellow": 0, "absent": 0, "maxcap": 0}    # count of satisfied constraints by type
```

### When to add shape comments

- **Always** on dataclass/namedtuple fields whose types involve `Dict`, `List[Dict]`, `set`, `Tuple`, or `Any`
- **Always** on function parameters typed as `Dict[str, Any]` or `Optional[Dict[str, Any]]`
- **Always** on return values that are untyped dicts (the common `-> Dict[str, Any]` pattern)
- **Optionally** on local variables where the name alone doesn't convey the structure

### When NOT to add shape comments

- The type is self-evident: `name: str`, `count: int`, `is_valid: bool`
- The variable name already describes the shape: `letter_to_count: Dict[str, int]`
- The value is constructed on the very next line and the construction is obvious

---

## Rule 4: Decompose into named steps

**The test:** If a function does N things in sequence and you can describe each in a short phrase, each phrase is a candidate function name. If the phrase describes real work (not just "call X"), extract it.

This is the complement of Rule 2. Rule 2 says don't extract trivial wrappers. Rule 4 says DO extract when a long function has distinct phases that each do meaningful work. The goal is that a reader can understand the top-level function by reading its body as a sequence of named steps, without needing to understand the internals of each step.

### Signs you should decompose

- A function is 80+ lines with distinct phases separated by blank lines or comments like `# Step 2: ...`
- You find yourself scrolling past one block to understand the next
- The function has nested loops or conditionals where each branch is really a different operation
- A block of code has its own local variables that aren't used by the rest of the function

### The named-step pattern

The top-level function reads like a plan:

```python
def get_eval_dataset(config, *, word_source=None):
    """Build the eval dataset — either uniform-random or balanced by turn."""
    task_cfg = config.get("task", {})
    n_samples = task_cfg.get("eval_samples", 100)
    seed = task_cfg.get("eval_seed", 42)
    exact_turns = task_cfg.get("eval_exact_turns")

    if exact_turns is None:
        return generate_synthetic_wordle_dataset(config, seed=seed, n_samples=n_samples, word_source=word_source)

    turns = _validate_eval_exact_turns(exact_turns)          # parse and validate [1..6]
    rows = _build_balanced_turn_samples(config, turns, seed, n_samples, word_source)  # round-robin per turn
    rows = _fill_remainder(config, rows, turns, seed, n_samples, word_source)         # top up to n_samples
    return Dataset.from_list(rows[:n_samples])
```

Each helper does one thing with a clear name. The top-level function is a readable outline. A reader who wants the details dives into one helper at a time.

### Example: monolithic scoring function split into phases

Before — one 90-line function that parses, validates, scores, and formats:

```python
def _score_constraint_completion(prompt_text, completion_text, valid_set, task_cfg, tokenizer, *, reward_max_output_tokens=None):
    # Parse config values (10 lines)
    rewards_cfg = task_cfg.get("rewards", {})
    format_reward = float(rewards_cfg.get("format", 0.2))
    dict_reward = float(rewards_cfg.get("dict", 0.2))
    # ... 6 more config reads ...

    # Check format and early-return if bad (15 lines)
    completion_tokens = len(tokenizer.encode(completion_text, add_special_tokens=False))
    is_overlength = int(completion_tokens > max_output_tokens)
    guess = parse_strict_guess(completion_text)
    strict_ok = bool(guess and re.fullmatch(r"[a-z]{5}", guess))
    if not strict_ok:
        return { ... 12-key dict ... }

    # Score constraints (20 lines)
    ac = aggregate_constraints(history_rows)
    sat_count, totals, sats = compute_sat_count(guess, ac)
    # ... compute ratios, bonuses ...

    # Build result dict (15 lines)
    return { ... 18-key dict ... }
```

After — the scoring function delegates to named phases:

```python
def _score_constraint_completion(prompt_text, completion_text, valid_set, task_cfg, tokenizer, *, reward_max_output_tokens=None):
    weights = _read_constraint_reward_weights(task_cfg)      # config -> typed weights
    parse_result = _parse_and_validate_guess(completion_text, tokenizer, max_output_tokens)
    if not parse_result.strict_ok:
        return _build_failed_parse_result(parse_result, weights)

    history_rows = _parse_history_from_prompt(prompt_text)
    constraint_score = _evaluate_constraints(parse_result.guess, history_rows, valid_set)
    return _build_scored_result(parse_result, constraint_score, weights, history_rows)
```

Each helper earns its name — it's not a trivial wrapper, it encapsulates a distinct phase with its own logic.

### How Rule 4 interacts with Rule 2

Rule 2 says: don't extract if the function body is shorter than its signature or wraps a single call.

Rule 4 says: DO extract if the parent function has distinct phases that each do real work.

The tiebreaker is always: **does the extraction make the parent function readable as a plan?** If extracting a 3-line block means the parent becomes a clear sequence of named steps, extract it even if the block is short. If extracting just moves code around without making the parent clearer, don't.

Bad decomposition (Rule 2 violation dressed as Rule 4):

```python
def process(config):
    data = _load_data(config)        # just calls load_dataset(config["path"])
    result = _transform(data)        # just calls data.map(fn)
    _save(result)                    # just calls result.to_csv(path)
```

Each "step" is a trivial wrapper. The parent isn't clearer than if you inlined all three.

Good decomposition (Rule 4 applied correctly):

```python
def process(config):
    raw_data = _load_and_validate_input(config)     # load, schema check, dedup, 25 lines
    scored_data = _apply_scoring_pipeline(raw_data)  # multi-pass scoring with rollups, 40 lines
    _publish_results(scored_data, config)            # format, upload, notify, 30 lines
```

Each step does substantial work. The parent reads like an outline. A reader can understand the flow without reading 95 lines of implementation.

---

## How to apply these rules

When refactoring a file:

1. **Map the call graph.** For every function, list who calls it and what it calls. Functions with one caller are candidates for Rule 2. Functions that extract values from a dict and immediately pass them to another function are candidates for Rule 1.

2. **Check external consumers.** Before deleting anything, grep the codebase for imports. If something is imported outside the file, either keep it with a shim comment or update the consumer.

3. **Preserve the public API.** If experiment configs, CLIs, or other modules import specific function names, those names must continue to exist. Refactor internals freely; rename or delete public names only if you update all consumers.

4. **Inline bottom-up.** Start with the lowest-level wrappers (the ones that call one thing and return). Work upward. Each deletion simplifies the layer above it.

5. **Verify after each change.** Run `python3 -c "import ast; ast.parse(open('file.py').read())"` after every edit. Catch syntax errors immediately, not after 10 edits.

---

## When NOT to refactor

- The function is called from 3+ places — even if it's simple, it prevents repetition
- The function hides a non-obvious algorithm — the name serves as documentation
- The function is part of a public API with external consumers you can't update
- The function will gain more callers soon (but be honest — "soon" usually means "never")
- The indirection exists to satisfy a framework contract (e.g., a plugin interface that requires a specific method signature)
