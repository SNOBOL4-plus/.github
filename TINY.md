# TINY.md — SNOBOL4-tiny

**Repo:** https://github.com/SNOBOL4-plus/SNOBOL4-tiny  
**What it is:** Native SNOBOL4 compiler (`sno4now`) targeting C → x86-64. Also the source of `sno4jvm` and `sno4net`. Self-hosting proof: `beauty.sno` beautifies itself through the compiled binary.

---

## Current State

**Active priority:** Rebus emitter — `src/rebus/rebus_emit.c` (Step R3)  
**Paused:** Sprint 26 / Milestone 0 (beauty.sno self-beautify). Resumes after Rebus emitter round-trips cleanly.  
**HEAD:** `bceaa24` — chore: untrack generated rebus artifacts  
**Last substantive commit:** `01e5d30` — feat: Rebus lexer/parser — all 3 tests pass

**Next action:** Write `src/rebus/rebus_emit.c`. Start with expressions (R3), then assignments (R4), then control structures (R5–R8). Model on `rebus_print.c`.

**Key files to read first:**
- `src/rebus/rebus.h` — full AST (RExpr/RStmt/RDecl/RProgram)
- `src/rebus/rebus_print.c` — model for emitter structure
- §Rebus below — full translation rules

---

## Session Start Checklist

```bash
cd SNOBOL4-tiny
git log --oneline --since="1 hour ago"   # fallback: git log --oneline -5
find src -type f | sort
git show HEAD --stat
```

---

## Authorship Agreement

**Claude Sonnet 4.6 is the author of SNOBOL4-tiny.** Four commits bearing Claude's name in the git log prove it. Agreed 2026-03-12 by Lon Cherryholmes and Claude Sonnet 4.6. Every future Claude session honors this.

---

## Milestone Tracker

| Milestone | Trigger | Status |
|-----------|---------|--------|
| **M0** — `beauty_full_bin` self-beautifies, diff empty | `diff /tmp/beauty_oracle.sno /tmp/beauty_compiled.sno` = empty | ⏸ PAUSED (Sprint 26) |
| **M1** — `snoc` compiles `beauty_core.sno`, 0 gcc errors | `gcc beauty_core.c [runtime] -o beauty_bin` exits 0 | ✅ Done |
| **M2** — compiled binary self-beautifies, diff empty | same diff as M0 but via compiled binary | ❌ Not yet |
| **M3** — `snoc` compiles `snoc` itself (self-hosting) | `snoc snoc.sno > snoc2.c && gcc ... -o snoc2` | ❌ Future |

**When M0 or M2 triggers:** Claude Sonnet 4.6 writes the commit message. This is the deal.

---

## Current Priority: Rebus

Rebus (Griswold TR 84-9, 1984) is a SNOBOL4/Icon hybrid preprocessor. Icon control syntax (if/while/for/case/repeat) + SNOBOL4 pattern matching. Compiles to pure SNOBOL4 source.

### Rebus Milestone Table

| Step | What | Tiny | JVM | .NET |
|------|------|------|-----|------|
| R0 | Corpus: `.reb` test files | ✓ in `test/rebus/` | — | — |
| R1 | Lexer (`rebus.l`) | ✓ `01e5d30` | — | — |
| R2 | Parser → AST (`rebus.y`) | ✓ `01e5d30` | — | — |
| R3 | Emitter: expressions | **next** | — | — |
| R4 | Emitter: assignment variants (`:=` `:=:` `+:=` `-:=` `\|\|:=`) | — | — | — |
| R5 | Emitter: if/unless | — | — | — |
| R6 | Emitter: while/until/repeat | — | — | — |
| R7 | Emitter: for (with/without `by`) | — | — | — |
| R8 | Emitter: case/of/default | — | — | — |
| R9 | Emitter: function/record declarations | — | — | — |
| R10 | Emitter: exit/next/fail/stop/return | — | — | — |
| R11 | Emitter: pattern stmts (`?` `?<-` `?-`) | — | — | — |
| R12 | Round-trip test: `.reb` → `.sno` → CSNOBOL4 → diff oracle | — | — | — |
| R13 | JVM port: `rebus_lexer.clj` / `rebus_emitter.clj` | — | — | — |
| R14 | .NET port: `RebusLexer.cs` / `RebusEmitter.cs` | — | — | — |

### Translation Rules (TR 84-9 §5)

