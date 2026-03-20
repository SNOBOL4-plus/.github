# DOTNET.md — snobol4dotnet (L2)

.NET/C# backend: SNOBOL4 → MSIL via GOTO-driven threaded bytecode runtime.

→ Backend reference: [BACKEND-NET.md](BACKEND-NET.md)
→ Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `net-perf-analysis` — re-run BenchmarkSuite2, confirm hotfix wins, fire M-NET-PERF
**HEAD:** `a029cae` D-156
**Invariant:** `dotnet test` → 1873/1876 before any work
**Milestone:** M-NET-PERF ❌ (hotfixes A–D landed; re-run + publish pending)

**⚠ CRITICAL NEXT ACTION — Session D-157:**

```bash
cd snobol4dotnet
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
export PATH=$PATH:/usr/local/dotnet
git log --oneline -3   # verify HEAD = a029cae D-156
dotnet build Snobol4.sln -c Release -p:EnableWindowsTargeting=true
dotnet test TestSnobol4/TestSnobol4.csproj -c Release -p:EnableWindowsTargeting=true  # must be 1873/1876
# Step 1: re-run BenchmarkSuite2 to confirm hotfix wins (A–D)
# Step 2: if ≥1 measurable win confirmed → M-NET-PERF fires
# Step 3: fix cross/@N cursor bug (105/106 → 106/106) → M-NET-CORPUS-RUNGS
```

**CRITICAL:** Always pass `-p:EnableWindowsTargeting=true` on Linux builds.

---

## Last Session Summary

**Session D-156 — hotfixes A–D + build infra:**
- Fix A: INTEGER→INTEGER fast path (zero allocation, InvariantCulture)
- Fix B: RealConversionStrategy InvariantCulture in STRING cases
- Fix C: Function.cs reuses `_reusableArgList` — no per-call List alloc
- Fix D: SystemStack.ExtractArguments O(n²) Insert(0) → O(n) Add+Reverse
- BUILDING.md + build_native.sh committed; .gitignore clean
- `dotnet test` + BenchmarkSuite2 re-run pending (no dotnet SDK in container at session end)

---

## Active Milestones (next 5)

| ID | Status | Notes |
|----|--------|-------|
| M-NET-PERF | ❌ | Hotfixes landed; re-run BenchmarkSuite2 to confirm wins |
| M-NET-CORPUS-RUNGS | ❌ | 105/106; `cross` @N cursor bug remaining |
| M-NET-POLISH | ❌ | 106/106 + diag1 35/35 + benchmark grid |
| M-NET-SNOCONE | ❌ | Snocone self-test |
| M-NET-BOOTSTRAP | ❌ | snobol4-dotnet compiles itself |

Full milestone history → [PLAN.md](PLAN.md)

---

## Performance Baseline (session154, pre-hotfix)

| Benchmark | Mean | Alloc/run |
|-----------|-----:|----------:|
| ArithLoop_1000 | 41.6ms | 1662 KB |
| VarAccess_2000 | 98.0ms | 6282 KB |
| Fibonacci_18 | 237.4ms | 11853 KB |
| MixedWorkload_200 | 220.6ms | 13930 KB |
| FuncCallOverhead_3000 | 19.0ms | 877 KB |

Post-hotfix numbers pending re-run. See `perf/baseline.md` + `perf/profile_session156.md`.

---

## SPITBOL Oracle Rule

When CSNOBOL4 and SPITBOL MINIMAL diverge: **SPITBOL MINIMAL wins.**
Reference: `sbl.min` in `snobol4ever/spitbol-x64`.
Key findings: DATATYPE builtins lowercase; user DATA types ToLowerInvariant;
&UCASE/&LCASE = exactly 26 ASCII letters; @N is 0-based cursor.
