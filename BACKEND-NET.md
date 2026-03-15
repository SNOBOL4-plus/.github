# BACKEND-NET.md — .NET Backend Reference (L3)

Full SNOBOL4/SPITBOL in C# targeting .NET/MSIL.
GOTO-driven threaded bytecode runtime, MSIL delegate JIT compiler, plugin system.

*Session state → DOTNET.md. Testing protocol → TESTING.md.*

---

## Solution Layout

```
Snobol4.Common/
  Builder/
    Builder.cs                  compile pipeline
    BuilderEmitMsil.cs          MSIL delegate JIT
    ThreadedCodeCompiler.cs     emits Instruction[] ← net-delegates eliminates this
    Token.cs                    Token.Type enum + Token class
  Runtime/Execution/
    ThreadedExecuteLoop.cs      main dispatch loop
    StatementControl.cs         RunExpressionThread()
    Executive.cs                partial class root
TestSnobol4/
  MsilEmitterTests.cs
  ThreadedCompilerTests.cs
```

---

## Open Issues

| # | Issue | Severity |
|---|-------|----------|
| 1 | Pattern.Bal — hangs under threaded execution | Medium |
| 2 | Deferred expressions `pos(*A)` — TEST_Pos_009 | Low |
| 3 | TestGoto _DIRECT — CODE() dynamic compilation | Medium |
| 4 | Function.InputOutput — Linux (hardcoded Windows paths) | Low |

**CRITICAL build flag:** Always pass `-p:EnableWindowsTargeting=true` on Linux.

---

## Performance

Roslyn → MSIL: Roman numerals 96ms → 7ms (13.7×). Pattern scan 40ms → 4ms (10.3×).

| Benchmark | Phase 9 | Phase 10 |
|-----------|--------:|---------:|
| FuncCallOverhead_3000 | 8.2 ms | **5.0 ms** (−39%) |
| StringConcat_500 | 3.0 ms | **0.4 ms** (−87%) |
| VarAccess_2000 | 81.6 ms | **64.8 ms** (−21%) |

Test baseline: 1,607 passing / 0 failing.
