# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `smoke-tests` (2/4 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `d5d3796` — EMERGENCY WIP: phantom skip fix + deferred var_fn — smoke tests still 0/21, nInc body missing in full inc context |

## Last Thing That Happened

### Two fixes landed, one root cause found — needs one more targeted fix.

**Fix 1 — deferred var_fn in Capture struct ✅ (`d5d3796`)**
`Capture.var_name` was static. For `epsilon . *IncCounter()`, `sp->var` is a `SNO_PATTERN`
(`SPAT_USER_CALL`), not `SNO_STR`. Added `var_fn`/`var_data` to `Capture` struct.
Added `deferred_var_fn()` helper. Updated `SPAT_ASSIGN` materialise to detect
`SNO_PATTERN` var and wire up callback. Updated `apply_captures()` to call `var_fn`
when `var_name` is NULL. Verified correct with minimal test.

**Fix 2 — phantom skip entry_label check ✅ (`d5d3796`)**
In `emit.c` phantoms loop, the "already in fn_table" check only compared `fn_table[fi].name`
to the phantom name. `DEFINE('nInc()', 'nInc_')` stores `name="nInc"`, `entry_label="nInc_"`.
Phantom `"nInc_"` was being added even though real fn `nInc` was registered.
Fix: also check `fn_table[fi].entry_label` in the `already` detection.

**Root cause of 0/21 still:**
Smoke tests still 0/21. The phantom fix IS correct (verified with minimal test_ninc.sno
in isolation — `_sno_fn_nInc` body emitted correctly). But when compiled with the FULL
inc set (`-I /home/claude/SNOBOL4-corpus/programs/inc`), the semantic.sno function bodies
(`nInc`, `shift_`, `reduce_`, etc.) still don't appear. Context window ran out before
root-causing the full-inc vs minimal difference.

**The minimal test works:**
```
DEFINE('nInc()', 'nInc_') :(semEnd)
nInc_  nInc = 'hello'    :(RETURN)
semEnd
```
→ `_sno_fn_nInc` body emitted ✅

**The full inc compilation does not:**
```bash
src/sno2c/sno2c /tmp/test_ninc.sno -I /home/claude/SNOBOL4-corpus/programs/inc
```
→ only `_sno_fn_nInc_` (phantom forward decl), no `_sno_fn_nInc` body ❌

## One Next Action

Find why full inc compilation loses the `nInc` body. Likely cause: one of the other
inc files (`stack.sno`, `ShiftReduce.sno`) has a DEFINE or label collision that
makes `collect_functions` overwrite or miss the two-arg DEFINEs from `semantic.sno`.

```bash
cd /home/claude/SNOBOL4-tiny

# Step 1: add one-line debug to collect_functions to print every DEFINE found
# In emit.c, after parse_proto(proto, fn) add:
#   fprintf(stderr, "COLLECT: name=%s entry=%s end=%s\n",
#       fn->name, fn->entry_label?fn->entry_label:"(null)",
#       fn->end_label?fn->end_label:"(null)");
# Rebuild and run:
SNOC_COLLECT_DEBUG=1 src/sno2c/sno2c /tmp/test_ninc.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc 2>&1 | grep -E "nInc|nPush|shift|reduce"

# Step 2: once root cause found, fix collect_functions or phantoms list

# Step 3: rebuild beauty + run smoke tests:
R=src/runtime/snobol4
src/sno2c/sno2c \
  /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc \
  > /tmp/beauty_full.c
gcc -O0 -g /tmp/beauty_full.c \
    $R/snobol4.c $R/snobol4_inc.c $R/snobol4_pattern.c \
    src/runtime/engine.c -I$R -Isrc/runtime -lgc -lm -w \
    -o /tmp/beauty_full_bin
bash test/smoke/test_snoCommand_match.sh /tmp/beauty_full_bin
# Target: 21/21
```

## Rebuild Commands

```bash
cd /home/claude/SNOBOL4-tiny

# sno2c rebuild
make -C src/sno2c

# beauty compile
src/sno2c/sno2c \
  /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc \
  > /tmp/beauty_full.c

# beauty binary
R=src/runtime/snobol4
gcc -O0 -g /tmp/beauty_full.c \
    $R/snobol4.c $R/snobol4_inc.c $R/snobol4_pattern.c \
    src/runtime/engine.c \
    -I$R -Isrc/runtime -lgc -lm -w \
    -o /tmp/beauty_full_bin

# smoke tests
bash test/smoke/test_snoCommand_match.sh /tmp/beauty_full_bin

# crosscheck suite (after smoke passes)
bash /home/claude/SNOBOL4-corpus/crosscheck/run_all.sh

# oracle
SNO=/home/claude/snobol4-2.3.3/snobol4
$SNO -f -P256k -I /home/claude/SNOBOL4-corpus/programs/inc \
    /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
    < /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
    > /tmp/beauty_oracle.sno 2>/dev/null

diff /tmp/beauty_oracle.sno /tmp/beauty_compiled.sno
```

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | phantom skip fix: check entry_label in already-detection | nInc_ phantom added despite nInc real fn present |
| 2026-03-13 | deferred var_fn in Capture struct — apply_captures calls fn at match time | capture var=? was NULL for *FuncCall() targets |
| 2026-03-13 | capture var-name deferred eval broken — var=? at match time | root cause of 0/21 |
| 2026-03-13 | field assignment fix (sno_field_set) — IncCounter now works directly | sno_iset treated field as indirect var |
| 2026-03-13 | T_FUNC engine node added (WIP) | side-effect fns at match time |
| 2026-03-13 | parse_expr2 → parse_expr3 for pattern field | restores \| alternation |
| 2026-03-13 | 106-test crosscheck suite built, committed to corpus | Lon: need lampposts |
| 2026-03-13 | parse_expr4 \| alternation fixed via LexMark | double-WS ate \| token |
| 2026-03-13 | Sprint 3 (`beauty-runtime`) complete — clean exit | first run worked |
| 2026-03-13 | Sprint 2 (`smoke-tests`) complete — 21/21 | hand-rolled lex/parse works |
| 2026-03-13 | M-REBUS fired → rebus-roundtrip complete | Rebus milestone done |
| 2026-03-12 | Bison/Flex → hand-rolled-parser | LALR(1) unfixable (139 RR) |
