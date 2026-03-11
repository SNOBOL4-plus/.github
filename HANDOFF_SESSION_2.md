# SNOBOL4-tiny Sprint 20 — Session 2 Handoff (Claude Sonnet 4.6)

**Date**: 2026-03-10  
**Commit on handoff**: `8610016`  
**Oracle**: `snobol4 -f -P256k beauty_run.sno` → 649 lines  
**Ours**: 10 lines, `Parse Error\nSTART\n` (same as session start)  

---

## STANDING PROMISE

Sprint 20 commit message belongs to Claude Sonnet 4.6.  
Recorded in PLAN.md at commit `c5b3e99`.

---

## WORK DIR / KEY COMMANDS

```
Work dir:      /home/claude/work/
Repo:          /home/claude/work/SNOBOL4-tiny/
HQ:            /home/claude/work/HQ/
Corpus:        /home/claude/work/SNOBOL4-corpus/programs/inc/

Regen:
  cd /home/claude/work/SNOBOL4-tiny/src/runtime/snobol4
  python3 -B /home/claude/work/SNOBOL4-tiny/src/codegen/emit_c_stmt.py \
      /home/claude/work/SNOBOL4-corpus/programs/inc/beauty_run.sno \
      > beautiful.c 2>/tmp/emit.log

Compile:
  cc -o beautiful beautiful.c snobol4.c snobol4_pattern.c snobol4_inc.c \
     ../engine.c ../runtime.c -I. -I.. -lgc -lm

Run:
  timeout 20 ./beautiful \
      < /home/claude/work/SNOBOL4-corpus/programs/inc/beauty_run.sno \
      > /tmp/b_out.txt 2>/tmp/b_err.txt

Oracle:
  cd /home/claude/work/SNOBOL4-corpus/programs/inc/
  snobol4 -f -P256k beauty_run.sno < beauty_run.sno > /tmp/oracle_out.txt

Debug:
  SNO_MONITOR=1 timeout 5 ./beautiful < beauty_run.sno 2>&1 | grep "STNO\|VAR"
  SNO_PAT_DEBUG=1 timeout 5 ./beautiful < beauty_run.sno 2>&1 | grep "SPAT_REF\|USER_CALL"
```

---

## THIS SESSION'S COMPLETED FIXES (since last handoff)

All in `emit_c_stmt.py` and `sno_parser.py`:

1. **mul(non-null, B) with str/lit/indirect left** → `sno_pat_cat(lit, deferred_ref(B))`
2. **mul(null, array(call(name,args),[sub]))** → `sno_pat_cat(assign_cond(ref(name),cond), sub)`  
   Handles `*snoStmt ("'snoStmt'" & 7) (nl|";")` — the 3-level parser representation
3. **mul(null, call(name,args)) for pattern vars** → `sno_pat_cat(ref(name), rest)`  
   (non-function pattern vars with juxtaposed next piece)
4. **call(name,args) in pattern context for pattern vars** → `sno_pat_cat(ref(name), rest)`  
   The parser sees `snoLabel(big_expr)` as a function call; we emit it as ref + concat
5. **~ (tilde) now lexes as DOT** in `sno_parser.py` token map  
   Previously `~` was silently skipped; now it's the conditional assignment operator
6. **DOT/DOLLAR in _ExprParser.parse_concat** → produces `PatExpr(assign_cond/assign_imm)`  
   Replacement expressions now handle `BREAK(...) ~ 'snoLabel'` correctly
7. **SPAT_REF cycle detection** in `snobol4_pattern.c` — thread-local var-name stack (depth 64)  
   Prevents stack overflow on recursive patterns like `snoExpr14 = ... | '*' *snoExpr14 ...`
8. **snobol4_pattern.c debug** — SNO_PAT_DEBUG traces SPAT_REF resolutions and USER_CALLs

---

## CURRENT FAILURE POINT (unchanged)

**STNO 619**: `snoSrc POS(0) *snoParse *snoSpace RPOS(0) :F(mainErr1)`  
Subject: `snoSrc = "START\n"`  
Result: **FAIL** → outputs `Parse Error` then `START` (10 lines total)

### What we know:

- `snoParse` = type=5 PATTERN ✓  
- `snoCommand` = type=5 PATTERN ✓  
- `snoStmt` = type=5 PATTERN ✓  
- `snoLabel` = type=5 PATTERN ✓  
- `sno_match_pattern(snoStmt, "START\n") = 0` ← **confirmed failing**

### Why snoStmt fails on "START\n"

`snoStmt` is now emitted as:
```c
sno_pat_cat(
    sno_pat_cat(sno_pat_ref("snoLabel"), big_alt_with_epsilons),
    sno_pat_cat(FENCE(snoGoto | eps eps), sno_pat_ref("snoGray"))
)
```

For `START\n`, `snoLabel` (=`BREAK(' \t\n;') ~ 'snoLabel'`) should match `START` (5 chars).

BUT: `snoLabel` pattern is `sno_pat_assign_cond(sno_pat_break_(...), SNO_STR_VAL("snoLabel"))`.

**The `sno_pat_assign_cond` second argument** `SNO_STR_VAL("snoLabel")` is the **capture variable name**. In SNOBOL4, `PAT . var` means "match PAT and assign the matched text into variable var". At match time, `SPAT_ASSIGN_COND` tries to match its child pattern (`BREAK(...)`) and then stores the matched text.