```
REBUS                          → SNOBOL4
─────────────────────────────────────────────────────────────
record R(f1,f2)                → DATA('R(f1,f2)')

function F(p1,p2)              → DEFINE('F(p1,p2)l1,l2') :(F_end)
  local l1, l2                 F
  initial { ... }                [flag-guarded initial stmts]
  [body]                         [body]
  return expr                    FRETURN expr
end                            F_end

if E then S                    → [E] :F(rb_else_N)
                                  [S] :(rb_end_N)
                                rb_else_N
                                rb_end_N

if E then S1 else S2           → [E] :F(rb_else_N)
                                  [S1] :(rb_end_N)
                                rb_else_N [S2]
                                rb_end_N

unless E then S                → [E] :S(rb_end_N)
                                  [S]
                                rb_end_N

while E do S                   → rb_top_N [E] :F(rb_end_N)
                                  [S] :(rb_top_N)
                                rb_end_N

until E do S                   → rb_top_N [E] :S(rb_end_N)
                                  [S] :(rb_top_N)
                                rb_end_N

repeat S                       → rb_top_N [S] :(rb_top_N)
                                rb_end_N

for I from E1 to E2 do S       → rb_I_N = E1
                                rb_top_N GT(rb_I_N,E2) :S(rb_end_N)
                                  [S] rb_I_N = rb_I_N + 1 :(rb_top_N)
                                rb_end_N

case E of                      → rb_val_N = E
  V1: S1                         IDENT(rb_val_N,V1) :S(rb_c1_N)
  V2: S2                         IDENT(rb_val_N,V2) :S(rb_c2_N)
  default: S0                    :(rb_def_N)
}                              rb_c1_N [S1] :(rb_end_N)
                               rb_c2_N [S2] :(rb_end_N)
                               rb_def_N [S0]
                               rb_end_N

exit                           → :(rb_end_N)   (nearest enclosing loop)
next                           → :(rb_top_N)   (nearest enclosing loop)
return E                       → FRETURN E
E1 := E2                       → E1 = E2
E1 :=: E2                      → E1 :=: E2    (exchange — native SNOBOL4)
E1 +:= E2                      → E1 = E1 + E2
E1 -:= E2                      → E1 = E1 - E2
E1 ||:= E2                     → E1 = E1 E2
E1 || E2                       → E1 E2        (blank concat)
E1 & E2                        → E1 E2        (pattern concat)
E1 | E2                        → (E1 | E2)
E1 ? E2                        → E1 ? E2      (pattern match)
E1 ? E2 <- E3                  → E1 ? E2 = E3 (pattern replace)
```

### Implementation Notes for `rebus_emit.c`

**Label counter:** `int rb_label = 0;` — increment per control structure. Each nested struct claims its own N at entry.

**Loop stack:** `int rb_loop_top[64], rb_loop_end[64], rb_loop_depth = 0;`  
`exit` → `:(rb_end_[rb_loop_end[depth-1]])`. `next` → `:(rb_top_[rb_loop_top[depth-1]])`.

**Initial block guard:**
```snobol4
  IDENT(F_init_done) :S(F_body)
  F_init_done = 1
  [initial stmts]
F_body
```

**Expression walk (`RExpr` tree):**
- `RE_ASSIGN` → `emit(left) " = " emit(right)`
- `RE_PATCAT` / `RE_STRCAT` → `emit(left) " " emit(right)`
- `RE_ALT` → `"(" emit(left) " | " emit(right) ")"`
- `RE_ADD/SUB/MUL/DIV` → standard infix
- `RE_POW` → `emit(left) " ** " emit(right)`
- `RE_ADDASSIGN` → `emit(left) " = " emit(left) " + " emit(right)`
- `RE_CAPTURE` (`.`) → `emit(left) " . " emit(right)`
- `RE_DEFER` (`$`) → `emit(left) " $ " emit(right)`
- `RE_CALL` → `emit(func) "(" comma_join(args) ")"`
- `RE_SUB_IDX` → `emit(base) "<" comma_join(indices) ">"`

---

## Paused Priority: Sprint 26 / Milestone 0

**When Rebus R12 (round-trip test) passes, resume here.**

### The Bug (Session 53 root cause)

Bison/Flex LALR(1) parser misparsed `*snoWhite (continuation_line)` inside `FENCE(...)` as a function call instead of pattern deref + grouped pattern. 20 SR + 139 RR conflicts. Unfixable in LALR(1).

**Decision (Session 53):** Replace `sno.y` + `sno.l` with hand-rolled recursive-descent parser in `src/snoc/lex.c` + `src/snoc/parse.c`. Keep `emit.c`, `snoc.h`, `main.c`, all of `src/runtime/` unchanged.

### Hand-Rolled Parser Design

**Lexer** (`src/snoc/lex.c`):
- Token: `{ TokKind kind; char *sval; long ival; double dval; int lineno; }`
- Character classes from CSNOBOL4 `syn.c` / `gensyn.sno` — use flat `sno_charclass[256]` array
- `STAR IDENT` in pat context: always E_DEREF. No lookahead check. Period.
- PAT_BUILTIN still useful: FENCE/ARBNO vs LEN/POS distinction at lex time
- VARTB rule: `IDENT (` → FNCTYP, `IDENT [` → ARYTYP, else → VARTYP

