# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)

---

## NOW

**Sprint:** `asm-backend` — Sprint A14: M-ASM-BEAUTIFUL (PIVOT session159)
**HEAD:** `db80921` session164
**Milestone:** M-ASM-CROSSCHECK ✅ session151 → **M-ASM-BEAUTIFUL** (A14, active)

**Session164 — label folds onto instruction (pending-label mechanism):**
- Added pending-label buffer in A() — label survives blank lines and STMT_SEP
- Rule: label on own line only when two labels are consecutive
- `L_sn_0:  GET_VAR S_457` — one line per state throughout program body
- beauty_prog_session164.s: 13664 lines (down 4556 from session159), assembles clean
- 106/106 C crosscheck PASS, 26/26 ASM crosscheck PASS

**⚠ CRITICAL NEXT ACTION:**
Lon reviews `artifacts/asm/beauty_prog_session164.s` → M-ASM-BEAUTIFUL fires.

**Session163 — four-column format complete: label: MACRO args ; comment**
- DOL_SAVE macro: 3 raw instructions → 1 line
- DOL_CAPTURE macro: 9 raw instructions → 1 line
- ALT_ALPHA macro: absorbs trailing jmp lα
- ALT_OMEGA macro: absorbs trailing jmp rα
- All \n\n double-newlines removed (45 instances)
- Every state is one line: `label:  MACRO args ; comment`
- beauty_prog_session163.s: 14448 lines (down 3772 from session159), assembles clean
- 106/106 C crosscheck PASS, 26/26 ASM crosscheck PASS

**⚠ CRITICAL NEXT ACTION:**
Lon reviews `artifacts/asm/beauty_prog_session163.s` → M-ASM-BEAUTIFUL fires.

**HEAD (previous):** `6ed79c5` session162
**Milestone:** M-ASM-CROSSCHECK ✅ session151 → **M-ASM-BEAUTIFUL** (A14, active)

**Session162 — three-column format: label: MACRO args ; comment:**
- Added `ALFC(lbl, comment, fmt, ...)` — folds preceding comment line onto instruction line
- Result: `seq_l26_alpha:  LIT_ALPHA lit_str_6, 2, ... ; LIT α`
- ALT emitter fully uses ALT_SAVE_CURSOR/ALT_RESTORE_CURSOR macros
- beauty_prog_session162.s: 14950 lines (down 3270 from session159), assembles clean
- 106/106 C crosscheck PASS, 26/26 ASM crosscheck PASS

**⚠ CRITICAL NEXT ACTION:**
Lon reviews `artifacts/asm/beauty_prog_session162.s` → M-ASM-BEAUTIFUL fires.

**HEAD (previous):** `0f7f20b` session161
**Milestone:** M-ASM-CROSSCHECK ✅ session151 → M-ASM-BEAUTY (A10, blocked 102-109) → **M-ASM-BEAUTIFUL** (A14, active)

**Session161 — label: MACRO args on one line:**
- Added `ALF(lbl, fmt, ...)` helper — emits `label:  INSTRUCTION args` on one line
- 40 `asmL()+A()` and `asmL()+asmJ()` pairs folded into single `ALF()` calls
- Every Byrd box port: `seq_l26_alpha:  LIT_ALPHA lit_str_6, 2, saved, cursor, ...`
- beauty_prog_session161.s: 15883 lines (was 16421 — 538 more eliminated), assembles clean
- 106/106 C crosscheck PASS, 26/26 ASM crosscheck PASS

**⚠ CRITICAL NEXT ACTION — Sprint A14 (M-ASM-BEAUTIFUL):**
Lon reviews `artifacts/asm/beauty_prog_session161.s` → M-ASM-BEAUTIFUL fires.

**Session160 — M-ASM-BEAUTIFUL: all pattern port macros landed:**
- All primitive emitters replaced with one macro call per port:
  LIT_ALPHA/LIT_BETA, SPAN_ALPHA/SPAN_BETA, BREAK_ALPHA/BREAK_BETA,
  ANY_ALPHA/ANY_BETA, NOTANY_ALPHA/NOTANY_BETA, POS_ALPHA/POS_BETA,
  RPOS_ALPHA/RPOS_BETA, LEN_ALPHA/LEN_BETA, TAB_ALPHA/TAB_BETA,
  RTAB_ALPHA/RTAB_BETA, REM_ALPHA/REM_BETA, SEQ_ALPHA/SEQ_BETA,
  ALT_SAVE_CURSOR/ALT_RESTORE_CURSOR, STORE_RESULT/SAVE_DESCR
- snobol4_asm.mac extended with all port macros (811 lines)
- emit_byrd_asm.c: all raw instruction sequences replaced; each port = 1 emitted line
- Body-only (-asm-body) now emits `%include "snobol4_asm.mac"`
- run_crosscheck_asm.sh: nasm -I src/runtime/asm/ added
- beauty_prog_session160.s: 16421 lines (was 18220 — 1799 eliminated), assembles clean
- 106/106 C crosscheck PASS, 26/26 ASM crosscheck PASS

**⚠ CRITICAL NEXT ACTION — Sprint A14 (M-ASM-BEAUTIFUL):**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3   # verify HEAD = 0f7f20b

apt-get install -y libgc-dev nasm
make -C src/sno2c

