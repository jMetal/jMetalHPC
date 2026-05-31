# experiments/

Performance experiment notes. One Markdown file per experiment, named:

```
YYYY-MM-DD-<short-idea>.md
```

Each file documents a single performance hypothesis: baseline vs. variant, JMH numbers, and conclusion. Notes are **part of the acceptance evidence** for `perf:` commits — see [`GIT_GUIDELINES.md`](../GIT_GUIDELINES.md).

## File template

```markdown
# 2026-06-01 — int[] domination buffers in FastNonDominatedSort

## Hypothesis

Replacing `List<Integer>` domination buffers with `int[]` removes
autoboxing in the O(M·N²) inner loop and should improve allocation
rate per benchmark op by >50%.

## Baseline

- Class: `FastNonDominatedSort` (commit `abc1234`)
- Implementation summary: `List<Integer>` per individual to track
  who dominates whom.

## Variant

- Class: `FastNonDominatedSortIntArray` (commit `def5678`)
- Implementation summary: single `int[populationSize * populationSize]`
  with a counter per row.

## Method

```bash
mvn -Pjmh package
java -jar target/benchmarks.jar FastNonDominatedSort \
    -wi 3 -i 5 -f 1 -prof gc
```

## Results

| Variant                       | Score (ops/s)    | alloc.rate.norm (B/op) |
|-------------------------------|------------------|------------------------|
| Baseline                      | 1234.5 ± 12.3    | 8 192                  |
| int[] buffers                 | 1801.2 ± 9.7     | 1 024                  |

## Numerical equivalence

`mvn test -Dtest=FastNonDominatedSortEquivalenceTest` passes against
the reference snapshot at `reference-snapshots/zdt1-front-100-25000.csv`.

## Conclusion

Accepted. ~46% throughput gain, 8× less allocation per op, output
identical to the baseline. Merged in commit `def5678`.
```

## Rules

- One experiment per file. If you compare three variants, that is three sections in one file, not three files.
- Include both **JMH numbers and the equivalence-test outcome**. A speed win without numerical equivalence is not a win.
- If the experiment is **negative** (variant rejected), keep the file. Negative results prevent re-investigation.
- Reference the relevant commits by hash so the experiment stays auditable.
