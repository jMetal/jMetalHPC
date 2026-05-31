# TESTING_GUIDELINES.md

How tests are written in jMetalHPC.

> This document is **subordinate to** [`JAVA_HPC_GUIDELINES.md`](./JAVA_HPC_GUIDELINES.md).
> If any rule here conflicts with the HPC guidelines, the HPC guidelines win.

---

## Frameworks

- **JUnit 6.1.0 (Jupiter)** — `org.junit.jupiter:junit-jupiter`.
- **AssertJ 3.26.x** — the only assertion library used. Double-tolerance comparisons (`isCloseTo(..., within(1e-10))`) are the central assertion of this project and JUnit's built-in `assertEquals(double, double, delta)` is too verbose for it.
- **No Mockito, no PowerMock, no Spring Test.** The units under test are pure-math operators, problems and algorithms. If a test seems to need a mock, the production code is over-abstracted — fix the production code, do not introduce mocks.

---

## Test categories

Every test in this project belongs to one of four categories. New code is accepted only when the relevant categories exist for it.

| Category               | Purpose                                                                                                          | Mandatory for                                            |
|------------------------|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| **Numerical equivalence** | Compare output against a fixed-seed jMetal oracle in `reference-snapshots/`. Same seed, same evaluations, same front (within tolerance). | Every new algorithm variant.                             |
| **Property**           | Check invariants that must hold for any input in the legal domain (bounds, sums, monotonicities, partitions).    | Every operator and every utility (sort, crowding, etc.). |
| **Determinism**        | Same seed → same result, byte-for-byte (or within `1e-12`).                                                      | Every algorithm.                                         |
| **Smoke**              | Run a full algorithm with a small population in well under 1 s. Catches integration regressions cheaply.         | Every algorithm.                                         |

Coverage percentage is **not** a target. The two-line summary of acceptance is:

> Every operator has a **property** test. Every algorithm variant has a **numerical-equivalence** test plus a **determinism** test plus a **smoke** test.

---

## Naming and structure

- One test class per production class. Name: `<ClassName>Test`.
- `@DisplayName("Unit tests for <ClassName>")` on the class.
- Group scenarios with `@Nested` classes when there are ≥3 tests sharing the same setup or describing the same operation.
- Method names: **short and behavioural** (`bounds_areRespected_whenMutatingMinimumValue`), not the long given/when/then sentence. Use `@DisplayName` for the human-readable description; keep the method name compact for IDE navigation.
- Inside the test body, use **AAA comment markers** (`// arrange`, `// act`, `// assert`). Combining `// act and assert` is allowed when the act *is* the assertion (e.g. asserting an exception).
- Declare the class under test as an uninitialised field; instantiate it in `@BeforeEach setUp()` when it carries state. For stateless operators, instantiating per test is fine.
- Field initialisers are reserved for **immutable test constants** (`private static final double TOLERANCE = 1e-10;`).
- Fixed seeds are mandatory in every test that exercises randomness. Define them as named constants (`private static final long SEED = 1234567L`).

---

## Assertions (AssertJ patterns)

Doubles — **always with tolerance**:

```java
assertThat(actual).isCloseTo(expected, within(1e-10));
```

Arrays of doubles:

```java
assertThat(actual).usingComparatorWithPrecision(1e-10).containsExactly(expected);
```

Exceptions:

```java
assertThatThrownBy(() -> new SBXCrossover(-0.5, 20.0, random))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("crossoverProbability must be in [0,1]");
```

Bound preservation (the most common operator assertion):

```java
assertThat(child.variables())
    .allSatisfy(value -> assertThat(value)
        .isBetween(problem.lowerBound(0), problem.upperBound(0)));
```

---

## Forbidden in tests

- Mocks of any kind.
- `Thread.sleep` for timing-based assertions.
- Wall-clock comparisons for performance claims — those live in JMH, not in unit tests.
- Loose tolerances (`within(0.1)` for double comparisons of algebraic results). If you need a loose tolerance, justify it in a comment.
- Random seeds taken from `System.currentTimeMillis()` or unseeded `Random()`.

---

## Example: property test for an operator