mkdir -p /home/snobol4corpus
ln -sf /home/claude/snobol4corpus/crosscheck /home/snobol4corpus/crosscheck
gcc -c src/runtime/asm/snobol4_asm_harness.c -o src/runtime/asm/snobol4_asm_harness.o
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh        # must be 106/106
bash test/crosscheck/run_crosscheck_asm.sh                   # must be 26/26
```

**M-ASM-BEAUTIFUL fires** when Lon reads `beauty_prog_session160.s` and declares it beautiful.

**Session158 — M-ASM-BEAUTY progress — 101_comment PASS:**
- `section .text` before named pattern bodies (was `.data` → segfault → **root cause**)
- Stack alignment: `sub rsp,56` (6 pushes + 56 = 112, 112%16=0 ✅)
- `PROG_END`: explicit pops (not `leave`)
- `E_FNC` → `stmt_apply()` in `prog_emit_expr`
- Case 1 S/F dispatch: expression-only stmts with `:F(label)` now check `is_fail`
- `stmt_set_capture()`: DOL/NAM captures materialised into SNOBOL4 variables
- Pattern capture: `X *PAT . V` → `V='bc'` PASS ✅
- **101_comment PASS ✅** — 102-109 `Parse Error`
- Root cause of Parse Error: `E_OR`/`E_CONC` → NULVCL for named pattern assignments

**⚠ CRITICAL NEXT ACTION — Sprint A10 (M-ASM-BEAUTY):**

**102-109 fail with `Parse Error`** — beauty's `*Parse` named pattern is assigned
using `E_OR` (alternation `|`) and `E_CONC` (concatenation) expressions.
These are currently fallback → NULVCL in `prog_emit_expr`.
Fix: register `pat_alt()` and `pat_concat()` as callable functions `ALT`/`CONCAT`,
add `E_OR` and `E_CONC` cases to `prog_emit_expr` that call `stmt_apply()`.

**File:** `src/sno2c/emit_byrd_asm.c` — `prog_emit_expr()` switch
**File:** `src/runtime/snobol4/snobol4.c` — add `_b_PAT_ALT`, `_b_PAT_CONCAT`, register

**Session151 — M-ASM-CROSSCHECK fires — 26/26 ASM PASS:**
- Per-variable capture buffers: `CaptureVar` registry, `cap_VAR_buf`/`cap_VAR_len` in `.bss`
- `cap_order[]` table in `.data` — harness walks it at `match_success`, one capture per line
- `E_INDR` case added to `emit_asm_node` — `*VAR` indirect pattern reference resolved via named-pattern registry
- `/dev/null` dry-run collection pass: replaces `open_memstream` two-pass; uid counter saved/restored so real pass generates identical labels
- `.asm.ref` convention: capture tests with harness-specific output use `TEST.asm.ref`; `run_crosscheck_asm.sh` prefers `.asm.ref` over `.ref`
- `run_crosscheck_asm.sh`: `extract_subject` now finds subject var from match line first; `build_bare_sno` keeps plain-string assignments when var referenced as `*VAR`
- 106/106 main crosscheck invariant holds; HEAD `3624d9d`

**⚠ CRITICAL NEXT ACTION — Sprint A10 (M-ASM-BEAUTY):**

Session154 state:
- `asm_emit_program()` walks all stmts, emits `main()` with `stmt_*` C-shim calls
- Label scheme: `_L_<alnum_base>_<N>` — N guarantees uniqueness, base aids readability
- `emit_jmp()` handles RETURN/FRETURN/END → `_SNO_END`; stub labels for undefined/computed gotos
- beauty.sno assembles and links cleanly via `-asm`
- Statement-only programs work: `OUTPUT = 'hello'` → correct output ✅
- **beauty.sno hangs**: pattern-match stmts (Case 2) fall through without running the pattern

**Next step — pattern-match stmt execution:**
Case 2 must: (1) get subject string via `stmt_get()`, (2) set `subject_data`/`subject_len_val`/`cursor` globals, (3) call `root_alpha` (Byrd box), (4) on `match_success` → apply replacement + goto S-label; on `match_fail` → goto F-label.
Approach: inline Byrd box + C-shim `match_success`/`match_fail` as ASM labels per stmt.

- `ref_astar_bstar.s`: ASTAR=ARBNO("a"), BSTAR=ARBNO("b") on "aaabb" → `aaabb\n` PASS ✅
- `anbn.s`: 4 sequential named-pattern call sites (2×A_BLOCK + 2×B_BLOCK) on "aabb" → `aabb\n` PASS ✅
- `emit_byrd_asm.c`: `AsmNamedPat` registry + `asm_scan_named_patterns()` pre-pass + `emit_asm_named_ref()` call-site + `emit_asm_named_def()` body emitter; `E_VART` wired in `emit_asm_node`
- Named pattern calling convention: Proebsting §4.5 gate — caller stores γ/ω absolute addresses into `pat_NAME_ret_gamma/omega` (.bss qwords), then `jmp pat_NAME_alpha/beta`; body ends `jmp [pat_NAME_ret_gamma/omega]`. No call stack.
- 106/106 crosscheck invariant confirmed; end-to-end `.sno → sno2c -asm → nasm → ld → run` verified

**⚠ CRITICAL NEXT ACTION — Sprint A9 (M-ASM-CROSSCHECK):**

The crosscheck corpus (`crosscheck/patterns/038_pat_literal.sno` etc.) are full SNOBOL4 programs using `OUTPUT`, variables, `:S(YES)F(NO)` gotos — **not** standalone pattern tests. The ASM backend currently only handles pattern-match nodes; it cannot yet compile full SNOBOL4 statements.

**Sprint A9 is therefore scoped differently than A0–A8:**

The path to M-ASM-CROSSCHECK is NOT "run existing crosscheck suite via -asm" — those tests require the full runtime (OUTPUT, goto, variables). Instead:

**Sprint A9 plan — ASM crosscheck harness:**
1. Write `src/runtime/asm/snobol4_asm_harness.c` — thin C harness:
   - Reads subject string from `argv[1]` (or stdin)
   - Declares `extern` symbols: `cursor`, `subject_data`, `subject_len_val`, `match_success`, `match_fail`
   - Provides `_start`-equivalent in C: initialises slots, calls `root_alpha` via function pointer or inline asm `jmp`
   - On `match_success`: prints matched span `subject[0..cursor]` to stdout, exit 0
   - On `match_fail`: exit 1
2. Update emitter: body-only mode (no `_start`, no `match_success/fail`) — extern the cursor/subject symbols
3. New crosscheck driver: for each `crosscheck/capture/*.sno` and `crosscheck/patterns/*.sno`, extract the pattern + subject, compile body-only `.s`, link with harness, run, diff
4. First target: `038_pat_literal` via harness PASS → grow to 106/106

**Key insight from corpus survey (session148):**
- `crosscheck/patterns/` has `038_pat_literal.sno` through `047_pat_rtab.sno` — pure pattern tests
- `crosscheck/capture/` has `058_capture_dot_immediate.sno` through `062_capture_replacement.sno`
- These are the natural first targets for ASM crosscheck since they exercise only pattern nodes

**Sprint A9 steps:**
1. `snobol4_asm_harness.c` — subject from argv[1], `extern` ASM symbols, C `_start`
2. `emit_byrd_asm.c` body-only mode: `-asm-body` flag, no `_start`/`match_success`/`match_fail`, emit `global root_alpha, root_beta` + `extern cursor, subject_data, subject_len_val`
3. `test/crosscheck/run_crosscheck_asm.sh` — new driver extracting pattern+subject from `.sno`, compiling+linking with harness, diffing output
4. `038_pat_literal` PASS → iterate to all patterns/ + capture/ rungs → M-ASM-CROSSCHECK

**PIVOT (session144):** Abandoned `monitor-scaffold` / `bug7-bomb` in favor of x64 ASM backend.
Rationale: C backend has a fundamental structural problem — named patterns require C functions
with reentrant structs, three-level scoping (`z->field`, `#define`/`#undef`), and `calloc` per
call. x64 ASM eliminates all of this: α/β/γ/ω become real ASM labels, all variables live flat
in `.bss`, named patterns are plain labels with a 2-way `jmp` dispatch. One scope. No structs.

**Architecture (session144):**
```
Frontend (lex/parse)     →     IR (Byrd Box)     →     Backend (emit/interpret)

SNOBOL4 reader                                          C emitter       ← existing, keep
Rebus reader              α/β/γ/ω four-port IR          x64 ASM emitter ← NEW PIVOT TARGET
Snocone reader            (byrd_ir.py / emit_byrd.c)    Interpreter     ← future debug tool
Icon reader
Prolog reader
```
5 frontends × 3 backends = 15 combinations. One IR. One compiler driver.

**Next steps (Sprint A0):**
1. Create `src/sno2c/emit_byrd_asm.c` — skeleton, mirrors emit_byrd.c structure.
2. Add `-asm` flag to `main.c` selecting ASM backend, output `.s` file.
3. NASM syntax, x64 Linux ELF64.
4. Emit null program: assemble (`nasm -f elf64`), link (`ld`), run → exit 0.
5. **M-ASM-HELLO fires** → begin Sprint A1 (LIT node).

---

## Milestone Map

| Milestone | Trigger | Status | Sprint |
|-----------|---------|--------|--------|
| **M-ASM-HELLO** | null.s assembles, links, runs → exit 0 | ✅ session145 | A0 |
| **M-ASM-LIT** | LIT node: lit_hello.s PASS | ✅ session146 | A1 |
| **M-ASM-SEQ** | SEQ/POS/RPOS: cat_pos_lit_rpos.s PASS | ✅ session146 | A2–A3 |
| **M-ASM-ALT** | ALT: alt_first/second/fail PASS | ✅ session147 | A4 |
| **M-ASM-ARBNO** | ARBNO: arbno_match/empty/fail PASS | ✅ session147 | A5 |
| **M-ASM-CHARSET** | ANY/NOTANY/SPAN/BREAK PASS | ✅ session147 | A6 |
| **M-ASM-ASSIGN** | $ capture: assign_lit/digits PASS | ✅ session148 | A7 |
| **M-ASM-NAMED** | Named patterns: ref_astar_bstar/anbn PASS | ✅ session148 | A8 |
| **M-ASM-CROSSCHECK** | 26/26 ASM crosscheck PASS | ✅ session151 | A9 |
| **M-ASM-BEAUTY** | beauty.sno self-beautifies via ASM backend | ❌ | A10 |
| **M-ASM-READABLE** | Label names: keep alnum, expand special chars to names (pp_>= → _L_pp_GT_EQ_N). Injective because named tokens are unique and _ delimits. Lon's idea session152. | ❌ | A11 |
| M-BOOTSTRAP | sno2c_stage1 output = sno2c_stage2 | ❌ | final goal |



**ASM backend design (session144):**

Why ASM solves the C structural problem:
- C named patterns require functions with reentrant structs (`pat_X_t *z`), `calloc` per call,
  three-level scoping (`z->field` + `#define`/`#undef` aliases), and `open_memstream` two-pass
  declaration collection. Bug5/Bug6/Bug7 all trace back to this complexity.
- x64 ASM: α/β/γ/ω become real labels. All variables are flat `.bss` qwords declared once at
  top of file. Named patterns are plain labels with a 2-instruction entry dispatch. One scope.
  No structs. No malloc. No scoping tricks.

**Sprint detail:**

| Sprint | What | Key oracle |
|--------|------|-----------|
| A0 | Skeleton + `-asm` flag + null program | `test/sprint0/null.s` |
| A1 | LIT node — inline byte compare | `test/sprint1/lit_hello.s` |
| A2 | POS / RPOS — pure compare, no save | `test/sprint2/pos0_rpos0.s` |
| A3 | SEQ (CAT) — wire α/β/γ/ω between nodes | `test/sprint2/cat_pos_lit_rpos.s` |
| A4 | ALT — left/right arms + backtrack | `test/sprint3/alt_*.s` |
| A5 | ARBNO — depth counter + cursor stack in `.bss` | `test/sprint5/arbno_*.s` |
| A6 | Charset: ANY/NOTANY/SPAN/BREAK — inline scan | corpus rungs |
| A7 | $ capture — span into flat `.bss` buffer | `test/sprint4/assign_*.s` |
| A8 | Named patterns — flat labels, 2-way jmp dispatch | `test/sprint6/ref_*.s` |
| A9 | Full crosscheck 106/106 via ASM backend | crosscheck suite |
| A10 | beauty.sno → ASM → self-beautify | M-ASM-BEAUTY |
| A11 | Label named expansion: pp_>= → L_pp_GT_EQ_N | M-ASM-READABLE |
| A12 | NASM macro library snobol4_asm.mac; emit uses macros; 3-column .s | M-ASM-MACROS |
| A13 | ASM IR phase (CNode-equivalent); separate tree walk from emit | M-ASM-IR |
| A14 | Generated .s as readable as generated .c | M-ASM-BEAUTIFUL |

**Build commands (ASM backend):**
```bash
cd /home/claude/snobol4x
# Install NASM once:
apt-get install -y nasm
# Compile a .sno to .s:
src/sno2c/sno2c -asm myprog.sno > myprog.s
# Assemble + link:
nasm -f elf64 myprog.s -o myprog.o
ld myprog.o src/runtime/snobol4/snobol4_asm.o -o myprog
# Run:
./myprog
```


---

## Confirmed Passing (session116 WIP)

- 101_comment ✅
- 102_output  ✅
- 103_assign  ✅
- 104_label   ✅ (WIP binary)
- 105_goto    ✅ (WIP binary)
- 106/106 rungs 1–11 ✅

---

## Bug History

**Bug7 — ACTIVE:** Ghost frame from Expr17 FENCE arm 1 (nPush without nPop on ω).
**Also check Expr15:** FENCE(nPush() *Expr16 ... nPop() | epsilon) same issue.
**Bug6a — FIXED in WIP (session115):** `:` lookahead guard in pat_X4 cat_r_168.
**Bug6b — FIXED in WIP (session115):** NV_SET_fn for Brackets/SorF; CONCAT_fn Reduce type.
**Bug5 — FIXED in WIP (session114); emit_byrd.c port IN PROGRESS (session116).**
**Bugs 3/4 — FIXED `4c2ad68`.**

---

## Frontend × Backend Frontier

| Frontend | C backend | x64 ASM | .NET MSIL | JVM bytecodes |
|----------|:---------:|:-------:|:---------:|:-------------:|
| SNOBOL4/SPITBOL | ⏳ Sprint A | — | — | — |
| Rebus | ✅ M-REBUS | — | — | — |
| Snocone | — | — | — | — |
| Tiny-ICON | — | — | — | — |
| Tiny-Prolog | — | — | — | — |

✅ milestone fired · ⏳ active · — planned

---

## M-BEAUTY-CORE Sprint Plan

### What beauty.sno does (essential model)

One big PATTERN matches the entire source. Immediate assignments (`$`) orchestrate
two stacks simultaneously during the match:

**Counter stack** — tracks children per syntactic level:
```
nPush()                  push 0       entering a level
nInc()                   top++        one more child recognized
Reduce(type, ntop())     read count   build tree node — fires BEFORE nPop
nPop()                   pop          exit the level — fires AFTER Reduce
```

**Value stack:**
```
shift(p,t)   pattern constructor — builds p . thx . *Shift('t', thx)
reduce(t,n)  pattern constructor — builds '' . *Reduce(t,n)
Shift(t,v)   match-time worker — push leaf node
Reduce(t,n)  match-time worker — pop n nodes, push internal node
~ is opsyn for shift · & is opsyn for reduce
```

**Invariant:** every `nPush()` must have exactly one `nPop()` on EVERY exit path —
success (γ) AND failure (ω). Missing `nPop` on FENCE backtrack = ghost frame.

### Bug7 — Active

`Expr17` arm1: `FENCE(nPush() $'(' *Expr ... nPop() | *Id ~ 'Id' | ...)`
→ nPush fires, `$'('` fails, FENCE backtracks to arm2 — **nPop SKIPPED**

`Expr15`: `FENCE(nPush() *Expr16 (...) nPop() | '')`
→ same issue when no `[` follows

**Fix location:** `emit_byrd.c` — emit `NPOP_fn()` on ω path of nPush arm.

### Skeleton ladder (Sprint steps)

Build minimal SNOBOL4 test programs, each a strict superset of previous.
Diff oracle vs compiled stderr traces. First diverging SEQ#### line = bug.

**All 5 instrumented primitives share `int _nseq` counter:**
```
SEQ0001 NPUSH depth=N top=N    <- snobol4.c NPUSH_fn
SEQ0002 NINC  depth=N top=N    <- snobol4.c NINC_fn
SEQ0003 NPOP  depth=N top=N    <- snobol4.c NPOP_fn
SEQ0004 SHIFT type=T val='V'   <- mock_includes.c Shift()
SEQ0005 REDUCE type=T n=N      <- mock_includes.c Reduce()
```

| Step | Input | Status |
|------|-------|--------|
| `micro0_skeleton.sno` | `N` | ✅ Bug7 does NOT fire — baseline |
| `micro1_concat.sno` | `N + 1` | Bug7 FIRES — next |
| `micro2_call.sno` | `GT(N,3)` | Expr17 arm2/3 — TODO |
| `micro3_grouped.sno` | `(N+1)` | Expr17 arm1 full path — TODO |
| `micro4_full.sno` | `109_multi.input` | Full 5-line program — TODO |

### In-PATTERN Bomb Technique

Place diagnostic calls **directly inside a PATTERN** at any edge using `'' . *fn()`.
The function fires exactly when the match engine reaches that point, including on backtrack.

```snobol4
* Sequence stamp at any pattern edge
        DEFINE('seq_(label)', 'seq_B')          :(seq_End)
seq_B   seqN = seqN + 1
        OUTPUT = 'SEQ' LPAD(seqN,4,'0') ' ' label
        seq_ = .dummy                           :(NRETURN)
seq_End

* Embed at FENCE edges to see exactly which path fires:
        Expr17 = FENCE(
+                   '' . *seq_('E17_arm1_enter')
+                   nPush()
+                   $'('
+                   '' . *seq_('E17_arm1_after_paren')   <- never fires if ( fails
+                   nPop()
+                |  '' . *seq_('E17_arm2_enter')         <- fires on backtrack
+                   *Id ~ 'Id'
+                )
```

**Bomb variant** — abort on wrong state:
```snobol4
        DEFINE('assertDepth(expected)', 'assertB') :(assertEnd)
assertB EQ(_ntop, expected)                        :S(RETURN)
        OUTPUT = '*** BOMB depth=' _ntop ' expected=' expected
        &STLIMIT = 0                               * force abort
assertEnd
```
Place `'' . *assertDepth(1)` immediately after `nPush()` in arm1 to confirm
depth is correct before `$'('` runs.

### Crosscheck ladder (one at a time, never skip)

```
104_label → 105_goto → 109_multi → 120_real_prog → 130_inc_file → 140_self
```
`140_self` PASS → **M-BEAUTY-CORE fires**.

### Diagnostic tools

- **&STLIMIT binary search** — set limit, halve on hang
- **&STCOUNT** — increments correctly on CSNOBOL4 (verified 2026-03-16)
- **TRACE:** `TRACE('var','VALUE')` works; `TRACE(...,'KEYWORD')` non-functional
- **DUMP():** full variable dump at any point

---

## Session Start (session165)

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3   # verify HEAD = db80921

apt-get install -y libgc-dev nasm && make -C src/sno2c

mkdir -p /home/snobol4corpus
ln -sf /home/claude/snobol4corpus/crosscheck /home/snobol4corpus/crosscheck
gcc -c src/runtime/asm/snobol4_asm_harness.c -o src/runtime/asm/snobol4_asm_harness.o
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh        # must be 106/106
bash test/crosscheck/run_crosscheck_asm.sh                   # must be 26/26
```

## Build beauty_full_bin

```bash
RT=src/runtime
INC=/home/claude/snobol4corpus/programs/inc
BEAUTY=/home/claude/snobol4corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > beauty_full.c
gcc -O0 -g beauty_full.c \
    $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o beauty_full_bin
```

## Session End

⛔ **ARTIFACTS FIRST — before any HQ update:**
```bash
# 1. Archive any new .s files that fired a milestone:
#    cp <generated>.s snobol4x/artifacts/asm/<name>.s
#    Update artifacts/README.md with entry (status, milestone, assemble cmd, design notes)
#    git add artifacts/ && git commit -m "sessionN: archive <sprint> oracle .s files"
#
# 2. Then update TINY.md HEAD, sprint status, next action
# 3. Then update PLAN.md milestone dashboard
# 4. Push all repos, .github last
```

```bash
# Artifact check — see IMPL-SNO2C.md §Artifact Snapshot Protocol
# Update this file: HEAD, frontier table, next action, pivot log
git add -A && git commit && git push
# Push .github last
```

---

## Milestones

| ID | Trigger | ✓ |
|----|---------|---|
| M-SNOC-COMPILES | snoc compiles beauty_core.sno | ✅ |
| M-REBUS | Rebus round-trip diff empty | ✅ `bf86b4b` |
| M-COMPILED-BYRD | sno2c emits Byrd boxes, mock_engine only | ✅ `560c56a` |
| M-CNODE | CNode IR, zero lines >120 chars | ✅ `ac54bd2` |
| **M-STACK-TRACE** | oracle_stack.txt == compiled_stack.txt for all rung-12 inputs | ✅ session119 |
| **M-BEAUTY-CORE** | beauty_full_bin self-beautifies (mock stubs) | ❌ |
| **M-BEAUTY-FULL** | beauty_full_bin self-beautifies (real -I inc/) | ❌ |
| M-CODE-EVAL | CODE()+EVAL() via TCC → block_fn_t | ❌ |
| M-SNO2C-SNO | sno2c.sno compiled by C sno2c | ❌ |
| M-COMPILED-SELF | Compiled binary self-beautifies | ❌ |
| M-BOOTSTRAP | sno2c_stage1 output = sno2c_stage2 | ❌ |

---

## Sprint Map

### Active → M-BEAUTY-FULL (SNOBOL4 × C)

| Sprint | Paradigm | Trigger | Status |
|--------|----------|---------|--------|
| `stack-trace` | Dual-stack instrumentation | oracle == compiled stack trace → **M-STACK-TRACE** | ✅ session119 |
| `bug7-bomb` | Bomb protocol → fix emit_byrd.c | trace diff clean + 109_multi PASS → ladder → **M-BEAUTY-CORE** | ⏳ NOW |
| `beauty-probe` | Probe | All failures diagnosed | ❌ B |
| `beauty-monitor` | Monitor | Trace streams match | ❌ C |
| `beauty-triangulate` | Triangulate | Empty diff → **M-BEAUTY-FULL** | ❌ D |

### Planned → M-BOOTSTRAP (SNOBOL4 × C, self-hosting)

| Sprint | Gates on |
|--------|----------|
| `trampoline` · `stmt-fn` · `block-fn` · `pattern-block` | M-BEAUTY-FULL |
| `code-eval` (TCC) · `compiler-pattern` (compiler.sno) | M-BEAUTY-FULL |
| `bootstrap-stage1` · `bootstrap-stage2` | M-SNO2C-SNO |

### Sprint A12 — M-ASM-MACROS

**Goal:** Generated `.s` is readable. Every emitted line follows:

```
LABEL          ACTION          GOTO
```

Three columns. No exceptions. The LABEL is a Byrd box port or SNOBOL4 label.
The ACTION is a NASM macro. The GOTO is the succeed or fail target — a label, never a raw address.

**NASM macro library: `src/runtime/asm/snobol4_asm.mac`**

One macro per Byrd box primitive. Each macro expands to whatever register
shuffling is needed, but the call site is always one readable line:

```nasm
; Pattern nodes — one line each:
P_12_α         SPAN            letter_cs,   P_12_γ,  P_12_ω
P_14_α         LIT             "hello",     P_14_γ,  P_14_ω
P_16_α         SEQ             P_14, P_12,  P_16_γ,  P_16_ω
P_18_α         ALT             P_14, P_16,  P_18_γ,  P_18_ω
P_20_α         DOL             ppTokName,   P_20_γ,  P_20_ω

; Statement — subject, match, replace, goto:
L_LOOP         SUBJECT         ppLine
               MATCH_PAT       P_16,        L_WRITE, L_END
L_WRITE        REPLACE         ppOut,       ppLine
               GOTO                         L_LOOP
```

Parallel C output for comparison:

```c
L_LOOP:   subj = GET("ppLine");
          if (MATCH(P_16, subj)) { SET("ppOut", subj); goto L_WRITE; }
          goto L_END;
L_WRITE:  SET("ppLine", GET("ppOut"));
          goto L_LOOP;
```

**Sprint A12 steps:**
1. Write `src/runtime/asm/snobol4_asm.mac` — macros for LIT/SPAN/SEQ/ALT/ALT/DOL/ARBNO/ANY/NOTANY/BREAK/POS/RPOS/REM/ARB/SUBJECT/MATCH_PAT/REPLACE/GOTO/GOTO_S/GOTO_F
2. Change `emit_byrd_asm.c` to `%include "snobol4_asm.mac"` at top of every `.s`
3. Change every `A("    mov rax...")` emission to `A("  MACRO_NAME  args")` 
4. Verify beauty_prog.s assembles clean with macros expanded
5. Diff generated .s before/after — three-column structure visible throughout
6. **M-ASM-MACROS fires** when beauty_prog.s is fully macro-driven and assembles

### Sprint A13 — M-ASM-IR

**Goal:** Separate the tree walk from code generation. Same architecture as C backend's CNode IR.

The ASM emitter currently does parse → emit in one pass. This makes it hard to:
- Inject comments and separators
- Optimise label names
- Share structure between C and ASM emitters

**Architecture:**
```
Parse → EXPR_t/STMT_t → [ASM IR walk] → AsmNode tree → [ASM emit] → .s file
```

The AsmNode tree is a list of `(label, macro_name, args[], goto_s, goto_f)` tuples.
The emit pass just prints them in three-column format. No logic in the emit pass.

**Sprint A13 steps:**
1. Define `AsmNode` struct: `{char *label; char *macro; char **args; int nargs; char *gs; char *gf;}`
2. Write `asm_ir_build(Program*)` → `AsmNode[]` — the tree walk, no emission
3. Write `asm_ir_emit(AsmNode[])` — pure pretty-printer, three columns
4. Replace current `asm_emit_program()` with `asm_ir_build()` + `asm_ir_emit()`
5. **M-ASM-IR fires** when beauty_prog.s generates identically via the new path

### Sprint A14 — M-ASM-BEAUTIFUL

**Goal:** beauty_prog.s is as readable as beauty_full.c. A human can follow the SNOBOL4 logic by reading the `.s` file directly.

**Trigger:** Open beauty_prog.s and beauty_full.c side by side. Every SNOBOL4 statement is recognisable in both. The Byrd box four ports are visible as α/β/γ/ω. Statement boundaries are clear. No raw register names in the body — only macro calls.

**M-ASM-BEAUTIFUL fires** when Lon reads beauty_prog.s and says it is beautiful.

### Completed

| Sprint | Commit |
|--------|--------|
| `space-token` | `3581830` |
| `compiled-byrd-boxes` | `560c56a` |
| `crosscheck-ladder` — 106/106 | `668ce4f` |
| `cnode` | `ac54bd2` |
| `rebus-roundtrip` | `bf86b4b` |
| `smoke-tests` — 21/21 | `8f68962` |
| sprints 0–22 (engine foundation) | `test/sprint*` |

---

## Pivot Log

| Sessions | What | Why |
|----------|------|-----|
| 159 | **PIVOT: M-ASM-BEAUTIFUL (A14) activated.** E_OR/E_CONC → ALT/CONCAT builtins registered; test 101 PASS. snobol4_asm.mac extended with STORE_ARG32/16, LOAD_NULVCL, APPLY_FN_0/N, SET_CAPTURE, IS_FAIL_BRANCH/16, SETUP_SUBJECT_FROM16. prog_emit_expr + asm_emit_program raw register sequences replaced with macro calls throughout. beauty_prog_session159.s archived (18220 lines, nasm clean). 106/106 26/26. HEAD a361318. | Lon requested M-ASM-BEAUTIFUL pivot. M-ASM-BEAUTY (102-109 Parse Error) deferred. |
| 158 | **M-ASM-BEAUTY progress — 101_comment PASS:** section .text fix; stack align; E_FNC/Case1-SF/capture; 106/106 26/26. Root cause of 102-109: E_OR/E_CONC → NULVCL. | — 3 issues diagnosed, sprint steps written.** Multi-capture (055): per-variable cap buffers + cap_order table in emitter + harness walk. E_INDR (056): add case + fix build_bare_sno to keep *VAR-referenced plain assigns + fix extract_subject to use subject var from match line. FAIL/057: already wired, unblocked once script continues past 055. SPITBOL p_imc studied for canonical multi-capture semantics. HQ updated. |
| 150 | **Sprint A9 — 17/20 ASM crosscheck PASS.** New emitters: ANY/NOTANY/SPAN/BREAK/LEN/TAB/RTAB/REM/ARB/FAIL all wired into E_FNC switch. E_VART: REM/ARB/FAIL intercepted as zero-arg builtins. Harness rewritten with setjmp/longjmp unanchored scan loop. DOL writes to harness cap_buf/cap_len externs. cap_len sentinel UINT64_MAX distinguishes no-capture from empty-string capture. build_bare_sno keeps pattern-variable assignments. DATATYPE lowercase fix (106/106). 038–054 PASS. 055 fails (multi-capture). Script stops early at first FAIL — next session fix extract_subject + skip multi-capture + wire E_INDR. HEAD d7a75cc. | |
| 149 | **Sprint A9 begun.** `snobol4_asm_harness.c`: flat `subject_data[65536]` array (preserves `lea rsi,[rel subject_data]` semantics), `match_success`/`match_fail` as C `noreturn` functions, inline `jmp root_alpha`. `-asm-body` flag: `asm_emit_body()` emits `global root_alpha,root_beta` + `extern cursor,subject_data,subject_len_val,match_success,match_fail`. `run_crosscheck_asm.sh`: extracts subject, builds bare `.sno`, sno2c→nasm→gcc→run, capture tests diff stdout vs `.ref`, match/no-match tests check exit code. **038_pat_literal PASS** end-to-end. Next: wire `emit_asm_any/span/break/notany/tab/rtab/len/rem/arb` into `E_FNC` switch. 106/106 holds. HEAD a7c324e. | |
| 148 | **M-ASM-ASSIGN + M-ASM-NAMED fire.** ASSIGN: assign_lit.s (LIT $ capture) + assign_digits.s (SPAN $ capture unanchored) PASS; emit_asm_assign() DOL Byrd box from v311.sil ENMI; E_DOL+E_NAM wired. NAMED: ref_astar_bstar.s (ASTAR=ARBNO("a"), BSTAR=ARBNO("b") on "aaabb") + anbn.s (4 sequential named-pattern call sites on "aabb") PASS; AsmNamedPat registry + asm_scan_named_patterns() pre-pass + emit_asm_named_ref() call-site + emit_asm_named_def() body emitter; E_VART wired; Proebsting §4.5 gate convention (pat_NAME_ret_gamma/omega .bss indirect-jmp, no call stack). End-to-end .sno→sno2c -asm→nasm→ld→run verified. 106/106 invariant holds. HEAD de085e1. Next: Sprint A9 — snobol4_asm_harness.c + body-only emitter + ASM crosscheck driver. | |
| 147 | **M-ASM-ALT + M-ASM-ARBNO + M-ASM-CHARSET fire; emit_byrd_asm.c real emitter written.** ALT: alt_first/second/fail. ARBNO: arbno_match/empty/alt (cursor stack 64 slots, zero-advance guard, v311.sil ARBN/EARB). CHARSET: any_vowel/notany_consonant/span_digits/break_space — all PASS. emit_byrd_asm.c: real recursive LIT/SEQ/ALT/POS/RPOS/ARBNO emitter — generates correct NASM but needs harness to connect to crosscheck (subject currently hardcoded). Next: Sprint A7 — snobol4_asm_harness.c + body-only emitter + first crosscheck pass. HEAD a114bcf. | |
| 147 | **M-ASM-ALT + M-ASM-ARBNO fire** — ALT: three oracles (alt_first/second/fail). ARBNO: three oracles (arbno_match "aaa", arbno_empty "aaa" vs 'x' → fail, arbno_alt "abba" vs ARBNO('a'\|'b')). ARBNO design: flat .bss cursor stack 64 slots + depth counter; α pushes+succeeds; β pops+tries one rep; zero-advance guard; rep_success pushes+re-succeeds. Proebsting §4.5 for ALT; v311.sil ARBN/EARB/ARBF for ARBNO. All PASS. Next: Sprint A6 (CHARSET). | |
| 146 | **M-ASM-LIT fires** — `lit_hello.s` hand-written: α/β/γ/ω real NASM labels, cursor+saved_cursor flat .bss qwords, repe cmpsb compare. Assembles, links, runs → `hello\n` exit 0. Diff vs oracle CLEAN. `artifacts/asm/null.s` + `artifacts/asm/lit_hello.s` placed in artifacts/asm/. HQ updated. No push per Lon. Next: Sprint A2 (POS/RPOS). |
| 145 | **M-ASM-HELLO fires** — `emit_byrd_asm.c` created, `-asm` flag added to `main.c`+`Makefile`, `null.s` assembles+links+runs → exit 0. 106/106 crosscheck clean. Next: Sprint A1 (LIT node). | Sprint A0 complete. |
| 144 | **PIVOT: x64 ASM backend** — abandon monitor-scaffold/bug7-bomb | C backend has structural flaw: named patterns require reentrant C functions, `pat_X_t` structs, `calloc`, three-level scoping. ASM eliminates all of it: α/β/γ/ω = real labels, all vars flat `.bss`, named patterns = labels + 2-way jmp. One scope. Sprint plan A0–A10 documented in NOW. |
| 80–89 | Attacked beauty.sno directly | Burned — needed smaller test cases first |
| 89 | Pivot: corpus ladder | Prove each feature before moving up |
| 95 | 106/106 rungs 1–11 | Foundation solid |
| 96–97 | Sprint 4 compiler internals | Retired — not test-driven |
| 97 | Pivot: test-driven only | No compiler work without failing test |
| 98–99 | HQ restructure (L1/L2/L3 pyramid) | Plan before code |
| 100 | HQ: frontend×backend split | One file per concern |
| 101 | Sprint A begins | Rung 12, beauty_full_bin, first crosscheck test (Session 101) |
| 103–104 | E_NAM~/Shift fix; E_FNC fallback fix | 101_comment PASS; 102+ blocked by named-pattern RHS truncation in byrd_emit_named_pattern |
| 105 | $ left-assoc parse fix + E_DOL chain emitter | Parser correct; emitter label-dup compile error blocks 102+ |
| 106 | E_DOL label-dup fixed (emit_seq pattern); 4x crosscheck speedup | 101 PASS; 102_output FAIL — assignment node blank in pp() |
| 108 | E_INDR(E_FNC) fix in emit_byrd.c; beauty_full.c patched; bug2 diagnosed: pat_ExprList epsilon | 102_output still FAIL — bug2 is pat_ExprList matching epsilon without '(' |
| 109 | bug2 '(' guards added (both Function+Id arms); pop_val()+skip; doc sno* names fixed in .github | 102_output still FAIL — OUTPUT not reaching subject slot; bare-Function arm not yet found |
| 110 | bug2 FIXED: bare-Function/Id go to fence_after_358 (keep Shift, succeed); parse tree verified correct by trace | 102_output still FAIL — Bug3: pp_Stmt drops subject; INDEX_fn(c,2) suspect |
| 107 | Shift(t,v) value fix; FIELD_GET debug removed; root cause diagnosed | 106/106 pass; 102 still FAIL — E_DEREF(E_FNC) in emit_byrd.c drops args |
| 111 | NPUSH not firing on backtrack in pat_Expr3/4; ntop()=0 at Reduce | Full stack probe confirmed; emit_simple_val E_QLIT fix applied; structural NPUSH hoist pending in emit_byrd.c |
| 112 | Bug3 FIXED (emit_seq NPUSH on backtrack); Bug4 FIXED (emit_imm literal-tok $'(' guard + stack rollback via STACK_DEPTH_fn) | 101/102/103 PASS; 104_label FAIL — next |
| 113 | Bug5 diagnosed: ntop() frame displacement by nested NPUSH; NINC_AT_fn + saved-frame fix in beauty_full.c; Reduce("..",2) fires; pp_.. crash unresolved | EMERGENCY WIP 7c17ffa |
| 114 | Bug5 FIXED: saved-frame pattern extended to pat_Parse/pat_Compiland/pat_Command; _command_pending_parent_frame global; Reduce(Parse,1) correct; 104_label PASS. Bug6 diagnosed: Bug6a spurious Reduce(..,2) for goto token; Bug6b unevaluated goto type string | EMERGENCY WIP 3f5bfda |
| 115 | Bug6a FIXED: `:` lookahead guard in pat_X4 cat_r_168. Bug6b FIXED: NV_SET_fn for Brackets/SorF in pat_Target/SGoto/FGoto; CONCAT_fn Reduce type; suppressed output_str+cond_OUTPUT in all pat_ gammas (23 sites). 101–105 PASS, 106/106. WIP only — emit_byrd.c port pending | EMERGENCY WIP — commit next session |
| 116 | emit_byrd.c port attempt: snobol4.h NTOP_INDEX/NSTACK_AT decls; pending_npush_uid + _pending_parent_frame globals; Bug5 saved-frame in emit_seq+E_FNC nPush; Bug6a colon guard in *X4 deref; Bug6b CONCAT_fn in E_OPSYN; output_str suppression gated on suppress_output_in_named_pat(); _parent_frame field in all named pat structs. 101-103 PASS from regen; 104-105 FAIL — pending_npush_uid not surviving nested CAT levels | EMERGENCY WIP — pending_npush_uid fix next session |
| 117 | Diagnosis: 104/105 fail because Reduce(..,2) never fires — ntop()=1 at ExprList level instead of 2. Dual-stack trace confirmed: spurious NPUSH idx=7/8 inside pat_Expr displaces counter stack so second NINC fires at wrong level. Root cause: nPush/nPop imbalance in pat_Expr4/X4 sub-pattern. Option A (parameter threading) attempted and backed out — correct diagnosis but wrong fix target. All files restored to session116 state. | Diagnosis only — no commit |
| 118 | Pivot: stack-trace sprint. Understand two-stack engine model fully. Instrument both oracle and compiled binary. Use diff to find exact imbalance location, not inference. New milestone M-STACK-TRACE gates on beauty-crosscheck. HQ updated. | Plan only — no commit |
| 119 | M-STACK-TRACE fires. oracle_stack.txt == compiled_stack.txt for all rung-12 inputs. | Stack trace matched — sprint beauty-crosscheck begins |
| 121 | Dual-stack trace infra built: oracle (patched counter.sno→TERMINAL) + compiled (fprintf in NPUSH/NINC/NPOP). 109_multi.input trace diff: first divergence line 2 — oracle NINC, compiled spurious NPUSH. Bug7 Bomb Protocol designed (Pass1 count, Pass2 limit+backtrace). emit_imm NPOP-on-fail drafted but emit_seq Expr15 fix caused double-pop regression on 105_goto. All WIP reverted. Bomb protocol is next. | Bomb protocol ready — awaiting next session |
| 120 | beauty.sno PATTERN read in full (lines 293–419). Bug7 confirmed: Expr17 FENCE arm 1 calls nPush() then $'(' fails — nPop() never called on ω path. Expr15 FENCE arm same issue. Fix target: emit_byrd.c FENCE backtrack path. HQ updated with full pattern structure. ~55% context at session start. | Plan only — awaiting instruction |
| 122 | Pivot: diag1-corpus sprint before bug7-micro. 35 tests 152 assertions rungs 2–11, 35/35 PASS CSNOBOL4 2.3.3. M-FLAT documented (flat() Gray/White bypass of pp/ss). HQ updated. Context ~94% at close. | diag1 corpus ready to commit with token; bug7-micro is next |
| 122b | PIVOT: M-DIAG1 now top priority. Run diag1 35-test suite on JVM + DOTNET. Fix failures. Fire M-DIAG1. Then bug7-micro. Priority order: M-DIAG1 → M-BEAUTY-CORE → M-FLAT → M-BEAUTY-FULL → M-BOOTSTRAP. | New session opens on snobol4jvm |
