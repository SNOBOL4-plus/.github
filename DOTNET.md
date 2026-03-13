# DOTNET.md — SNOBOL4-dotnet

**Repo:** https://github.com/SNOBOL4-plus/SNOBOL4-dotnet  
**What it is:** Full SNOBOL4/SPITBOL implementation in C# targeting .NET/MSIL. GOTO-driven threaded bytecode runtime, MSIL delegate JIT compiler, plugin system (LOAD/UNLOAD), Windows GUI.

---

## Current State

**Active priority:** Step 14 — eliminate `Instruction[]`, store delegates directly in `Func<Executive, int>[]`  
**HEAD:** `63bd297`  
**Test baseline:** 1,607 passing / 0 failing  
**Test runner:** `dotnet test TestSnobol4/TestSnobol4.csproj -c Release`

**Next action:** Implement Step 14 in `ThreadedCodeCompiler.cs` — replace `Instruction[]` storage with direct `Func<Executive, int>[]`. No intermediate instruction objects.

---

## Session Start Checklist

```bash
cd SNOBOL4-dotnet
export PATH=$PATH:/usr/local/dotnet
git log --oneline --since="1 hour ago"   # fallback: -5
dotnet build Snobol4.sln -c Release -p:EnableWindowsTargeting=true
dotnet test TestSnobol4/TestSnobol4.csproj -c Release
```

**CRITICAL:** Always pass `-p:EnableWindowsTargeting=true` on Linux builds.

---

## MSIL Emitter Steps

`BuilderEmitMsil.cs` JIT-compiles each statement's expression token list into a `DynamicMethod` / `Func<Executive, int>` delegate. Delegate return convention: `>= 0` = jump to IP; `-1` = halt; `int.MinValue` = fall through.

| Steps | Status |
|-------|--------|
| 1–5: Scaffolding, expression emission, var reads/writes, operator coverage | ✅ |
| 6–10: Init/Finalize inline, delegate signature, fall-through/direct/conditional gotos | ✅ |
| 11–13: Indirect gotos, collapse execute loop, TRACE hooks | ✅ |
| **14: Eliminate `Instruction[]` — store `Func<Executive, int>[]` directly** | **NEXT** |

---

## Snocone Progress

| Step | Status |
|------|--------|
| 0 | ✅ `ab5f629` |
| 1 | ✅ `dfa0e5b` |
| 2 | ✅ `63bd297` — shunting-yard expression parser, 35 tests |
| 3+ | Not started |

---

## Open Issues

| # | Issue | Severity |
|---|-------|----------|
| 1 | Pattern.Bal — hangs under threaded execution | Medium |
| 2 | Deferred expressions `pos(*A)` — TEST_Pos_009 | Low |
| 3 | TestGoto _DIRECT — CODE() dynamic compilation | Medium |
| 4 | Function.InputOutput — Linux (hardcoded Windows paths) | Low |

---

## Solution Layout

```
Snobol4.Common/
  Builder/
    Builder.cs                  compile pipeline
    BuilderEmitMsil.cs          MSIL delegate JIT (Steps 1–14)
    ThreadedCodeCompiler.cs     emits Instruction[] from token lists
    Token.cs                    Token.Type enum + Token class
  Runtime/Execution/
    ThreadedExecuteLoop.cs      main dispatch loop
    StatementControl.cs         RunExpressionThread()
    Executive.cs                partial class root
TestSnobol4/
  MsilEmitterTests.cs           MSIL emitter tests
  ThreadedCompilerTests.cs
```

---

## Test History

| Milestone | Tests |
|-----------|------:|
| `master` (Roslyn) | 1,271 |
| threaded execution | 1,386 |
| msil-emitter | 1,484 |
| main (merged) | 1,466 |
| Snocone Step 2 | **1,607** |

---

## Performance

**Roslyn → MSIL headline:** Roman numerals 96 ms → 7 ms (13.7×). Pattern scan 40 ms → 4 ms (10.3×).

| Benchmark | Phase 9 | Phase 10 |
|-----------|--------:|---------:|
| FuncCallOverhead_3000 | 8.2 ms | **5.0 ms** (-39%) |
| StringConcat_500 | 3.0 ms | **0.4 ms** (-87%) |
| VarAccess_2000 | 81.6 ms | **64.8 ms** (-21%) |
