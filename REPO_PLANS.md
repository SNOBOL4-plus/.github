# REPO_PLANS.md — Per-Repo Deep Plans

> Read the section for the repo you are working in.
> These plans were in PLAN.md and have been extracted to keep PLAN.md lean.
> Order: dotnet → jvm → tiny → harness → snocone → org decisions.

---

# SNOBOL4-dotnet — Full Plan

## What This Repo Is

A complete SNOBOL4/SPITBOL implementation in Clojure targeting JVM bytecode.
Full semantic fidelity: pattern engine with backtracking, captures, alternation,
TABLE/ARRAY, GOTO-driven runtime, multi-stage compiler.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-jvm  
**Test runner**: `lein test` (Leiningen 2.12.0, Java 21)  
**Baseline**: 1,896 tests / 4,120 assertions / 0 failures — commit `9cf0af3` (2026-03-10)

## Design Decisions (Immutable)

1. **ALL UPPERCASE keywords.** No case folding.
2. **Single-file engine.** `match.clj` is one `loop/case`. Cannot be split.
3. **Immutable-by-default, mutable-by-atom.** TABLE and ARRAY use `atom`.
4. **Label/body whitespace contract.** Labels flush-left, bodies indented.
5. **INVOKE is the single dispatch point.** Add both lowercase and uppercase entries.
6. **nil means failure; epsilon means empty string.**
7. **`clojure.core/=` inside `operators.clj`.** Bare `=` builds IR lists.
8. **INVOKE args are pre-evaluated.** Never call `EVAL!` on args inside INVOKE.
9. **Two-tier generator discipline.** `rand-*` probabilistic. `gen-*` exhaustive lazy.
10. **Typed pools are canonical fixtures.** `I J K L M N` integers, `S T X Y Z` strings, `P Q R` patterns, `L1 L2` labels.
11. **Two-strategy debugging.** (a) run a probe; (b) read CSNOBOL4/SPITBOL source. Never speculate.

## Tradeoff Prompt — Read Before Every JVM Design Decision

1. **Single-file engine.** `match.clj` is one `loop/case`. `recur` requires all targets in the same function body. Do not refactor.
2. **Immutable-by-default, mutable-by-atom.**
3. **Label/body whitespace contract.** Tests must always indent statement bodies.
4. **INVOKE is the single dispatch point.** Add both lowercase and uppercase entries for every new function.
5. **nil means failure; epsilon means empty string.**
6. **ALL keywords UPPERCASE.**
7. **`clojure.core/=` inside `operators.clj`.** Bare `=` builds IR lists.
8. **INVOKE args are pre-evaluated.** Never call `EVAL!` on args arriving in INVOKE.
9. **Two-tier generator discipline.** `rand-*` probabilistic. `gen-*` exhaustive lazy.
10. **Typed pools are canonical fixtures.**

## Key Semantic Notes

**BREAK vs BREAKX**: `BREAK(cs)` does not retry on backtrack. `BREAKX(cs)` slides one char past each break-char on backtrack.

**FENCE**: `FENCE(P)` commits to P's match; backtracking INTO P blocked. `FENCE()` bare aborts the entire match.

**CONJ** (extension — no reference source): `CONJ(P, Q)` — P determines span, Q is pure assertion. Not in SPITBOL or CSNOBOL4.

**$ vs . capture**: `P $ V` — immediate assign. `P . V` — conditional on full MATCH success. (Both assign immediately — deferred infra pending.)

**Operator precedence** (from v311.sil): `**`(50/50, right-assoc) > `*`/`/` > concat > `+`/`-` > `|`.

## File Map

