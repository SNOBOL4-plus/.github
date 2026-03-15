# FRONTEND-SNO2C.md — sno2c Compiler Frontend (L3)

`sno2c` is the C-targeting SNOBOL4 compiler frontend: lexer, parser, and emitter.
Takes `.sno` source, produces C code that links against the C runtime.

*Session state → TINY.md. C backend arch → BACKEND-C.md. Rebus language → FRONTEND-REBUS.md.*

---

## Source Layout

```
src/sno2c/
  lex.c           Hand-rolled lexer (~200 lines) — flat sno_charclass[256]
  parse.c         Hand-rolled parser (~500 lines) — parse_expr() + parse_pat_expr()
  emit.c          Statement emitter — DEFINE bodies, goto resolution, setjmp guards
  emit_byrd.c     Pattern emitter — Byrd box C functions (Technique 1)
  emit_cnode.c    Expression IR + pretty-printer (CNode)
  emit_cnode.h    CNode type definitions
  snoc.h / sno2c.h  AST + IR node types
  main.c          Driver
src/runtime/
  snobol4/snobol4.c   Core runtime (SIL execution model)
  snobol4/mock_includes.c
  snobol4/snobol4_pattern.c
  mock_engine.c   Stub engine — beauty_full_bin links this, NOT engine.c
```

## Build

```bash
make -C src/sno2c          # produces src/sno2c/sno2c
```

---

## Hand-Rolled Parser (replaced Bison, Session 53)

Bison had 20 SR + 139 RR conflicts. Root cause: `*snoWhite (continuation)` misparsed
as function call inside `FENCE(...)`. LALR(1) state merging structural — unfixable.

Key invariant: `STAR IDENT` in `parse_pat_atom()` is always `E_DEREF(E_VAR)`.
No lookahead. `*foo (bar)` = concat(deref(foo), grouped(bar)) — two sequential atoms.

Keep: `emit.c`, `snoc.h`, `main.c`, all `src/runtime/`
Replaced: `sno.y` → `parse.c`, `sno.l` → `lex.c`

The stash `WIP Session 53: partial Bison fixes` — reference only. DO NOT APPLY.

---

## CNode IR (emit_cnode.c — M-CNODE ✅ `ac54bd2`)

Problem: `emit_expr`/`emit_pat` were streaming printers — irrevocable decisions,
no lookahead. Long expressions stayed on one line because chain depth was shallow.

Solution (same as beauty.sno's pp/qq split):
- **Build:** `build_expr()`/`build_pat()` → CNode tree. No output.
- **Measure:** `cn_flat_width(n, limit)` — early exit at limit. The "qq" lookahead.
- **Print:** `pp_cnode()` — inline if fits, multiline+indent if not. The "pp" decision.

```c
typedef enum { CN_RAW, CN_CALL, CN_SEQ } CNodeKind;
typedef struct CNode {
    CNodeKind kind;
    const char *text;       // CN_RAW: literal; CN_CALL: fn name
    struct CNode **args; int nargs;
    struct CNode *left, *right;  // CN_SEQ
} CNode;
```

Column budget: 120 chars. Arena allocator per statement — no GC pressure.

Before: `SnoVal _v34 = concat_sv(aply("REPLACE",...),aply("REPLACE",...));` — 340 chars
After: multiline with 4-space indent per arg level.

---

## Three-Column Generated C Format (target for all emit.c output)

```
Col 1  Label   chars  0..17   (4-space indent + label + ":" + padding)
Col 2  Stmt    chars 18..59   (C statement body)
Col 3  Goto    chars 60+      (goto target)
```
Macros: `PLG(label, goto)`, `PL(label, goto, stmt)`, `PS(goto, cond)`, `PG(goto)`.
Shared header `emit_pretty.h` — both `emit.c` and `emit_byrd.c` include it.
Currently: emit_byrd.c uses 3-column. emit.c still uses raw `E(...)` — fix after M-BEAUTY-FULL.

---

## Artifact Snapshot Protocol

End of every session that touches sno2c, emit*.c, or runtime/:
```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp_candidate.c
LAST=$(ls artifacts/beauty_tramp_session*.c 2>/dev/null | sort -V | tail -1)
md5sum $LAST /tmp/beauty_tramp_candidate.c
# CHANGED: cp /tmp/beauty_tramp_candidate.c artifacts/beauty_tramp_sessionN.c
# SAME: update artifacts/README.md "no change" note only
```
`artifacts/README.md` records: session N, date, md5, line count, compile status, active bug.
N = last sessionN in artifacts/ + 1. Check `git log --oneline -- artifacts/` if unsure.

---

## SIL Naming Convention (Session 84–85, canonical)

All names emitted by emit.c, emit_byrd.c, emit_cnode.c:

| Category | Names |
|----------|-------|
| Types | `DESCR_t`, `DTYPE_t` |
| Fields | `.v` (type tag), `.a` (address), `.f` (flags) |
| Values | `NULVCL`, `STRVAL(s)`, `INTVAL(i)`, `FAILDESCR`, `IS_FAIL_fn()` |
| Vars | `NV_GET_fn`, `NV_SET_fn` |
| Functions | `APLY_fn`, `DEFINE_fn`, `CONC_fn`, `FNCEX_fn`, `MAKE_TREE_fn` |
| Stack | `NPUSH_fn`, `NPOP_fn`, `NINC_fn`, `NDEC_fn`, `PUSH_fn`, `POP_fn`, `TOP_fn` |
| Pattern codes | `XCHR` (lit), `XARBN` (arbno), `XNME` (cond `.`), `XDNME`/`XFNME` (imm `$`) |
| Expr IR | `E_MPY` (not MUL), `E_OR` (not ALT), `E_NAM` (not COND), `E_DOL`, `E_FNC` |

Intentional lowercase (domain primitives, no `_fn`):
`eq/ne/lt/le/gt/ge`, `add/sub/mul/divyde/powr/neg`, `ident/differ`, `to_int/to_real`

Known bug: `ARRAY_VAL` macro uses `.a`, should be `.arr` — dormant, fix before use.
(`snobol4.h:399` — one character: `.a =` → `.arr =`)

---

## Paused Sprint: `hand-rolled-parser`

Status: sprint exists in TINY.md history, currently paused.
Resumes after M-BEAUTY-FULL.

Implementation order when sprint resumes:
1. `src/snoc/lex.c` (~200 lines) — `sno_charclass[256]` table
2. `src/snoc/parse.c` (~500 lines) — `parse_expr()` and `parse_pat_expr()` separate
3. Update `src/snoc/Makefile` — remove bison/flex dependencies
4. Build → compile beauty.sno → confirm `sno_apply("snoWhite",...)` count = 0
5. Smoke tests: 0/21 → 21/21

---

## Bootstrap Path (post-M-BEAUTY-FULL)

`sno2c.sno` = rewrite of sno2c front-end in SNOBOL4.
What stays C forever: `snobol4.c`, `mock_includes.c`, `snobol4_pattern.c`, `mock_engine.c`
What moves to SNOBOL4: `lex.c`, `parse.c`, `emit.c`, `emit_byrd.c`, `emit_cnode.c`

Stage 0: C sno2c (exists).
Stage 1: compile `sno2c.sno` via C sno2c → `sno2c_stage1`.
Stage 2: compile `sno2c.sno` via `sno2c_stage1` → `sno2c_stage2`.
Verify: diff output stage1 vs stage2 on beauty.sno → empty = **M-BOOTSTRAP**.
