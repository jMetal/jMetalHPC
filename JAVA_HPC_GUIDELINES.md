# JAVA_HPC_GUIDELINES.md

Style and implementation guide for jMetalHPC. The goal is **maximum code quality** (meaningful identifiers, Javadoc, rigorous tests) while **prioritising computational performance** over extensibility and reusability.

This document is **prescriptive**: if a rule here conflicts with a common Java enterprise habit, the rule wins.

---

## Part I — Principles

### 1. Priority hierarchy

When two goals compete, this is the order:

1. **Numerical correctness** — output must be equivalent (within tolerance) to the reference implementation in jMetal using the same seed.
2. **Performance** — ops/s, allocations/op, wall-clock time.
3. **Readability** — clear identifiers, Javadoc, code structure.
4. **Extensibility** — ability to add variants without rewriting.
5. **Reusability** — ability to be consumed from outside this project.

This hierarchy is the default answer to any design discussion. If a change improves (3) and hurts (2), it does not go in.

### 2. Do not abstract for hypotheticals

An interface is created when there are **≥2 real implementations**. A base class is extracted when there is **duplication measured in the current code**. No `Solution<T>` "just in case binary problems show up someday", no factories for a single implementation, no GoF patterns without a concrete use case.

### 3. Optimise with numbers, not intuition

No optimisation goes in without a before/after JMH benchmark documented in `experiments/`. "This should be faster" is not an argument.

### 4. Numerical equivalence before speed

If a variant is 3× faster but does not reproduce jMetal's fronts with the same seed and the same number of evaluations, it does not go in. Verification is done against snapshots in `reference-snapshots/`.

---

## Part II — Rules

### Style, naming and Javadoc

- **Identifiers in English, full words.** `populationSize`, not `popSize`; `numberOfObjectives`, not `nObj`; `crowdingDistance`, not `cd`.
- Allowed exceptions: well-established domain conventions (`sbx`, `pm`, `eta`, `f`, `g`, `h` when they come directly from the original paper).
- Loop indices `i, j, k` allowed only in simple, local loops.
- Methods as verbs (`evaluate`, `dominates`, `selectParent`); classes as nouns (`DoubleSolution`, `FastNonDominatedSort`).
- Constants in `SCREAMING_SNAKE_CASE`.
- **Javadoc mandatory** on:
  - Every public class and method.
  - Non-trivial package-private methods.
- `@param`, `@return`, `@throws` always when applicable. Do not "pad" with trivial descriptions.
- In **algorithm and operator** classes, cite the original paper in the class Javadoc: author(s), year, title, DOI or full bibliographic reference.
- Use `@implNote` to document non-obvious HPC decisions. Example:
  ```java
  /**
   * @implNote Uses {@code int[] dominationCount} instead of {@code List<Integer>}
   *           to avoid autoboxing inside the O(M·N²) inner loop.
   */
  ```
- **Do not** document trivial getters/setters or constructors that only assign fields.
- Comments in the code: write them in English. Prefer explaining **why**, not **what** (the code already says what).

### Data structures (the central rule)

- **`double[]` for variables and objectives. Always.** Never `List<Double>`, `Double[]`, `ArrayList<Double>`.
- `int[]` when the size is known or bounded. Avoid `List<Integer>` in hot paths.
- For domination or membership masks, consider a hand-rolled `long[]` (each `long` = 64 bits) before `BitSet`.
- **SoA (struct-of-arrays)** over AoS when the access pattern is columnar (e.g. sort the population by objective `k`). Example: instead of `DoubleSolution[]`, keep `double[][] objectives` indexed by `[solutionIndex][objectiveIndex]` or even `[objectiveIndex][solutionIndex]` depending on access.
- `record` allowed in configuration, results and immutable data outside the hot path. **Forbidden** in hot-path structures: implicit defensive copies of records can add overhead.
- Prefer final classes over hierarchies. Each problem (ZDT1, ZDT2, ZDT3...) is a **flat, independent class**, not extending an `AbstractDoubleProblem`.

### Allocation discipline

- **Zero allocations** inside the generational loop except those strictly necessary (new population, offspring).
- Reuse scratch arrays across calls. Pattern:
  ```java
  private final double[] scratchVariables;  // allocated once in constructor
  ```
- If `DoubleSolution` creation shows up as a hot spot in the flame graph, consider a solution pool.
- **Autoboxing forbidden** in the hot path: `Double`, `Integer`, `Optional<Double>`, boxed `Boolean`.
- Correct initial capacity in `ArrayList`/`HashMap` when used outside the hot path (`new ArrayList<>(populationSize)`).
- Be suspicious of `String.format` and string concatenation on any hot path. If logging is needed, it must be cheap or toggleable.

### JIT-friendly code

- **Small methods.** Aim for **< 325 bytes** of bytecode so that C2 inlines them without question. Verifiable with `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`.
- **Avoid megamorphism** in the hot path. Ideally every call site inside the generational loop is monomorphic. At most two implementations of the same type active at a time.
- `final` on classes and methods when no subclasses are foreseen. Helps the inliner and devirtualisation.
- **No `Comparator.comparing(...)`** or lambdas inside inner loops. Devirtualise by hand using a static method.
- **No streams** in the hot path. Streams in configuration or result aggregation are fine.
- Canonical loop pattern so the JIT can hoist bounds checks:
  ```java
  for (int i = 0, n = a.length; i < n; i++) {
      // a[i] no bounds check after JIT
  }
  ```