| File | Responsibility |
|------|----------------|
| `env.clj` | globals, DATATYPE, NAME/SnobolArray deftypes, `$$`/`snobol-set!`, TABLE/ARRAY |
| `primitives.clj` | scanners: LIT$, ANY$, SPAN$, NSPAN$, BREAK$, BREAKX$, POS#, RPOS#, LEN#, TAB#, RTAB#, BOL#, EOL# |
| `match.clj` | MATCH state machine engine + SEARCH/MATCH/FULLMATCH/REPLACE/COLLECT! |
| `patterns.clj` | pattern constructors: ANY, SPAN, NSPAN, BREAK, BREAKX, BOL, EOL, POS, ARBNO, FENCE, ABORT, REM, BAL, CURSOR, CONJ, DEFER |
| `functions.clj` | built-in fns: REPLACE, SIZE, DATA, ASCII, CHAR, REMDR, INTEGER, REAL, STRING, INPUT, ITEM, PROTOTYPE |
| `grammar.clj` | instaparse grammar + parse-statement/parse-expression |
| `emitter.clj` | AST to Clojure IR transform |
| `compiler.clj` | CODE!/CODE: source text to labeled statement table; -INCLUDE preprocessor |
| `operators.clj` | operators, EVAL/EVAL!/INVOKE, comparison primitives |
| `runtime.clj` | RUN: GOTO-driven statement interpreter |
| `core.clj` | thin facade, explicit re-exports of full public API |
| `harness.clj` | Three-oracle diff harness |
| `generator.clj` | Worm test generator: rand-* and gen-* tiers |
| `jvm_codegen.clj` | Stage 23D: ASM-generated JVM `.class` bytecode |
| `transpiler.clj` | Stage 23B: SNOBOL4 IR → Clojure `loop/case` fn |
| `vm.clj` | Stage 23C: flat bytecode stack VM |

## Open Issues

| # | Issue | Status |
|---|-------|--------|
| 1 | CAPTURE-COND (`.`) assigns immediately like `$`; deferred-assign infra not built | Open |
| 2 | ANY(multi-arg) inside EVAL string — ClassCastException | Open |
| 3 | Sprint 23E — inline EVAL! in JVM codegen (arithmetic bottleneck) | **NEXT** |

## Acceleration Architecture (Sprint 23+)

| Stage | What | Status |
|-------|------|--------|
| 23A — EDN cache | Skip grammar+emitter via serialized IR | **DONE** `b30f383` — 22× per-program |
| 23B — Transpiler | SNOBOL4 IR → Clojure `loop/case`; JVM JIT | **DONE** `4ed6b7e` — 3.5–6× |
| 23C — Stack VM | Flat bytecode, 7 opcodes, two-pass compiler | **DONE** `d9e4203` — 2–6× |
| 23D — JVM bytecode gen | ASM-generated `.class`, DynamicClassLoader | **DONE** `c185893` — 7.6×; EVAL! still bottleneck |
| 23E — Inline EVAL! | Emit arith/assign/cmp directly into JVM bytecode | **NEXT** |
| 23F — Compiled pattern engine | Compile pattern objects to Java methods | PLANNED |

---

# SNOBOL4-dotnet — Full Plan

## What This Repo Is

Full SNOBOL4/SPITBOL implementation in C# targeting .NET/MSIL. GOTO-driven runtime,
threaded bytecode execution, MSIL delegate JIT compiler, plugin system (LOAD/UNLOAD),
Windows GUI (Snobol4W.exe).

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-dotnet  
**Test runner**: `dotnet test TestSnobol4/TestSnobol4.csproj -c Release`  
**Baseline**: 1,607 passing / 0 failing — commit `63bd297` (2026-03-10)

```bash
cd /home/claude/SNOBOL4-dotnet
export PATH=$PATH:/usr/local/dotnet
dotnet build Snobol4.sln -c Release -p:EnableWindowsTargeting=true
dotnet test TestSnobol4/TestSnobol4.csproj -c Release
```

## MSIL Emitter — Steps 1–13 (All Complete)

`BuilderEmitMsil.cs` JIT-compiles each statement's expression-level token list into a
`DynamicMethod` / `Func<Executive, int>` delegate. One `CallMsil` opcode invokes the
cached delegate. Delegate return convention: `>= 0` = jump to IP; `-1` = halt; `int.MinValue` = fall through.

| Step | Status |
|------|--------|
| 1–5: Scaffolding, expression emission, var reads/writes, operator coverage | **DONE** |
| 6–10: Init/Finalize inline, delegate signature, fall-through/direct/conditional gotos | **DONE** |
| 11–13: Indirect gotos, collapse execute loop, TRACE hooks | **DONE** |

## Next Step — SNOBOL4-dotnet

**Step 14** — Eliminate `Instruction[]` entirely. Store delegates directly in `Func<Executive, int>[]`.

## Solution Layout