```java
@DisplayName("Unit tests for PolynomialMutation")
class PolynomialMutationTest {

    private static final long SEED = 1234567L;
    private static final double TOLERANCE = 1e-12;

    private PolynomialMutation mutation;
    private RandomGenerator random;

    @BeforeEach
    void setUp() {
        random = RandomGenerator.of("L64X128MixRandom");
        mutation = new PolynomialMutation(1.0, 20.0, random);
    }

    @Nested
    @DisplayName("Bound preservation")
    class Bounds {

        @Test
        @DisplayName("mutated values stay within problem bounds for ZDT1")
        void bounds_areRespected_onZDT1() {
            // arrange
            DoubleProblem problem = new ZDT1(30);
            DoubleSolution solution = randomSolution(problem, SEED);

            // act
            for (int i = 0; i < 10_000; i++) {
                mutation.mutate(solution, problem);
            }

            // assert
            assertThat(solution.variables())
                .allSatisfy(value -> assertThat(value)
                    .isBetween(problem.lowerBound(0), problem.upperBound(0)));
        }
    }

    @Nested
    @DisplayName("Argument validation")
    class Validation {

        @Test
        @DisplayName("negative distribution index is rejected")
        void negativeDistributionIndex_isRejected() {
            // act and assert
            assertThatThrownBy(() -> new PolynomialMutation(1.0, -1.0, random))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("distributionIndex");
        }
    }
}
```

What this test deliberately does:

- Uses a **real `ZDT1`**, not a mock.
- Pins the **seed** as a named constant.
- Runs the operator **10 000 times** to make the bound-preservation invariant a real probe of corner-rounding bugs rather than a single-shot smoke check.
- Uses AssertJ's `isBetween` for the per-value bound check.

---

## Example: numerical-equivalence test for an algorithm

```java
@DisplayName("Numerical equivalence tests for NSGAII")
class NSGAIIEquivalenceTest {

    private static final long SEED = 1234567L;
    private static final double TOLERANCE = 1e-10;

    @Test
    @DisplayName("NSGAII on ZDT1 reproduces the jMetal snapshot front")
    void nsgaii_onZDT1_reproducesJMetalSnapshot() throws IOException {
        // arrange
        DoubleProblem problem = new ZDT1(30);
        NSGAII algorithm = new NSGAII(problem, 100, 25_000, SEED);
        double[][] expectedFront = Snapshots.load("nsgaii-zdt1-seed-1234567.csv");

        // act
        algorithm.run();
        double[][] actualFront = algorithm.front();

        // assert
        assertThat(actualFront)
            .usingComparatorWithPrecision(TOLERANCE)
            .containsExactlyInAnyOrder(expectedFront);
    }
}
```

The snapshot is produced once from jMetal with the same seed and committed under `reference-snapshots/`. If the snapshot drifts because of a fix in jMetal, the test must be regenerated and the change explained in `experiments/`.

---

## Example: determinism test

```java
@Test
@DisplayName("two runs with the same seed produce identical fronts")
void sameSeed_producesIdenticalFronts() {
    // arrange
    DoubleProblem problem = new ZDT1(30);

    // act
    double[][] firstRun  = new NSGAII(problem, 100, 25_000, SEED).run().front();
    double[][] secondRun = new NSGAII(problem, 100, 25_000, SEED).run().front();

    // assert
    assertThat(secondRun).isDeepEqualTo(firstRun);
}
```

Determinism is checked with **exact** equality, not tolerance. Any nondeterminism (e.g. hidden `HashMap` iteration order in the production code) is a bug.

---

## What lives outside this document

- **JMH benchmarks** are not tests. They live under `src/jmh/java`, follow the JMH conventions described in `JAVA_HPC_GUIDELINES.md` (`@State`, `@Setup`, `Blackhole`), and never run during `mvn test`.
- **Experiment notes** (`experiments/*.md`) record performance comparisons between variants. They are not tests either.
- **`reference-snapshots/`** holds the oracle data produced from jMetal. New snapshots are added with a commit that documents how they were generated (jMetal commit hash, seed, problem parameters).
