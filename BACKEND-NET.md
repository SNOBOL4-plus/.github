# BACKEND-NET.md — .NET Backend Reference (L3)

Full SNOBOL4/SPITBOL in C# targeting .NET/MSIL.
GOTO-driven threaded bytecode runtime, MSIL delegate JIT compiler, plugin system.

*Session state → DOTNET.md. Testing protocol → TESTING.md.*

---

## TINY NET Runtime DLL Architecture

The sno2c `-net` backend emits `.il` files that reference two shared DLLs compiled once:

### `Snobol4Run.dll` — runtime internals
- Assembly: `snobol4run`, Class: `Snobol4Run`
- Statement dispatch, keyword state (`&STNO`, `&ANCHOR`, `&TRIM`, `&FULLSCAN`)
- Input/output primitives
- Location: `src/runtime/net/snobol4run.il` → `snobol4run.dll`

### `Snobol4Lib.dll` — SNOBOL4 standard library
- Assembly: `snobol4lib`, Class: `Snobol4Lib`
- All `sno_*` helper functions: arithmetic, string ops, comparisons, pattern primitives
- **Will grow to include all INCLUDES from beauty.sno and the full SNOBOL4 function repertoire**
  (SIZE, DUPL, REPLACE, SUBSTR, UCASE, LCASE, LPAD, RPAD, TRIM, INTEGER, IDENT, DIFFER,
   DATATYPE, GT/LT/GE/LE/EQ/NE, LGT etc. — everything beauty pulls in via `-I`)
- Location: `src/runtime/net/snobol4lib.il` → `snobol4lib.dll`

### Emitted `.il` references both:
```
.assembly extern snobol4run {}
.assembly extern snobol4lib {}
call string [snobol4lib]Snobol4Lib::sno_add(string, string)
```

### Test runner compiles both DLLs once, caches in `CACHE_DIR`:
```bash
ilasm snobol4run.il /dll /output:$CACHE_DIR/snobol4run.dll
ilasm snobol4lib.il /dll /output:$CACHE_DIR/snobol4lib.dll
# then every mono run: MONO_PATH=$CACHE_DIR mono prog.exe
```

**Speed benefit:** helpers no longer inlined into every `.il` — ilasm per-program is ~10x faster.

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