```
Snobol4.Common/
  Builder/
    Builder.cs              ← compile pipeline
    BuilderEmitMsil.cs      ← MSIL delegate JIT compiler (Steps 1–13)
    ThreadedCodeCompiler.cs ← emits Instruction[] from token lists
    Token.cs                ← Token.Type enum + Token class
  Runtime/Execution/
    ThreadedExecuteLoop.cs  ← main dispatch loop
    StatementControl.cs     ← RunExpressionThread()
    Executive.cs            ← partial class root
TestSnobol4/
  MsilEmitterTests.cs       ← MSIL emitter tests (Steps 1–13)
  ThreadedCompilerTests.cs
```

## Known Gaps

| # | Issue | Severity |
|---|-------|----------|
| 1 | Pattern.Bal — hangs under threaded execution | Medium |
| 2 | Deferred expressions in patterns `pos(*A)` — TEST_Pos_009 | Low |
| 3 | TestGoto _DIRECT — CODE() dynamic compilation | Medium |
| 4 | Function.InputOutput — hangs on Linux (hardcoded Windows paths) | Low |

---

# SNOBOL4-tiny — Sprint Plan

## What This Repo Is

A native SNOBOL4 compiler targeting x86-64 ASM (and eventually JVM bytecode + MSIL).
Python front-end (`emit_c_stmt.py`) compiles SNOBOL4 source → C → `cc` → binary.
Pattern engine: Byrd Box α/β/γ/ω model in portable C.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-tiny  
**Sprint 20 in progress.**

## Architecture Decisions (Resolved 2026-03-10)

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Memory model: **Boehm GC** | No ref-counting complexity. GC ptrs flow through SnoVal transparently. |
| D2 | Tree children: **realloc'd dynamic array** | Unbounded arity required (snoExprList, snoExpr3, snoParse). |
| D3 | cstack: **Thread-local** (`__thread MatchState *`) | Future-proof. Matches SNOBOL4-csharp `[ThreadStatic]` design. |
| D4 | Tracing modules: **Full implementation** | doDebug=0/xTrace=0 means zero cost in normal use. |
| D5 | SNOBOL4cython destination: **Own org repo `SNOBOL4-cpython`** | v1 Arena, v2 per-node malloc. Two-commit history preserved. |
| D6 | ByrdBox struct reconciliation: **After Sprint 20** | Breaking risk. Sprint 20 test suite is the safety net. |

## Sprint Plan

| Sprint | Mechanism | Status |
|--------|-----------|--------|
| 0 | α/β/γ/ω skeleton + runtime | ✓ done |
| 1 | LIT, POS, RPOS | ✓ done |
| 2 | CAT (Σ) | ✓ done |
| 3 | ALT (Π) | ✓ done |
| 4 | ASSIGN ($, .) | done |
| 5 | SPAN β | done |
| 6 | BREAK, ANY, NOTANY | done |
| 7 | LEN, TAB, RTAB, REM | done |
| 8 | ARB | done |
| 9 | ARBNO | done |
| 10 | REF (ζ) simple | done |
| 11 | Mutual REF + Shift/Reduce + nPush | done |
| 12 | @cursor + -INCLUDE | done |
| 13 | cstack | done |
| 14–19 | Python front-end, Stage B runtime, DEFINE/APPLY, EVAL/OPSYN | done |
| **20** | **Beautiful.sno runs — self-beautify oracle** | **IN PROGRESS** |
| 21 | DEFINE dispatch (sno_apply → compiled C label) | NEXT |

## Key Structs

```c
typedef enum { SNO_NULL, SNO_STR, SNO_INT, SNO_REAL, SNO_TREE,
               SNO_PATTERN, SNO_ARRAY, SNO_TABLE, SNO_FAIL=10 } SnoType;
typedef struct SnoVal { SnoType type; union {
    char *s; long i; double r; struct Tree *t; void *p;
}; } SnoVal;

typedef struct Tree {
    char *tag; SnoVal val; int n, cap; struct Tree **c;
} Tree;

typedef struct MatchState {
    const char *subject; int pos;
    CEntry *cstack; int cstack_n, cstack_cap;
    int *istack; int itop;
    StackNode *vstack;
} MatchState;
extern __thread MatchState *sno_current_match;
```

## Inc File → C File Mapping (see COMPILAND_REACHABILITY.md for full detail)

