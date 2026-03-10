# SNOBOL4-plus — Performance Benchmarks

Cross-platform performance history. All timings are wall-clock including full
pipeline: parse → compile → execute, unless noted otherwise.

---

## SNOBOL4-dotnet (.NET / C#)

**Platform**: Linux / .NET 10.0.103 / Release build
**Methodology**: 3 warmup + 15 timed runs, median reported
**Date**: 2026-03-07

### Backend Comparison

| Benchmark | Roslyn (master) | Threaded bytecode | MSIL delegates | vs Roslyn |
|-----------|----------------:|------------------:|---------------:|----------:|
| Roman numerals (recursive) | 96 ms | 7.8 ms | 7 ms | **13.7×** |
| Pattern scan (vowel count) | 40 ms | — | 4 ms | **10.3×** |
| String build (500 concat) | 39 ms | 2.4 ms | 18 ms | **2.2×** |
| Counter loop (10,000 iter) | 168 ms | 14.6 ms | 97 ms | **1.7×** |
| Fibonacci(20) recursive | 591 ms | 200.6 ms | 322 ms | **1.8×** |

**Notes**:
- Roslyn baseline: Roslyn C# codegen per-program — 25–100ms startup overhead dominates short programs
- Threaded bytecode: custom `ThreadedExecuteLoop` with pre-resolved dispatch, VarSlotArray
- MSIL delegates: `DynamicMethod` via `ILGenerator`, all GOTO logic absorbed into delegates, zero-overhead hot loop
- Fibonacci sees less improvement because recursive call overhead dominates over compilation cost

### Test Coverage by Branch

| Branch | Tests |
|--------|------:|
| `master` (Roslyn) | 1,271 passing |
| `feature/threaded-execution` | 1,386 passing |
| `feature/post-threaded-dev` | 1,413 passing |
| `feature/msil-emitter` | 1,484 passing |
| `feature/msil-trace` | 1,484 passing |
| `main` (merged) | 1,484 passing |

---

## SNOBOL4-jvm (JVM / Clojure)

**Platform**: Linux / OpenJDK 21 / Leiningen 2.12.0
**Methodology**: `bench-compare-jvm` — 1000 worm corpus programs, median ratio
**Date**: 2026-03-09 (session 13–13d)

### Backend Comparison (vs interpreter baseline)

| Backend | Simple programs | Loop programs | Branch programs | Notes |
|---------|----------------:|--------------:|----------------:|-------|
| Interpreter (runtime.clj) | 1× (baseline) | 1× | 1× | GOTO-driven statement interpreter |
| EDN cache (Stage 23A) | **22×** | **22×** | **22×** | Memoised compile — grammar never runs twice |
| Transpiler (Stage 23B) | 3.5× | 6× | — | SNOBOL4 IR → Clojure `loop/case` fn |
| Stack VM (Stage 23C) | 5.7× | 2.5× | 4× | Flat bytecode, 7 opcodes, two-pass compiler |
| JVM bytecode (Stage 23D) | **7.6×** | 3.8× | 1.7× | ASM-generated `.class`, JVM JIT, `DynamicClassLoader` |

**Notes**:
- All backends produce identical output — validated against 1,000+ worm programs
- JVM bytecode bottleneck: loop overhead entirely in `EVAL!` — Sprint 23E (inline EVAL!) will eliminate it
- Cold-start cumulative speedup (EDN cache + JVM backend): ~190×

### Test Suite History

| Session/Sprint | Tests | Assertions | Failures |
|----------------|------:|-----------:|---------:|
| Sprint 13 (baseline) | 220 | 548 | 0 |
| Sprint 18D | 967 | 2,161 | 0 |
| Sprint 18B (catalog migration) | 1,488 | 3,249 | 0 |
| Session 11 | 1,749 | 3,786 | 0 |
| Session 12b | 1,811 | 3,910 | 0 |
| Session 12c | 1,865 | 4,018 | 0 |
| Sprint 19 (var shadowing fix) | 2,017 | 4,375 | 0 |
| Sprint 25A–25F (OPSYN, I/O, CODE) | 2,033 | 4,417 | 0 |
| **Current baseline (2026-03-09)** | **2,033** | **4,417** | **0** |

---

## SNOBOL4-python (Python + C)

**Date**: 2026-03-02 (version 0.5.0)

| Backend | Speed | Notes |
|---------|-------|-------|
| Pure Python (≤ 0.4.x) | 1× (baseline) | Generator-based engine |
| C / SPIPAT (0.5.0+) | **7–11×** | Phil Budne's SPIPAT engine, CPython extension `sno4py` |

---

## SNOBOL4-csharp (C#)

Benchmarks pending. 263 tests passing as of 2026-03-07.

---

## Cross-Platform Comparison

*To be populated as implementations mature and share common benchmark programs.*

Target benchmark suite: Roman numerals, Fibonacci, string manipulation,
pattern scan, loop arithmetic — programs that can run identically on all platforms.

| Benchmark | dotnet (MSIL) | jvm (bytecode) | Notes |
|-----------|--------------|----------------|-------|
| — | — | — | Pending |

