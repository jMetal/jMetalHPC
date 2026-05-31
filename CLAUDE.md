# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

jMetalHPC is an experimental playground for **high-performance** implementations of multi-objective metaheuristics, starting with **NSGA-II for continuous optimization**. The goal is to iterate on implementations that are as fast as possible (CPU, memory, allocations) while remaining numerically equivalent to jMetal.

The reference implementation used as a behavioural baseline is the `jmetal-component` subproject at:

```
/Users/ajnebro/Softw/jMetal/jMetal/jmetal-component
```

When porting an algorithm, operator, or problem, **read the corresponding class in jMetal first** to mirror the math exactly (e.g. SBX distribution index, polynomial mutation repair, ZDT formulas), then deviate freely in the data layout and control flow to optimise performance.

## Design constraints

- **Standalone**, no Maven dependency on jMetal. Keep it that way — depending on jMetal would re-introduce the abstractions (boxed `Double`, `List<Double>`, `Solution<T>` generics) we want to avoid.
- **Continuous optimisation only** for now. Variables are `double[]`, objectives are `double[]`. Do not introduce a generic `Solution<T>` interface; if/when binary or permutation problems are added, prefer parallel hierarchies over premature generification.
- **Avoid allocation in the hot path**. Operators, ranking, and crowding distance should reuse arrays/objects where possible. Treat per-generation allocation as a regression to investigate.
- **Avoid abstractions that the JIT struggles with**: megamorphic call sites in the inner loop, lambdas inside per-individual loops, `List<Double>`, autoboxing, `Comparator.comparing(...)` chains in tight loops.
- **No `List<Double>` for variables/bounds**. Use `double[]`.

## Tech stack

- Java 21 (`maven.compiler.release=21`).
- Maven build.
- JUnit 6 (Jupiter) + AssertJ for tests.
- JMH for benchmarks (the source set lives under `src/jmh/java`).

## Commands

```bash
# Build + run tests
mvn test

# Single test class
mvn test -Dtest=NSGAIITest

# Single test method
mvn test -Dtest=NSGAIITest#runsOnZDT1

# Build JMH benchmarks jar
mvn -Pjmh package

# Run a JMH benchmark by regex
java -jar target/benchmarks.jar NSGAII -wi 3 -i 5 -f 1
```

## Repository layout

```
src/main/java/org/uma/jmetalhpc/
  solution/       DoubleSolution (double[] vars + double[] objs)
  problem/        DoubleProblem interface + zdt/ZDT1..ZDT6
  operator/       SBXCrossover, PolynomialMutation, BinaryTournamentSelection
  algorithm/      NSGAII (baseline) and optimized variants
  util/           FastNonDominatedSort, CrowdingDistance, PseudoRandom
src/test/java/    JUnit 5 + AssertJ tests (correctness, smoke runs)
src/jmh/java/     JMH benchmarks (e.g. NSGA-II on ZDT1 vs ZDT4)
```

## Architecture notes

- `DoubleSolution` is a plain class wrapping two `double[]` (variables, objectives) plus an `int rank` and `double crowdingDistance`. No interface, no generics — instantiated millions of times so the layout matters.
- `DoubleProblem` is a small interface: `numberOfVariables()`, `numberOfObjectives()`, `lowerBound(i)`, `upperBound(i)`, `evaluate(DoubleSolution)`. ZDT problems implement it directly; no abstract base class that boxes bounds into a `List<Double>`.
- `NSGAII` runs the standard `(P_t ∪ Q_t)` truncation loop: evaluate, fast non-dominated sort, crowding distance per front, fill next population front-by-front, sort the last partially-included front by crowding distance.
- `FastNonDominatedSort` operates on raw arrays (`int[] dominationCount`, `int[][] dominatedBy`) rather than `List<List<S>>` of solutions to keep allocations bounded and cache-friendly.
- `PseudoRandom` is a single source of randomness. Algorithms/operators take a `java.util.random.RandomGenerator` so we can plug in faster generators (e.g. `L64X128MixRandom`) and seed deterministically for tests and JMH.

## Conventions for new variants

When experimenting with an optimised variant of an existing component:

- Keep the baseline class untouched and add a sibling (e.g. `NSGAII` and `NSGAIIArrayBased`). Benchmarks should compare both.
- Document the optimisation idea in a one-line class comment (e.g. "Replaces per-front `ArrayList<S>` with a flat `int[]` front index buffer to avoid GC churn.").
- Verify numerical equivalence against the baseline with a fixed-seed test on at least ZDT1 before declaring the variant correct.
