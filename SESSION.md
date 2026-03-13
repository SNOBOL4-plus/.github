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
| **HEAD** | `a69971e` — fix(emit): field assignment lvalue — sno_field_set for E_CALL nargs==1; T_FUNC engine node; deferred user_call at match time |

## Last Thing That Happened

### Bug chain fully diagnosed. Two fixes landed, one remaining.

**Fix 1 — parse_expr3 for pattern field ✅ (`f359079`)**
`parse_body_field` called `parse_expr2` → ate `|` alternation.
Changed to `parse_expr3` → includes `|`, excludes `=` and `?`.
Smoke tests still 0/21 (deeper runtime bug masked this).

**Fix 2 — field assignment lvalue ✅ (`a69971e`)**
`emit_assign_target` catch-all emitted `sno_iset(sno_apply("val",{n},1), rhs)`.
`sno_iset` converts its first arg to a string and calls `sno_var_set` — treating
a DATA field accessor as an indirect variable. So `value($'#N') = value($'#N') + 1`
was silently doing nothing.

Fix: added `E_CALL && nargs==1` branch in `emit_assign_target`:
```c
} else if (lhs->kind == E_CALL && lhs->nargs == 1) {
    E("sno_field_set("); emit_expr(lhs->args[0]); E(", \"%s\", %s);\n", lhs->sval, rhs_str);
}
```
6 sites fixed in beauty_full.c (session52 artifact). Direct counter calls now work.

**Also landed: T_FUNC engine node (WIP, not yet needed)**
Added `T_FUNC = 44` to `engine.h` + 4 cases in `engine.c`.
Added `func` / `func_data` fields to `Pattern` struct.
`SPAT_USER_CALL` in `snobol4_pattern.c` now uses `T_FUNC` for side-effect
functions (those returning SNO_NULL). Not yet the fix path — see below.

**Fix 3 — capture var-name deferred evaluation ❌ (NEXT ACTION)**

Root cause of smoke tests still 0/21:

In SNOBOL4, `epsilon . *IncCounter()` means:
- match epsilon (zero-width)
- assign matched substring ("") to the variable named by `*IncCounter()`
- `*IncCounter()` is evaluated AT MATCH TIME to yield a variable name
- IncCounter's SIDE EFFECT (`value($'#N') += 1`) runs during that evaluation

Our capture materialisation in `snobol4_pattern.c` stores a **static `char* var_name`**
at materialise time. For deferred capture vars (`SPAT_USER_CALL`, `SPAT_DEREF`, etc.)
it stores `NULL` → debug shows `var=?` → capture fires but does nothing.

Evidence:
```
CAPTURE_CB: slot=0 var=? start=0 end=0   ← var_name is NULL
CAPTURE: ? = ""                           ← assigned to nobody
matched, top=                             ← TopCounter() returns empty
```

**Where to fix:** `snobol4_pattern.c`, in the capture materialisation block.
Find where `Capture.var_name` is set. When the var expression is a `SPAT_USER_CALL`
or `SPAT_DEREF`, instead of storing a static name, store a callback that evaluates
the expression at match time to get the name.

**Concrete approach:**
1. Add `var_fn` / `var_data` fields to `Capture` struct (alongside existing `var_name`)
2. In `apply_captures()`: if `cap->var_fn` is set, call it to get the var name, then assign
3. In capture materialisation: when var expr is `SPAT_USER_CALL`, set `var_fn` to call the function

## One Next Action

Find and fix capture var-name deferred evaluation in `snobol4_pattern.c`:

```bash
cd /home/claude/SNOBOL4-tiny

# First: find where Capture.var_name is set during materialisation
grep -n "var_name\|cap->var\|Capture" src/runtime/snobol4/snobol4_pattern.c | head -30

# The Capture struct is defined around line 295-305 in snobol4_pattern.c
# var_name is set in the SPAT_CAPTURE case of materialise()
# Need to handle SPAT_USER_CALL / SPAT_DEREF var expressions

# After fix, verify with test_ninc:
R=src/runtime/snobol4
src/sno2c/sno2c /tmp/test_ninc.sno > /tmp/test_ninc.c
gcc -O0 /tmp/test_ninc.c $R/snobol4.c $R/snobol4_inc.c $R/snobol4_pattern.c \
    src/runtime/engine.c -I$R -Isrc/runtime -lgc -lm -w -o /tmp/test_ninc_bin 2>&1 | grep error
/tmp/test_ninc_bin
# Expected: "matched, top=2"

# Then rebuild beauty and run smoke tests:
src/sno2c/sno2c \
  /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc \
  > /tmp/beauty_full.c
gcc -O0 -g /tmp/beauty_full.c \
    $R/snobol4.c $R/snobol4_inc.c $R/snobol4_pattern.c \
    src/runtime/engine.c -I$R -Isrc/runtime -lgc -lm -w -o /tmp/beauty_full_bin 2>&1 | grep error
bash test/smoke/test_snoCommand_match.sh /tmp/beauty_full_bin
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

## Artifacts Protocol (do this every session sno2c changes)

After any sno2c fix, regenerate and commit beauty_full.c to artifacts/:
```bash
cd /home/claude/SNOBOL4-tiny
src/sno2c/sno2c \
  /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc \
  > artifacts/beauty_full_session_<slug>.c
wc -l artifacts/beauty_full_session_<slug>.c
# Update artifacts/README.md, then:
git add artifacts/
git commit -m "artifact: beauty_full_session_<slug>.c — <N> lines, <what changed>"
```

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
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