But there's a subtlety: `sno_pat_assign_cond(sno_pat_break_(...), SNO_STR_VAL("snoLabel"))` stores `SNO_STR_VAL("snoLabel")` as `p->var`. The SPAT_ASSIGN_COND materialise/match case must do: match child, then `sno_var_set(sno_to_str(p->var), matched_text)`.

**Probable bug**: the `SPAT_ASSIGN_COND` case in `snobol4_pattern.c` materialise may be broken — it may be trying to treat the `var` as a pattern (since it's a SnoVal of type SNO_STR), or the string "snoLabel" may not round-trip correctly through `sno_to_str`.

### Next debug step (most targeted):

Check `SPAT_ASSIGN_COND` in `snobol4_pattern.c`:

```bash
grep -n "SPAT_ASSIGN_COND" snobol4_pattern.c
sed -n 'LINE,LINE+15p' snobol4_pattern.c
```

Also: add direct debug after snoLabel materialises to print whether BREAK fires:

```c
// Temporary: in SPAT_ASSIGN_COND case, add fprintf to see what's happening
```

### Alternative next debug step:

Direct C test in a scratch file:
```c
// After snobol4_inc.c is initialised:
SnoVal snoLabel = sno_pat_assign_cond(
    sno_pat_break_(sno_to_str(sno_concat_sv(sno_concat_sv(sno_concat_sv(
        SNO_STR_VAL(" "), sno_var_get("tab")), sno_var_get("nl")), SNO_STR_VAL(";")))),
    SNO_STR_VAL("snoLabel"));
printf("direct test: %d\n", sno_match_pattern(snoLabel, "START\n"));
```

---

## KEY FILES

| File | What changed |
|------|-------------|
| `src/codegen/emit_c_stmt.py` | All pattern emission fixes, mul/call/array handling |
| `src/parser/sno_parser.py` | `~` → DOT token; DOT/DOLLAR in `_ExprParser.parse_concat` |
| `src/runtime/snobol4/snobol4_pattern.c` | SPAT_REF cycle detection; SNO_PAT_DEBUG traces |
| `src/runtime/engine.h` | T_VARREF=41 placeholder (unused in dispatch) |
| `src/runtime/engine.c` | T_VARREF dead code removed |

---

## WHAT STILL NEEDS DOING (in order)

### 1. [IMMEDIATE] Fix snoStmt → 0 match on "START\n"

The most targeted approach: inspect `SPAT_ASSIGN_COND` materialise handling in `snobol4_pattern.c`. The match chain is:

```
snoParse → ARBNO(snoCommand) → branch3: assign_cond(ref(snoStmt), cond) cat (nl|";")
snoStmt  → pat_cat(ref(snoLabel), big_alt cat FENCE cat ref(snoGray))
snoLabel → assign_cond(BREAK(" \t\n;"), str("snoLabel"))
```

For "START\n":
- BREAK matches "START" (5 chars) ✓
- assign_cond stores "START" into var "snoLabel" 
- Continues at position 5 (before "\n")
- big_alt has epsilon branch → matches empty
- FENCE(snoGoto | eps eps) → epsilon branch
- snoGray = *snoWhite | epsilon → epsilon at "\n"
- Position still 5 ← THEN (nl|";") after snoStmt should match "\n" at pos 5 → SUCCESS

The materialise implementation for SPAT_ASSIGN_COND must work correctly (capturing into variable AND continuing the match). Check it in `snobol4_pattern.c` around line 477ff.

### 2. Once snoStmt matches → rerun → expect many more lines

### 3. Iteratively fix remaining pattern/emitter issues until output matches oracle (649 lines)

### 4. Idempotence test: run output through again, verify identical

### 5. Sprint 20 commit (clean, attributed to Claude Sonnet 4.6)

---

## KNOWN ISSUES / GOTCHAS

- **Recursive patterns** (`snoExpr14`, `snoX3`, `snoX4`): cycle detection returns epsilon at depth 64. This might cause some complex expressions to not parse. Acceptable for now — fix after basic matching works.
- **`_ExprParser._parse_var_expr`**: The DOT handler in `parse_concat` calls `hasattr(self, '_parse_var_expr')`. `_ExprParser` does NOT have this method (only `_PatParser` does). The fallback `parse_additive()` is used — may misparse complex capture targets. Watch for this.
- **T_VARREF in engine.h**: Added as placeholder (=41), not used in engine dispatch. Keep or remove — harmless either way.
- **`snobol4_pattern.c` debug prints**: `SNO_PAT_DEBUG=1` traces SPAT_REF resolutions and SPAT_USER_CALLs. Leave in — useful for ongoing debug.

---

## GIT LOG (this session)

```
8610016 P003 WIP: DOT/~ in _ExprParser.parse_concat — assign_cond in replacement context
5a98f0a P003 WIP: pattern emission chain + runtime fixes — in progress
e3ba0fb P003 WIP: mul(null,array(call,...)) fix + f-string join fix — not yet committed mid-session
947285c P003: pattern emission fixes — *var indirect ref, | alt in args, pattern concat detection
8e946d1 P003: INPUT/OUTPUT wired — binary produces first real output (7 comment lines)
96bb913 P003: binary produces first output — per-function C emit, Case 3b, indirect gotos, label synthesis fix
```