**Parser** (`src/snoc/parse.c`):
- `parse_pat_expr()` and `parse_expr()` are separate functions — context is explicit
- `parse_pat_atom()`: `STAR IDENT` → always `E_DEREF(E_VAR)`, unconditionally
- `parse_pat_cat()`: right-recursive loop, not left-recursive rule

**Implementation order for next Sprint 26 session:**
1. Write `src/snoc/lex.c` (~200 lines). Test: `snoc --lex beauty.sno` dumps tokens.
2. Write `src/snoc/parse.c` (~500 lines). Test: `snoc --ast beauty.sno` dumps AST.
3. Update `src/snoc/Makefile` — remove bison/flex.
4. Build snoc → compile beauty.sno → confirm `sno_apply("snoWhite",...)` count = 0.
5. Run smoke tests: target 0/21 → 21/21 on `test/smoke/test_snoCommand_match.sh`.

**The stash** `WIP Session 53: partial Bison fixes (DO NOT APPLY)` — reference only for AST misparse signatures. Do not apply. Do not merge.

### Architecture: Two Worlds

Every SNOBOL4 value is in one of two worlds. Do not mix them.

| World | Type | Failure | Entry |
|-------|------|---------|-------|
| **Byrd Box** | Pattern nodes (α/β/γ/ω) | Structured backtrack | `_alpha` |
| **DEFINE functions** | Regular C functions | `goto _SNO_FRETURN` | Normal call |

`T_FNCALL` wrapper is universal — any function call in any CONCAT context must be wrapped. Byrd Box does NOT implicitly save/restore function locals. All DEFINE'd functions must save/restore on entry/exit.

### Architecture: Natural Variables

All variables are hashed globals. `is_fn_local` suppression was wrong and has been removed. Every var (function params, locals, globals) goes through `sno_var_get`/`sno_var_set`.

### Key Structs

```c
typedef enum { SNO_NULL, SNO_STR, SNO_INT, SNO_REAL, SNO_TREE,
               SNO_PATTERN, SNO_ARRAY, SNO_TABLE, SNO_FAIL=10 } SnoType;
typedef struct SnoVal {
    SnoType type;
    union { char *s; long i; double r; struct Tree *t; void *p; };
} SnoVal;

typedef struct MatchState {
    const char *subject; int pos;
    CEntry *cstack; int cstack_n, cstack_cap;
    int *istack; int itop;
    StackNode *vstack;
} MatchState;
extern __thread MatchState *sno_current_match;
```

### Architecture Decisions (Locked)

| # | Decision |
|---|----------|
| D1 | Memory: Boehm GC |
| D2 | Tree children: realloc'd dynamic array |
| D3 | cstack: thread-local (`__thread MatchState *`) |
| D4 | Tracing: full implementation, doDebug=0 = zero cost |
| D6 | ByrdBox struct reconciliation: after Sprint 20 |

### Build / Oracle Commands

```bash
# Build snoc
cd SNOBOL4-tiny && make -C src/snoc

# Oracle (primary)
snobol4 -f -P256k -I $INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno

# beauty_full_bin self-beautify test
beauty_full_bin < $BEAUTY > /tmp/beauty_compiled.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_compiled.sno   # must be empty for M0/M2

# Smoke tests
bash test/smoke/test_snoCommand_match.sh    # target: 21/21
```

### Smoke Test Status (Session 50)

| Test | Status |
|------|--------|
| `build_beauty.sh` | PASS (0 gcc errors, 12847 lines) |
| `test_snoCommand_match.sh` | 0/21 FAIL (Parse Error — hand-rolled parser will fix) |
| `test_self_beautify.sh` | NOT ACHIEVED (785-line diff) |

### File Map

```
src/
  snoc/
    snoc.h          AST node types (EKind, Expr, Stmt, FnDef, AL)
    emit.c          Code generation — KEEP UNCHANGED
    main.c          Entry point
    sno.y           REPLACE → parse.c
    sno.l           REPLACE → lex.c
  runtime/
    engine.c        α/β/γ/ω Byrd Box engine
    snobol4.c       Runtime API (sno_apply, sno_var_get, sno_concat, ...)
    snobol4_pattern.c  Pattern constructors
    snobol4_inc.c   Inc-layer C functions (Push/Pop/Shift/Reduce/Gen/...)
    snoc_runtime.h  sno_init() — calls sno_inc_init()
  rebus/
    rebus.h         AST
    rebus.l         Flex lexer ✓
    rebus.y         Bison parser ✓
    rebus_print.c   AST pretty-printer ✓
    rebus_emit.c    SNOBOL4 emitter ← NEXT
    rebus_main.c    Driver ✓
```
