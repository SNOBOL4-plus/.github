# ARCH.md — SNOBOL4-tiny Architecture Reference

Stable architecture decisions. Updated only when architecture changes.
For sprint status → TINY.md. For session history → SESSIONS_ARCHIVE.md.

---

## Core Execution Model — Statement IS a Byrd Box

```
label:  subject  pattern  =replacement  :S(x) :F(y)
          α         →          γ            γ    ω
```
α=evaluate subject, pattern=Byrd box labeled gotos, γ=success/replacement, ω=failure.
Hot path: pure C gotos. Cold path: longjmp for ABORT/FENCE/errors only.

## Byrd Box Layout

```
┌─────────────────────────┐
│  DATA: cursor, locals,  │
│        captures, ports  │
├─────────────────────────┤
│  CODE: α/β/γ/ω gotos   │
└─────────────────────────┘
DATA section: [ box0.data | box1.data | ... ]
TEXT section: [ box0.code | box1.code | ... ]
```

## Four Techniques for *X (Byrd box instantiation)

| # | Name | Target | Status |
|---|------|--------|--------|
| 1 | Struct-passing | C target, M-BEAUTY-FULL | **CURRENT** |
| 2 | mmap+memcpy+relocate | ASM/native, post-BOOTSTRAP | Future |
| 3 | Iota functions | Flat-model C bridge | Concept |
| 4 | GCC &&label port table | GCC extension | Concept |

**Technique 1 (current):** Each named pattern → `pat_X(pat_X_t **zz, int entry)`.
All locals in typed struct. Child frame = pointer field in parent struct.
calloc on entry==0, dispatch to beta on entry==1.

**Technique 2 (future):** memcpy box → new address, relocate relative jumps
and absolute DATA refs, mprotect TEXT RX. LIFO = discard on backtrack.

## Block Function Execution Model (The New Plan, 2026-03-14)

Every SNOBOL4 statement → C function returning next block address:
```c
typedef block_fn_t (*block_fn_t)(void);
block_fn_t pc = block_START;
while (pc) pc = pc();   // the entire engine
```
`:(L42)` → return block_L42. `*X` static → call block_X. `*X` dynamic → call stored block_fn_t.
CODE()/EVAL() → TCC in-process compile → block_fn_t.

## Bootstrap Strategy (M-BOOTSTRAP)

Stage 0: C sno2c (exists). Stage 1: compile sno2c.sno → sno2c_stage1.
Stage 2: compile sno2c.sno via stage1 → sno2c_stage2.
Verify: diff output of stage1 vs stage2 on beauty.sno → empty = **M-BOOTSTRAP**.
C runtime stays C forever. Front-end (lex/parse/emit) moves to sno2c.sno.

## compiler.sno Strategy (Architecture B)

compiler.sno = beauty.sno grammar + `compile(sno)` replacing `pp(sno)`.
Same parse tree, same Shift/Reduce machinery. Final action emits C Byrd boxes.
Proof: M-BEAUTY-FULL proves tree is correct; compile() just walks it differently.

## Three-Column Generated C Format

```
Col 1  Label   0..17    (4-space indent + label + ":" + pad)
Col 2  Stmt   18..59    (C statement body)
Col 3  Goto   60+       (goto target)
```
Macros: PLG(label, goto), PL(label, goto, stmt), PS(goto, cond), PG(goto).
Shared header emit_pretty.h — included by emit.c and emit_byrd.c.

## Save/Restore in DEFINE Functions

WRONG: C-local var_get/var_set preamble/postamble.
RIGHT: α port saves caller locals into struct; γ/ω ports restore. Byrd box form.

## setjmp Model

Per-function setjmp: ✅ done (emit.c emit_fn()).
Per-statement setjmp: ❌ not done (emit_stmt() — needed for line diagnostics).
Glob-sequence optimization: ❌ not done (one setjmp per unlabeled DEFINE body).
Non-Gimpel DEFINE: ❌ not done (standalone DEFINE statements need own guard).

## SIL Naming (Session 84–85, canonical)

Types: DESCR_t, DTYPE_t. Fields: .v (type), .a (address), .f (flags).
Values: NULVCL, STRVAL(), INTVAL(), FAILDESCR, IS_FAIL_fn().
Vars: NV_GET_fn/NV_SET_fn. Functions: APLY_fn, DEFINE_fn, CONC_fn.
Pattern codes: XCHR (lit), XARBN (arbno), XNME (cond assign), etc.
Known bug: ARRAY_VAL macro uses .a, should be .arr — dormant until ARRAY_VAL called.