- Avoid variable captures in lambdas inside loops (hidden allocations).

### Arithmetic

- `Math.fma(a, b, c)` when applicable. Single rounding and usually faster on modern hardware.
- `x * x` instead of `Math.pow(x, 2.0)`.
- `Math.sqrt(x)` instead of `Math.pow(x, 0.5)`.
- Compare doubles with explicit tolerance. **Never `==`** between doubles.
- `StrictMath` only if bit-for-bit reproducibility across JVMs is needed. In general `Math` is preferable (faster).
- Watch out for `NaN`/`Infinity`: algorithms must have defined behaviour if an evaluation produces them.

### Randomness

- Use `java.util.random.RandomGenerator` (Java 17+). **Inject** the generator via constructors; never use hidden `Math.random()` or `ThreadLocalRandom`.
- Recommended default: `L64X128MixRandom` (high statistical quality, fast). Configurable.
- Every algorithm run must be **reproducible**: explicit seed in the constructor and no global random state.

### Errors and validation

- Validate parameters in **constructors** and JMH `@Setup`. Never in every hot-path call.
- `IllegalArgumentException` with messages that include the received value and the expected constraint:
  ```java
  throw new IllegalArgumentException(
      "populationSize must be > 0, was " + populationSize);
  ```
- **Forbidden** `try/catch` and `Optional` in the hot path.
- **Forbidden** exceptions as control flow.
- Unrecoverable errors: throw and let it crash, do not rescue.

### Visibility and API

- Default visibility: **package-private**. `public` only when needed from another package.
- Constructors take all required parameters. Builders only if there are **≥4 optional parameters** and it is justified.
- **No setters** on hot-path classes (DoubleSolution, problems, operators). Configuration via constructor.
- Utility classes: `final` with a `private` constructor that throws `AssertionError`.
- Avoid exposing internal mutables (`return this.population` without copying is OK here because we prioritise performance, **but document it**).

### Immutability

- `final` fields wherever possible.
- **Solutions are mutable** for performance. Document it in the class Javadoc: "This class is mutable for performance reasons; do not share instances across threads without external synchronisation."
- Configuration (algorithm parameters) and problems: **immutable**.

### Tests

- JUnit 6 (Jupiter) + AssertJ. No Mockito in this project: if you need mocks to test something, you are abstracting too much.
- **Numerical equivalence test** with a fixed seed against jMetal for each new variant. Oracle in `reference-snapshots/`.
- Explicit tolerance in double comparisons:
  ```java
  assertThat(actual).isCloseTo(expected, within(1e-10));
  ```
- Property tests for operators:
  - SBX and polynomial mutation **respect bounds** always.
  - Fast non-dominated sort produces the same total number of solutions as the input.
  - Crowding distance is non-negative and the extremes are `+∞`.
- Fast smoke tests (< 1 s) that run a full NSGA-II with a small population.
- Test names and assertion messages in English.

### Benchmarks (JMH)

- `@State(Scope.Benchmark)` for expensive setup (problem, seed, pre-computed evaluations).
- Minimum configuration:
  ```java
  @Warmup(iterations = 3, time = 2)
  @Measurement(iterations = 5, time = 2)
  @Fork(1)
  ```
- `Blackhole` in every `@Benchmark` to prevent dead code elimination.
- **Every optimised variant must have its sibling benchmark.** "It is faster" is not accepted without numbers.
- Results are versioned in `experiments/YYYY-MM-DD-<short-idea>.md`:
  - Hypothesis.
  - JMH command executed.
  - Baseline vs. variant results table.
  - Conclusion (and if discarded, **why**).

### Profiling

- Allocation diagnosis: `-prof gc` in JMH (`alloc.rate.norm` = bytes/op).
- CPU diagnosis: async-profiler (`-prof async:output=flamegraph`).
- Inlining diagnosis: `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` ad-hoc.
- JFR for long sessions: `-XX:StartFlightRecording=...`.

---

## Explicit anti-patterns (do not do)

- `List<Double>` or `Double[]` for variables or objectives.
- Deep problem hierarchies (`AbstractDoubleProblem → ZDT1 → ZDT2 → ZDT4`). Each problem is flat.
- Chained `Comparator.comparing(...).thenComparing(...)` in hot paths.
- `Stream.of(...)`, `IntStream.range(...).forEach(...)` in hot paths.
- Exceptions as control flow.
- `equals/hashCode` on hot-path classes that end up in `HashSet`/`HashMap`. Prefer `int` indices.
- `Optional` as return type of hot-path methods.
- `System.out.println` in algorithm code. If logging is needed, it must be structured and toggleable.
- Unused variables left in "just in case".

---

## When to use abstraction

So this guide is not read as "anti-abstraction militancy", here are the cases where it **is** justified:

- At the **system edge**: the public API of algorithm and problem. A `DoubleProblem` with `evaluate(DoubleSolution)` is reasonable because there are N real problems implementing it.
- When there are **≥2 real implementations** (not hypothetical) sharing an exact signature.
- To **inject** interchangeable, verified components (e.g. `RandomGenerator`).

Operating rule: **write the concrete class first**. When the second implementation appears, extract the interface from the real code, not from imagined code.

---

## Language

All artefacts in this project — source code, identifiers, comments, Javadoc, tests, commit messages, experiment notes, internal documentation — are written in **English**. The only exception is private chat with the maintainer.
