# SNOBOL4ever — HQ

SNOBOL4/SPITBOL compilers: one shared frontend (beauty.sno → compiler.sno),
three backends (C native, JVM, .NET). Self-hosting goal: sno2c compiles sno2c.
**Team:** Lon Jones Cherryholmes (arch), Jeffrey Cooper M.D. (DOTNET), Claude Sonnet 4.6 (TINY author).

---

## ⚡ NOW

| | |
|-|-|
| **Active repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-crosscheck` — Sprint A — rung 12 crosscheck tests |
| **HEAD** | `08eabba` |
| **Next action** | Build beauty_full_bin → write 101_comment test → run run_beauty.sh |
| **Invariant** | 106/106 rungs 1–11 must pass before any work |

**Working on TINY → [TINY.md](TINY.md) · JVM → [JVM.md](JVM.md) · .NET → [DOTNET.md](DOTNET.md)**

---

## Milestone Dashboard

| ID | Trigger | Repo | ✓ |
|----|---------|------|---|
| M-SNOC-COMPILES | snoc compiles beauty_core.sno | TINY | ✅ |
| M-REBUS | Rebus round-trip diff empty | TINY | ✅ `bf86b4b` |
| M-COMPILED-BYRD | sno2c emits Byrd boxes, mock_engine only | TINY | ✅ `560c56a` |
| M-CNODE | CNode IR, zero lines >120 chars | TINY | ✅ `ac54bd2` |
| **M-BEAUTY-CORE** | beauty_full_bin self-beautifies (mock stubs) | TINY | ❌ |
| **M-BEAUTY-FULL** | beauty_full_bin self-beautifies (real -I inc/) | TINY | ❌ |
| M-CODE-EVAL | CODE()+EVAL() via TCC → block_fn_t | TINY | ❌ |
| M-BYRD-SPEC | Language-agnostic Byrd box spec, all backends | HQ | ❌ |
| M-SNO2C-SNO | sno2c.sno compiled by C sno2c | TINY | ❌ |
| M-BOOTSTRAP | sno2c_stage1 output = sno2c_stage2 | TINY | ❌ |
| M-JVM-EVAL | JVM inline EVAL! | JVM | ❌ |
| M-NET-DELEGATES | .NET Instruction[] eliminated | DOTNET | ❌ |

---

## Platform Map

| Platform | L2 doc | Sprint | Milestone |
|----------|--------|--------|-----------|
| C native | [TINY.md](TINY.md) | `beauty-crosscheck` | M-BEAUTY-CORE |
| JVM/Clojure | [JVM.md](JVM.md) | `jvm-inline-eval` | M-JVM-EVAL |
| .NET/C# | [DOTNET.md](DOTNET.md) | `net-delegates` | M-NET-DELEGATES |
| Corpus | [CORPUS.md](CORPUS.md) | Stable | — |
| Harness | [HARNESS.md](HARNESS.md) | Stable | — |

---

## L3 Reference — read only what you need

| Concern | File |
|---------|------|
| beauty.sno: TDD protocol, rung 12, probe/monitor/triangulate scripts | [FRONTEND-BEAUTY.md](FRONTEND-BEAUTY.md) |
| sno2c: lex/parse/emit, SIL naming, CNode, artifacts, bootstrap | [FRONTEND-SNO2C.md](FRONTEND-SNO2C.md) |
| Rebus: language rules, translation table, round-trip | [FRONTEND-REBUS.md](FRONTEND-REBUS.md) |
| C backend: Byrd box techniques, block functions, setjmp, arch decisions | [BACKEND-C.md](BACKEND-C.md) |
| JVM backend: design decisions, file map, open issues | [BACKEND-JVM.md](BACKEND-JVM.md) |
| .NET backend: solution layout, open issues, performance | [BACKEND-NET.md](BACKEND-NET.md) |
| Shared arch: Byrd box concept, oracle hierarchy, corpus ladder | [ARCH.md](ARCH.md) |
| Testing: four paradigms, corpus ladder protocol | [TESTING.md](TESTING.md) |
| Mandatory rules: token, identity, artifacts, hierarchy | [RULES.md](RULES.md) |
| Session history | [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md) |
| Runtime patches | [PATCHES.md](PATCHES.md) |
| Background, JCON, keyword tables | [MISC.md](MISC.md) |

---

*PLAN.md = L1 index only. ~3KB max. Edit L2/L3 files, not this one.*
