# jMetalHPC

High-performance implementations of multi-objective metaheuristics in Java.

**Status:** early bootstrap. No algorithm code yet; only project scaffolding, conventions and build configuration. First milestone is a baseline NSGA-II for continuous optimisation, benchmarked against [jMetal](https://github.com/jMetal/jMetal).

## Goals

- Maximum runtime performance (ops/s, allocations/op) on multi-objective metaheuristics, starting with NSGA-II for continuous problems.
- Numerical equivalence with jMetal under the same seed and number of evaluations.
- No dependency on jMetal: implementations are standalone and avoid the abstractions (`List<Double>`, `Solution<T>`) that hurt JIT performance.

## Requirements

- Java 21+
- Maven 3.9+

## Commands

```bash
# Build and run tests
mvn test

# Build the JMH benchmarks jar
mvn -Pjmh package

# Run a JMH benchmark by regex
java -jar target/benchmarks.jar <regex>
```

## Layout

```
src/main/java/org/uma/jmetalhpc/   Production code
src/test/java/                     JUnit 6 + AssertJ tests
src/jmh/java/                      JMH benchmarks (jmh profile)
experiments/                       Performance experiment notes (Markdown)
reference-snapshots/               Fixed-seed jMetal output used as numerical oracle
```

## Conventions

Authoritative documents (read in this order before contributing):

- [`CLAUDE.md`](./CLAUDE.md) — project purpose and constraints (also used by Claude Code).
- [`JAVA_HPC_GUIDELINES.md`](./JAVA_HPC_GUIDELINES.md) — Java style and HPC rules.
- [`GIT_GUIDELINES.md`](./GIT_GUIDELINES.md) — commit conventions and atomicity.
- [`TESTING_GUIDELINES.md`](./TESTING_GUIDELINES.md) — test categories, JUnit 6 + AssertJ patterns.

## License

[MIT](./LICENSE).
