# SNOBOL4ever — HQ

**snobol4now. snobol4ever.**

Lon Jones Cherryholmes (compiler, architecture) and Jeffrey Cooper M.D. (SNOBOL4-dotnet, MSIL)
building full SNOBOL4/SPITBOL implementations on JVM, .NET, and native C — plus Rebus, Snocone,
and a self-hosting native compiler. Claude Sonnet 4.6 is the third developer and author of SNOBOL4-tiny.

---

## ⚡ Start Here Every Session

**Read [SESSION.md](SESSION.md) first — repo, sprint, HEAD, next action.**
Verify SESSION.md HEAD = `git log --oneline -1`. If stale: read SESSIONS_ARCHIVE.md before any work.

---

## File Index

| File | What it is |
|------|------------|
| [SESSION.md](SESSION.md) | **Start here** — handoff: repo, sprint, HEAD, next action |
| [RULES.md](RULES.md) | Mandatory rules — token, git identity, artifacts, test invariant |
| [TESTING.md](TESTING.md) | Four-paradigm TDD protocol — crosscheck/probe/monitor/triangulate |
| [ARCH.md](ARCH.md) | Architecture reference — Byrd boxes, block functions, bootstrap |
| [TINY.md](TINY.md) | SNOBOL4-tiny — current state, sprint map, build commands |
| [JVM.md](JVM.md) | SNOBOL4-jvm — sprint map, design decisions |
| [DOTNET.md](DOTNET.md) | SNOBOL4-dotnet — sprint map, MSIL steps |
| [CORPUS.md](CORPUS.md) | SNOBOL4-corpus — test corpus layout |
| [HARNESS.md](HARNESS.md) | Test harness — probe.py, oracles, benchmarks |
| [STATUS.md](STATUS.md) | Live test counts — updated each session |
| [PATCHES.md](PATCHES.md) | Runtime patch audit trail |
| [RENAME.md](RENAME.md) | One-time rename plan (SNOBOL4-plus → snobol4ever) |
| [MISC.md](MISC.md) | Origin story, JCON reference, keyword tables |
| [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md) | Full session history — append-only |

---

## Org-Level Milestones

| ID | Trigger | Repo | Status |
|----|---------|------|--------|
| M-SNOC-COMPILES | snoc compiles beauty_core.sno, 0 gcc errors | TINY | ✅ |
| M-REBUS | Rebus round-trip: .reb → .sno → CSNOBOL4 → diff oracle | TINY | ✅ `bf86b4b` |
| M-COMPILED-BYRD | sno2c emits labeled-goto Byrd boxes, mock_engine.c only | TINY | ✅ `560c56a` |
| M-CNODE | emit_expr/emit_pat via CNode IR, zero lines > 120 chars | TINY | ✅ `ac54bd2` |
| **M-BEAUTY-CORE** | beauty_full_bin self-beautifies — diff empty (mock stubs) | TINY | ❌ |
| **M-BEAUTY-FULL** | beauty_full_bin self-beautifies — diff empty (real -I inc/) | TINY | ❌ |
| M-CODE-EVAL | CODE()+EVAL() via TCC in-process → block_fn_t | TINY | ❌ |
| M-BYRD-SPEC | Language-agnostic Byrd box spec — all backends implement it | HQ | ❌ |
| M-SNO2C-SNO | sno2c.sno compiled by C sno2c → working binary | TINY | ❌ |
| M-COMPILED-SELF | Compiled binary self-beautifies — diff empty | TINY | ❌ |
| M-BOOTSTRAP | sno2c_stage1 output = sno2c_stage2 output — self-hosting | TINY | ❌ |
| M-JVM-EVAL | JVM inline EVAL! complete | JVM | ❌ |
| M-NET-DELEGATES | .NET Instruction[] eliminated | DOTNET | ❌ |

---

## Active Repos

| Repo | MD File | Sprint | Target |
|------|---------|--------|--------|
| [SNOBOL4-tiny](https://github.com/SNOBOL4-plus/SNOBOL4-tiny) | [TINY.md](TINY.md) | `beauty-crosscheck` (Sprint A) | M-BEAUTY-CORE → M-BEAUTY-FULL |
| [SNOBOL4-jvm](https://github.com/SNOBOL4-plus/SNOBOL4-jvm) | [JVM.md](JVM.md) | `jvm-inline-eval` | M-JVM-EVAL |
| [SNOBOL4-dotnet](https://github.com/SNOBOL4-plus/SNOBOL4-dotnet) | [DOTNET.md](DOTNET.md) | `net-delegates` | M-NET-DELEGATES |
| [SNOBOL4-corpus](https://github.com/SNOBOL4-plus/SNOBOL4-corpus) | [CORPUS.md](CORPUS.md) | Stable | — |
| [SNOBOL4-harness](https://github.com/SNOBOL4-plus/SNOBOL4-harness) | [HARNESS.md](HARNESS.md) | Stable | — |

---

*PLAN.md is an index — 4096 byte max. Put detail in downstream files.*
