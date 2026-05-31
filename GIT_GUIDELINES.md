# Git and Commit Guidelines

This project follows the [Conventional Commits](https://www.conventionalcommits.org/) specification.

## Message format

```
<type>: <short imperative description>
```

Use the **imperative present**: `add`, not `added` or `adds`.

## Allowed types

| Type       | When to use                                                         |
|------------|---------------------------------------------------------------------|
| `feat`     | A new feature or public method                                      |
| `fix`      | A bug fix                                                           |
| `perf`     | Performance change with no behavioural change (JMH-measured)        |
| `test`     | Adding or correcting tests or JMH benchmarks (no production code)   |
| `refactor` | Code change with no behavioural change **and** no measurable perf change |
| `docs`     | Documentation only (`README.md`, `CLAUDE.md`, etc.)                 |
| `chore`    | Build config, dependencies, `.gitignore`, etc.                      |

A change that improves performance is **`perf`**, not `refactor`. A `refactor:` commit must be a no-op for both behaviour and measured performance.

### `perf:` commits

Every `perf:` commit must reference an entry under `experiments/YYYY-MM-DD-<short-idea>.md` containing the JMH command, baseline vs. variant numbers, and a conclusion. The reference goes in the commit body:

```
perf: switch fast non-dominated sort to int[] domination buffers

See experiments/2026-05-31-int-array-domination.md
```

## Atomic commits

Each commit must represent **one single logical change**. Guidelines:

- If the commit message needs "and" to describe what it does, split it into two commits.
- Ensure the project builds and all tests pass before committing (e.g., `mvn clean test`).
- Never mix production code changes with test changes in the same commit.
- Never mix code changes with documentation changes in the same commit.

### Carve-outs

- A `perf:` commit **may** include its `experiments/*.md` note and any `reference-snapshots/*` it produced or updated. The note is part of the change's acceptance evidence, not separate documentation.
- A `feat:` commit that introduces a new algorithm variant **may** include the equivalence test that gates its acceptance. The rationale: without the test, the variant cannot be claimed correct, so the two are one logical change.

## Examples

```bash
# Good
git commit -m "feat: add NSGAII baseline for continuous problems"
git commit -m "test: add ZDT1 numerical equivalence test against jMetal"
git commit -m "fix: respect upper bound in polynomial mutation"
git commit -m "perf: replace List<Integer> domination buffers with int[]"
git commit -m "refactor: extract crowding distance into static method"
git commit -m "docs: document scratch array reuse in SBXCrossover"

# Bad — too broad, mixes concerns
git commit -m "add stuff and fix tests and update readme"

# Bad — should be perf, not refactor (measurable speedup)
git commit -m "refactor: use int[] instead of List<Integer> in non-dominated sort"
```
