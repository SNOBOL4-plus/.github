# JVM.md — SNOBOL4-jvm

**Repo:** https://github.com/SNOBOL4-plus/SNOBOL4-jvm  
**What it is:** Full SNOBOL4/SPITBOL implementation in Clojure targeting JVM bytecode. Multi-stage compiler: interpreter → transpiler → stack VM → JVM `.class` bytecode.

---

## Current State

**Active priority:** Sprint 23E — inline EVAL! in JVM codegen (arithmetic bottleneck)  
**HEAD:** `9cf0af3`  
**Test baseline:** 1,896 tests / 4,120 assertions / 0 failures (Snocone Step 2)  
**Test runner:** `lein test`

**Next action:** Implement inline EVAL! in `jvm_codegen.clj` — emit arithmetic/assign/cmp directly into JVM bytecode instead of calling back into the interpreter. This is the last major performance bottleneck.

---

## Session Start Checklist

```bash
cd SNOBOL4-jvm
git log --oneline --since="1 hour ago"   # fallback: -5
lein test                                 # confirm baseline
git show HEAD --stat
```

---

## Design Decisions (Immutable)

1. **ALL UPPERCASE keywords.** No case folding.
2. **Single-file engine.** `match.clj` is one `loop/case`. Cannot be split — `recur` requires all targets in the same function body.
3. **Immutable-by-default, mutable-by-atom.** TABLE and ARRAY use `atom`.
4. **Label/body whitespace contract.** Labels flush-left, bodies indented. Tests must always indent statement bodies.
5. **INVOKE is the single dispatch point.** Add both lowercase and uppercase entries for every new function.
6. **nil means failure; epsilon means empty string.**
7. **`clojure.core/=` inside `operators.clj`.** Bare `=` builds IR lists.
8. **INVOKE args are pre-evaluated.** Never call `EVAL!` on args inside INVOKE.
9. **Two-tier generator discipline.** `rand-*` probabilistic. `gen-*` exhaustive lazy.
10. **Typed pools are canonical fixtures.** `I J K L M N` integers, `S T X Y Z` strings, `P Q R` patterns, `L1 L2` labels.
11. **Two-strategy debugging.** (a) run a probe; (b) read CSNOBOL4/SPITBOL source. Never speculate.

---

## Key Semantic Notes

**BREAK vs BREAKX:** `BREAK(cs)` does not retry on backtrack. `BREAKX(cs)` slides one char past each break-char on backtrack.

**FENCE:** `FENCE(P)` commits to P's match; backtracking INTO P blocked. `FENCE()` bare aborts the entire match.

**$ vs . capture:** `P $ V` — immediate assign. `P . V` — deferred (fires only after full MATCH success). Deferred infra currently assigns immediately — see Open Issues #1.

**Operator precedence** (from v311.sil): `**`(50/50, right-assoc) > `*`/`/` > concat > `+`/`-` > `|`.

---

## Acceleration Architecture (Sprint 23+)

| Stage | What | Status |
|-------|------|--------|
| 23A — EDN cache | Skip grammar+emitter via serialized IR | ✅ `b30f383` — 22× per-program |
| 23B — Transpiler | SNOBOL4 IR → Clojure `loop/case` | ✅ `4ed6b7e` — 3.5–6× |
| 23C — Stack VM | Flat bytecode, 7 opcodes | ✅ `d9e4203` — 2–6× |
| 23D — JVM bytecode | ASM `.class`, DynamicClassLoader | ✅ `c185893` — 7.6× |
| **23E — Inline EVAL!** | Emit arith/assign/cmp directly into JVM bytecode | **NEXT** |
| 23F — Compiled pattern engine | Compile pattern objects to Java methods | PLANNED |

---

## Snocone Progress

| Step | What | Status |
|------|------|--------|
| 0 | Corpus reference files | ✅ `ab5f629` |
| 1 | Lexer | ✅ `d1dec27` |
| 2 | Expression parser (`&&`, `\|\|`, `~`, `$`, `.`) | ✅ `9cf0af3` |
| 3 | `if/else` → label/goto | Not started |
| 4–8 | `while`, `for`, `procedure`, `struct`, `#include` | — |
| 9 | Self-test: compile `snocone.sc`, diff oracle | — |

---

## Open Issues

| # | Issue | Status |
|---|-------|--------|
| 1 | CAPTURE-COND (`.`) assigns immediately — deferred-assign not built | Open |
| 2 | ANY(multi-arg) inside EVAL string — ClassCastException | Open |
| 3 | Sprint 23E — inline EVAL! in JVM codegen | **NEXT** |

---

## File Map

| File | Responsibility |
|------|---------------|
| `env.clj` | globals, DATATYPE, NAME/SnobolArray, TABLE/ARRAY |
| `primitives.clj` | scanners: LIT$, ANY$, SPAN$, NSPAN$, BREAK$, BREAKX$, POS#, etc. |
| `match.clj` | MATCH state machine + SEARCH/MATCH/FULLMATCH/REPLACE/COLLECT! |
| `patterns.clj` | pattern constructors: ANY, SPAN, ARBNO, FENCE, ABORT, BAL, CONJ, DEFER |
| `functions.clj` | built-in fns: REPLACE, SIZE, DATA, ASCII, CHAR, REMDR, etc. |
| `grammar.clj` | instaparse grammar + parse-statement/parse-expression |
| `emitter.clj` | AST → Clojure IR |
| `compiler.clj` | CODE!/CODE: source → labeled statement table; -INCLUDE |
| `operators.clj` | operators, EVAL/EVAL!/INVOKE, comparison primitives |
| `runtime.clj` | RUN: GOTO-driven statement interpreter |
| `jvm_codegen.clj` | Stage 23D: ASM-generated JVM `.class` bytecode |
| `transpiler.clj` | Stage 23B: SNOBOL4 IR → Clojure `loop/case` |
| `vm.clj` | Stage 23C: flat bytecode stack VM |

---

## Test History

| Sprint | Tests | Assertions | Failures |
|--------|------:|----------:|---------:|
| Sprint 13 baseline | 220 | 548 | 0 |
| Sprint 18B | 1,488 | 3,249 | 0 |
| Sprint 19 | 2,017 | 4,375 | 0 |
| Sprint 25E | 2,033 | 4,417 | 0 |
| Snocone Step 2 | **1,896** | **4,120** | **0** |