| Inc file | C file | Complexity |
|----------|--------|------------|
| `global.inc` | `runtime/global.c` | trivial |
| `case.inc` | `runtime/case.c` | trivial |
| `counter.inc` | `runtime/counter.c` | trivial |
| `stack.inc` | `runtime/stack.c` | trivial |
| `tree.inc` | `runtime/tree.c` | moderate |
| `Gen.inc` | `runtime/gen.c` | moderate |
| `ShiftReduce.inc` | `runtime/shiftreduce.c` | moderate |
| `semantic.inc` | `runtime/semantic.c` | moderate |

---

# Snocone Front-End Plan

## What This Is

A clean, purpose-built Snocone compiler written from scratch targeting our own IR.
Snocone (Andrew Koenig, AT&T Bell Labs, 1985) adds C-like syntactic sugar to SNOBOL4:
`if/else`, `while`, `do/while`, `for`, `procedure`, `struct`, `&&` explicit concatenation,
`#include`. Same semantics as SNOBOL4. Better syntax.

**Reference material**: `SNOBOL4-corpus/programs/snocone/`

## Architecture

```
.sc source → Lexer → tokens → Parser → AST → Code gen → SNOBOL4 IR
```

Every Snocone control structure desugars to labels + gotos.
`procedure` → `DEFINE()` + label. `struct` → `DATA()`. `&&` → blank concatenation.

## Milestones

| Step | What | Dotnet | JVM | SNOBOL4 (corpus) |
|------|------|--------|-----|------------------|
| 0 | Corpus: reference files | ✓ `ab5f629` | ✓ | ✓ |
| 1 | Lexer | ✓ `dfa0e5b` | ✓ `d1dec27` | — |
| 2 | Expression parser: `&&`, `||`, `~`, comparisons, `$`, `.` | ✓ `63bd297` | ✓ `9cf0af3` | — |
| 3 | `if/else` → label/goto pairs | Not started | Not started | — |
| 4–8 | `while`, `for`, `procedure`, `struct`, `#include` | — | — | — |
| 9 | Self-test: compile `snocone.sc`, diff against `snocone.snobol4` | — | — | — |

## Key Semantic Rules

- **`if (e) s1 else s2`** → `e :F(sc_else_N)` / `[s1] :(sc_end_N)` / `sc_else_N [s2]` / `sc_end_N`
- **`while (e) s`** → `sc_top_N e :F(sc_end_N)` / `[s] :(sc_top_N)` / `sc_end_N`
- **`procedure f(a,b; local c,d)`** → `DEFINE('f(a,b)c,d') :(f_end)` / `f [body] :(RETURN)` / `f_end`
- **`&&`** (explicit concat) → blank in generated SNOBOL4
- **`~`** (logical negation) → wrap in `DIFFER()`/`IDENT()`
- Generated labels: `sc_N` prefix, monotonic counter, never reused.

## CANONICAL NOTE — Corpus Is Shared By All Three Platforms

Every `.sno`, `.inc`, and `.spt` file lives in `SNOBOL4-corpus`. All three platforms
(JVM, dotnet, tiny) use and share the corpus. Test programs, library includes, and
`snocone.sno` itself are corpus files — not per-platform duplicates.

`snocone.sno` is written in SNOBOL4, modeled exactly on `beauty.sno`:
patterns-as-parser (no separate lexer), `nPush`/`nInc`/`nTop`/`nPop` counter stack,
`~` = `shift(p,t)`, `&` = `reduce(t,n)`, `Shift()`/`Reduce()` tree building,
`pp()` recursive code generator, `Gen()`/`GenTab()` output primitives.

---

# Org-Level Decisions (Permanent)

1. **Canonical repos are in the org.** Personal repos archived, delete ~April 10, 2026.
2. **All default branches are `main`.**
3. **SNOBOL4-jvm submodule** → `SNOBOL4-plus/SNOBOL4-corpus` at `corpus/lon`.
   **SNOBOL4-dotnet submodule** → `SNOBOL4-plus/SNOBOL4-corpus` at `corpus`.
4. **PyPI publishes from `SNOBOL4-plus/SNOBOL4-python`** via Trusted Publisher (OIDC, no token).
5. **Jeffrey's authorship is preserved.** His commit history is intact throughout.
6. **This org has one plan file: PLAN.md.** No separate per-repo plan files.
7. **SNOBOL4-corpus is the single source of truth** for all `.sno`, `.inc`, `.spt` files.
