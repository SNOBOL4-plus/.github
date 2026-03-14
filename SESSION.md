# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `compiled-byrd-boxes` (new — replaces retired `smoke-tests`) |
| **Milestone** | M-COMPILED-BYRD |
| **HEAD** | `be4fbb1` — runtime: INPUT file-redirect + VarCache, smoke-tests sprint retired |

---

## The Architectural Decision (Lon + Claude, 2026-03-14)

**The `smoke-tests` sprint was validating the wrong runtime and has been retired.**

`sno_pat_*` / `engine.c` is a stopgap interpreter. Validating `sno2c` against it
proves nothing about the compiled Byrd box path that actually matters.

The correct test: does `sno2c` emit correct labeled-goto Byrd box C?
Validated against the hand-written sprint0–22 oracle files that already exist.

**The Python pipeline (lower.py + emit_c_byrd.py) produced correct output — 609/609
worm cases, full Chomsky hierarchy. That is the ground truth. `emit_byrd.c` is a
C port of that pipeline, wired into sno2c.**

---

## What `compiled-byrd-boxes` Sprint Does

Replace `emit_pat()` in `src/sno2c/emit.c` with calls to a new `src/sno2c/emit_byrd.c`
that emits labeled-goto C — same structure as the Python pipeline output.

**Before (wrong — interpreter):**
```c
sno_pat_cat(sno_pat_lit("hello"), sno_pat_var("REM"))
```

**After (correct — compiled Byrd boxes):**
```c
_n42_alpha: /* CAT entry */
    goto _n43_alpha;
_n43_gamma: /* left succeeded */
    goto _n44_alpha;
_n44_gamma: /* right succeeded */
    goto _n42_gamma;
_n43_beta:
_n44_beta:
    goto _n42_beta;
```

---

## One Next Action — Step by Step

### Step 1: Study the Python pipeline (read these files first)
```
src/ir/byrd_ir.py          — IR node types and port structure
src/ir/lower.py            — lowers pattern AST to Byrd IR
src/codegen/emit_c_byrd.py — emits labeled-goto C from Byrd IR
```
These are the ground truth. emit_byrd.c is a C port of lower.py + emit_c_byrd.py.

### Step 2: Study the sprint oracles
```
test/sprint0/   through   test/sprint5/
```
Understand exactly what C the emitter should produce for LIT, CAT, ALT, EPSILON,
ARBNO, CAPTURE. These are the correctness gate.

### Step 3: Write `src/sno2c/emit_byrd.c`
Start with just: LIT, CAT, ALT, EPSILON.
Wire into emit.c — replace emit_pat() for those node types.
Get sprint0–5 oracles compiling and passing.

### Step 4: Add remaining node types
ARBNO, CAPTURE (dot/dollar), FENCE, POS, TAB, RPOS, RTAB.
Get sprint6–15 passing.

### Step 5: Add USER_CALL nodes
nInc, nPush, nPop, Reduce — direct C function calls at the right port.
Get sprint16–22 passing.

### Step 6: Drop sno_pat_* from compiled path
- emit.c: remove all sno_pat_* emission
- snoc_runtime.h: remove sno_match / sno_pat_* macros (keep SnoVal, sno_init)
- Compiled binary no longer links engine.c or snobol4_pattern.c
- Both files stay for EVAL/dynamic pattern path

**Milestone trigger:** sprint0–22 all pass, beauty_full_bin links without engine.c.

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — retired from compiled path.
  Any test failure means fix emit_byrd.c, not the interpreter.
- Do NOT chase sno_match_pattern / materialise bugs — irrelevant to Byrd boxes.
- Do NOT run test_snoCommand_match.sh — validates the wrong runtime.

---

## Container State (as of this handoff)

```
/home/claude/snobol4-install/bin/snobol4  — CSNOBOL4 2.3.3, built and working
/home/claude/SNOBOL4-corpus/              — cloned
/home/claude/SNOBOL4-tiny/               — cloned, sno2c built
```

These will NOT be present in next Claude's container. Clone fresh:
```bash
git config --global user.name "LCherryholmes"
git config --global user.email "lcherryh@yahoo.com"

git clone https://TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
git clone https://TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
git clone https://TOKEN@github.com/SNOBOL4-plus/.github.git snobol4-plus-github

# Build CSNOBOL4 — Lon will provide tarball or it may be in uploads
# apt-get install -y m4 libgc-dev
# tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
# ./configure --prefix=/home/claude/snobol4-install && make -j$(nproc) && make install
```

---

## Rebuild Commands

```bash
cd /home/claude/SNOBOL4-tiny

make -C src/sno2c

src/sno2c/sno2c \
  /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc \
  > /tmp/beauty_full.c

R=src/runtime/snobol4
gcc -O0 -g /tmp/beauty_full.c \
    $R/snobol4.c $R/snobol4_inc.c $R/snobol4_pattern.c \
    src/runtime/engine.c \
    -I$R -Isrc/runtime -lgc -lm -w \
    -o /tmp/beauty_full_bin

# Oracle
SNO=/home/claude/snobol4-install/bin/snobol4
$SNO -f -P256k -I /home/claude/SNOBOL4-corpus/programs/inc \
    /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
    < /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
    > /tmp/beauty_oracle.sno 2>/dev/null
```

---

## What Is Keeper Work

| File | What it is | Status |
|------|-----------|--------|
| `src/sno2c/` | C compiler — lex, parse, emit | ✅ Keeper |
| `src/runtime/snobol4/snobol4.c` | Value runtime, builtins, var table, I/O | ✅ Keeper |
| `src/runtime/snobol4/snobol4_inc.c` | Gen, Qize, Shift/Reduce, stack, counter | ✅ Keeper |
| `src/runtime/snobol4/snobol4.h` | Public API header | ✅ Keeper |
| `src/runtime/snobol4/snoc_runtime.h` | Glue header for emitted C | ✅ Keeper (shrinks) |
| `src/runtime/engine.c` | Byrd box interpreter | ⚠️ EVAL only |
| `src/runtime/snobol4/snobol4_pattern.c` | SnoPattern tree + materialise | ⚠️ EVAL only |
| `src/codegen/emit_c_byrd.py` | Python emitter — ground truth | ✅ Do not delete |
| `src/ir/lower.py` | Python lowering pass — ground truth | ✅ Do not delete |
| `src/ir/byrd_ir.py` | Python IR — ground truth | ✅ Do not delete |

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | `compiled-byrd-boxes` sprint opened; `smoke-tests` retired | smoke-tests validated wrong runtime |
| 2026-03-14 | VarCache + INPUT redirect committed (`be4fbb1`) | Keeper work |
| 2026-03-14 | T_FUNC denylist reverted | Wrong approach |
| 2026-03-13 | Architecture recorded: sno_pat_* stopgap, M-COMPILED-BYRD locked | Agreement with Lon |
| 2026-03-13 | materialise-once fix | SPAT_USER_CALL called N times per match |
| 2026-03-13 | ROOT CAUSE: materialise() called per scan position | Reduce() pops stack → 0/21 |
