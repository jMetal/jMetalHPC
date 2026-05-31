# reference-snapshots/

Fixed-seed outputs from jMetal used as the **numerical oracle** for equivalence tests in jMetalHPC. See [`TESTING_GUIDELINES.md`](../TESTING_GUIDELINES.md) for the test category that consumes these files.

## Naming convention

```
<algorithm>-<problem>-<populationSize>-<maxEvaluations>-seed-<seed>.csv
```

Examples:

```
nsgaii-zdt1-100-25000-seed-1234567.csv
nsgaii-dtlz2-200-50000-seed-1234567.csv
```

## File format

CSV. One row per non-dominated solution in the final population. One column per objective. No header row. Decimal separator: `.`. Field separator: `,`.

For `nsgaii-zdt1-100-25000-seed-1234567.csv` (2 objectives, ≤100 rows):

```
0.0001234,0.9998765
0.0123456,0.8901234
...
```

## How to generate a snapshot

Snapshots are produced from a checked-out jMetal at a known commit. Document the source in `experiments/` when introducing a new snapshot:

```markdown
## Snapshot: nsgaii-zdt1-100-25000-seed-1234567.csv

- jMetal commit: <hash>
- Algorithm: NSGAII from jmetal-component with default operators
  (SBX pc=0.9 di=20, polynomial mutation pm=1/n di=20)
- Random seed: 1234567L
- Generated with: <command or example class used>
```

## Rules

- **Never regenerate a snapshot to make a failing test pass.** If a test fails, fix the production code. Regeneration is only allowed when the jMetal reference itself fixes a bug — and the change must be documented in `experiments/`.
- Snapshots are committed under the same `feat:` commit that introduces the equivalence test consuming them.
- Keep snapshots small. If a population is larger than ~1 000 solutions, store the front only (Pareto-optimal subset) instead of the full population.
