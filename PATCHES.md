# PATCHES.md — Runtime Patches and Fixes Log
## SNOBOL4-tiny Sprint 20+

> **What this file is**: Every time we patch the runtime (`snobol4.c`,
> `snobol4.h`, `emit_c_stmt.py`, `snobol4_inc.c`, `snobol4_pattern.c`)
> to fix a bug found by the double-trace monitor or other diagnosis,
> it is recorded here. Patch number, symptom, root cause, fix, commit.
>
> **Why**: The monitor finds bugs fast. We need a record of what was
> found, why it was wrong, and what the fix was. This is the audit trail.
> Each patch is a data point about the gap between SNOBOL4 semantics
> and our compiled C translation.
>
> **Cross-reference**: See `PLAN.md` § Outstanding Items for priority
> classification (P1/P2/P3). See `MONITOR.md` for the diagnostic
> methodology. See `STRING_ESCAPES.md` for string literal handling.
> See `COMPILAND_REACHABILITY.md` for inc-file → C mapping.

---

## Patch Index

| # | File | Symptom | Root Cause | Status |
|---|------|---------|------------|--------|
| P001 | `snobol4.c` | `./beautiful` hangs forever | `&STLIMIT` declared but never enforced | **APPLYING NOW** |

---

## P001 — &STLIMIT Not Enforced
**Date**: 2026-03-10
**Found by**: Double-trace monitor (binary STNO stream, timeout run)
**Symptom**: `./beautiful < beauty_run.sno` hangs. Exit 124 (timeout).
STNO stream shows loop at statements 160↔161, variable `i` incrementing
without bound (observed value: 92020+ after 5 seconds).

**Diagnosis**:
```
STNO 160
VAR i "92018"
STNO 161
VAR  ""
STNO 160
VAR i "92019"
...repeating forever...
```
Statement 160 increments `i`. Statement 161 checks a condition and loops
back to 160. This is `&STLIMIT` territory — the loop should terminate
when the statement count exceeds the limit. But `sno_kw_stlimit = 50000`
is declared in `snobol4.c` and never checked. `sno_kw_stcount` does not
exist. The guard that should stop this loop has never been wired up.

**Root Cause**: `snobol4.h` declares `sno_kw_stlimit` as an `int64_t`
global. `snobol4.c` initializes it to 50000. Neither file increments
a counter or checks it. The keyword is a stub — present in name only.

**Fix**:
1. Add `int64_t sno_kw_stcount = 0;` to `snobol4.c`
2. Add `extern int64_t sno_kw_stcount;` to `snobol4.h`
3. In `sno_comm_stno(n)`: increment `sno_kw_stcount`; if it exceeds
   `sno_kw_stlimit` (and `sno_kw_stlimit >= 0`), abort with error message.
4. `sno_kw_stlimit` default: keep at 50000 for safety during development.
   Beautiful.sno's full run needs more — raise to 500000 once confirmed.

**Files changed**: `snobol4.c`, `snobol4.h`
**Commit**: (applying now)

---

*Template for future patches:*
```
## PNNN — Short title
**Date**: YYYY-MM-DD
**Found by**: [monitor / oracle diff / manual]
**Symptom**: [what the user/monitor saw]
**Diagnosis**: [trace output that identified it]
**Root Cause**: [the actual code defect]
**Fix**: [what was changed and why]
**Files changed**: [list]
**Commit**: [hash]
```

**P001 Resolution**: Fixed. `sno_kw_stcount` added, incremented in
`sno_comm_stno()`, checked against `sno_kw_stlimit`. Binary now exits
cleanly with error message at the limit instead of hanging forever.
Confirmed: `** &STLIMIT exceeded at statement 161 (&STCOUNT=50001)`.
But raising `&STLIMIT` to 2,000,000 still loops at 161 — revealing P002.

---

## P002 — sno_subscript_get2 Never Returns Failure
**Date**: 2026-03-10
**Found by**: P001 fix revealing the underlying loop structure
**Symptom**: After P001, `./beautiful` exits at `&STLIMIT` instead of
hanging — but the loop is still at STNO 160↔161, variable `i` incrementing
to 2,000,001. Raising `&STLIMIT` to any value does not help.

**Diagnosis**:
```c
/* Statement 161 — beautiful.c */
_stmt_152: {  /* L161 */
    SnoVal _subj = sno_var_get(sno_to_str(sno_var_get("UTF_Array")));
    int _ok = sno_match_and_replace(&_subj,
                  sno_pat_epsilon(),
                  sno_subscript_get2(sno_var_get("UTF_Array"),
                                     sno_var_get("i"),
                                     SNO_INT_VAL(1LL)));
    sno_var_set(sno_to_str(sno_var_get("UTF_Array")), _subj);
    if (_ok) goto SNO_G1;   /* loop back to stmt 160: i = i + 1 */
    else     goto _stmt_153; /* exit loop */
}
```
The loop exit condition is `_ok == 0` (match failure). This fires when
`sno_subscript_get2(UTF_Array, i, 1)` returns a value that causes the
match to fail — i.e. when `i` is past the end of the array.

`sno_subscript_get2` is returning a value for every `i` (likely
`SNO_NULL_VAL` or empty string) rather than signaling end-of-array.
The pattern `sno_pat_epsilon()` matches the empty string, so even a
null subscript result causes `_ok = 1` and the loop continues.

**Root Cause**: Two interacting bugs:
1. `sno_subscript_get2` does not return a sentinel that signals
   out-of-bounds — it returns NULL_VAL or empty string for any `i`.
2. `sno_pat_epsilon()` matches NULL_VAL/empty string, so the replacement
   always succeeds regardless of subscript result.

In SNOBOL4 semantics, accessing a nonexistent array element should either
(a) fail the statement (causing the `F:` branch — `goto _stmt_153`), or
(b) return an uninitialized value that causes the match to fail.
CSNOBOL4 takes approach (a): out-of-bounds subscript causes statement failure.

**Fix**: `sno_subscript_get2` must return a failure signal when `i` is
out of bounds. The emitter's match-and-replace wrapper must propagate
that signal as `_ok = 0`.

**Files to change**: `snobol4.c` (`sno_subscript_get2`), possibly `emit_c_stmt.py`
**Status**: DIAGNOSING
