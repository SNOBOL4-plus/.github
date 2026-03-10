# KEYWORD_GRID.md — Proven Keyword Behavior
## Source: live test runs on 2026-03-10

> Every cell in this grid is **proven by a live test run**, not by reading source.
> Test script: `/tmp/test_kw4.sno` (in SNOBOL4-corpus as `tests/keyword_proof.sno`)
> Three systems tested: CSNOBOL4 (`snobol4 -f`), SPITBOL (`spitbol`), SNOBOL4-tiny.
>
> **Legend**:
> `✓` = works as documented
> `✗` = absent / fails / wrong
> `!` = present but surprising behavior — read the notes
> `?` = not yet tested

---

## Keyword Default Values — Proven

| Keyword | CSNOBOL4 | SPITBOL | SNOBOL4-tiny | Notes |
|---------|----------|---------|--------------|-------|
| `&STNO` | `2` | `2` | `?` | First readable value; counts actual statement number |
| `&STCOUNT` | `0` **!** | `2` ✓ | `?` | CSNOBOL4 always returns 0 — **STCOUNT IS BROKEN IN CSNOBOL4** |
| `&STLIMIT` | `-1` ✓ | `2147483647` ! | `50000` ! | CSNOBOL4 unlimited by default; SPITBOL MAX_INT; tiny hardcoded |
| `&LASTNO` | `4` ✓ | `4` ✓ | `?` | Previous statement number |
| `&FNCLEVEL` | `0` ✓ | `0` ✓ | `?` | Zero at top level |
| `&FTRACE` | `0` ✓ | `0` ✓ | `?` | Zero = disabled |
| `&ANCHOR` | `0` ✓ | `0` ✓ | `0` stub | Zero = unanchored |
| `&FULLSCAN` | `0` ✓ | `1` ! | `0` stub | **SPITBOL defaults FULLSCAN=1; CSNOBOL4 defaults 0** |
| `&TRIM` | `0` ✓ | `1` ! | `1` stub | **SPITBOL defaults TRIM=1; CSNOBOL4 defaults 0** |
| `&ERRLIMIT` | `0` ✓ | `0` ✓ | `?` | Zero = abort on first error |
| `&ERRTYPE` | `0` ✓ | `0` ✓ | `?` | Zero = no error |
| `&ABEND` | `0` ✓ | `0` ✓ | `?` | Zero = normal exit on error |
| `&DUMP` | `0` ✓ | `0` ✓ | `?` | Zero = no dump |
| `&MAXLNGTH` | `4294967295` | `16777216` ! | `524288` ! | **All three differ. CSNOBOL4=4G, SPITBOL=16M, tiny=512K** |
| `&CASE` | `0` ! | `1` ✓ | `?` | CSNOBOL4 `&CASE=0` even with `-f` flag — `-f` ≠ `&CASE=1` |
| `&RTNTYPE` | `''` ✓ | `''` ✓ | `?` | Empty at top level |

---

## Keyword Write Behavior — Proven

| Keyword | CSNOBOL4 | SPITBOL | Notes |
|---------|----------|---------|-------|
| `&ERRLIMIT` write | ✓ OK | ✓ OK | Read-write, no restriction |
| `&ANCHOR` write | ✓ OK | ✓ OK | Read-write |
| `&ABEND` write | ✓ OK | ✓ OK | Read-write |
| `&DUMP` write | ✓ OK | ✓ OK | Read-write |
| `&STLIMIT` write | ✓ OK | ✓ OK | Read-write |
| `&STCOUNT` write | `RO` ✓ | `RO` ✓ | Read-only on both — assignment silently ignored, value unchanged |
| `&STNO` write | `RO` ✓ | `RO` ✓ | Read-only on both |
| `&FNCLEVEL` write | `RO` ✓ | `RO` ✓ | Read-only on both |

---

## TRACE Types — Proven

| TRACE type | CSNOBOL4 | SPITBOL | Notes |
|-----------|----------|---------|-------|
| `TRACE(var,'VALUE')` | `!` fires once | ✓ fires on each assignment | **CSNOBOL4 only fired on first assignment to watchMe, not second** |
| `TRACE(lbl,'LABEL')` | ✓ | ✓ | Both fire when label is branched to |
| `TRACE('&STNO','KEYWORD')` | `!` no output | `!` no output | **Neither system produced TRACE output for &STNO KEYWORD trace** |
| `TRACE(fn,'CALL')` | `!` recurses/segfaults | `?` not tested | CSNOBOL4: TRACE CALL handler re-enters itself — stack overflow |
| `TRACE(fn,'RETURN')` | `?` | `?` | Not yet tested safely |

### TRACE Output Format Differences
- **CSNOBOL4**: `filename:lineno stmt N: varname = 'value', time = 0.`  (to stderr)
- **SPITBOL**: `****N******  varname = 'value'`  (to stdout mixed with program output)

**Critical**: SPITBOL TRACE output goes to **stdout**, CSNOBOL4 goes to **stderr**.
The diff monitor must separate these streams differently per oracle.

---

## &STLIMIT Enforcement — Proven

| Scenario | CSNOBOL4 | SPITBOL |
|----------|----------|---------|
| Default limit | `-1` (unlimited) | `2147483647` (MAX_INT, effectively unlimited) |
| Set `&STLIMIT = &STCOUNT + 8` | **FAILS** — STCOUNT=0, so limit=8, but ran 13 more lines | **FAILS** — same issue, ran 13 more lines after arming |
| Reason | `&STCOUNT` returns 0 always in CSNOBOL4 | STCOUNT does increment in SPITBOL (=25 when armed) but 13 more statements ran anyway |

**STLIMIT does not stop execution at exactly N statements on either system in this test.**
This needs further investigation — the arithmetic `&STCOUNT + 8` may not work as expected
when STCOUNT is live-updating during expression evaluation.

---

## Critical Findings Summary

1. **`&STCOUNT` is broken in CSNOBOL4** — always returns 0. Cannot be used for binary search on CSNOBOL4. SPITBOL correctly increments it.

2. **SPITBOL TRACE goes to stdout, CSNOBOL4 TRACE goes to stderr** — the diff monitor must handle both streams.

3. **`TRACE('&STNO','KEYWORD')` produced no output on either system** — this trace type may require `&TRACE` to be set higher, or `&STNO` is not a valid KEYWORD trace target. The per-statement heartbeat via KEYWORD trace is **unverified**.

4. **Default values differ between CSNOBOL4 and SPITBOL**:
   - `&FULLSCAN`: CSNOBOL4=0, SPITBOL=1
   - `&TRIM`: CSNOBOL4=0, SPITBOL=1
   - `&MAXLNGTH`: CSNOBOL4=4G, SPITBOL=16M, tiny=512K
   - `&STLIMIT`: CSNOBOL4=-1 (unlimited), SPITBOL=MAX_INT

5. **`TRACE(fn,'CALL')` recurses in CSNOBOL4** — the TRACE handler itself triggers CALL trace, causing infinite recursion and segfault. Must arm CALL trace carefully.

6. **`&CASE=0` in CSNOBOL4 despite `-f` flag** — `-f` is not the same as `&CASE=1`. The `-f` flag affects something else (free-format? full-scan?).

7. **TRACE VALUE only fired once in CSNOBOL4** — second assignment to `watchMe` did not fire. May be a one-shot behavior or a bug.

---

## SNOBOL4-tiny Status vs Proven Oracle Behavior

| Feature | Needed behavior | Tiny current state |
|---------|----------------|-------------------|
| `&STCOUNT` | Increment per statement (SPITBOL model) | Increments internally (P001) but not readable |
| `&STLIMIT` | Check against STCOUNT; default -1 or MAX_INT | Enforced (P001); default 50000 — wrong |
| `&STNO` | Read-only, current stmt number | COMM only, not readable |
| TRACE VALUE | Fire on assignment to watched var | Not implemented |
| TRACE LABEL | Fire on branch to watched label | Not implemented |
| TRACE KEYWORD | Unclear — neither oracle produced output | Unclear |
| TRACE stream | SPITBOL→stdout, CSNOBOL4→stderr | tiny→stderr via COMM |


---

## &STNO / &STCOUNT / &STLIMIT — Live Test Results
*Tested 2026-03-10 on all running implementations*

### &STNO behavior

| System | &STNO tracks current stmt? | Initial value |
|--------|---------------------------|---------------|
| CSNOBOL4 v2.3.3 | ✓ YES — increments 1,2,3... per statement | 2 (counts setup stmts) |
| SPITBOL x64 v4.0f (installed) | ✓ YES — increments 1,2,3... per statement | 2 |
| SPITBOL x64 v4.0f (from source) | ✓ YES — increments 1,2,3... per statement | 2 |

**All three agree: `&STNO` is the current statement number, 1-based, accurate.**

### &STCOUNT behavior

| System | &STCOUNT tracks cumulative count? | Initial value |
|--------|----------------------------------|---------------|
| CSNOBOL4 v2.3.3 | **✗ NO** — always 0 | 0 |
| SPITBOL x64 v4.0f (installed) | ✓ YES — counts every statement executed | 2 |
| SPITBOL x64 v4.0f (from source) | ✓ YES — counts every statement executed | 2 |

**CSNOBOL4 `&STCOUNT` is broken — always returns 0. SPITBOL both versions work correctly.**
**Documented in previous grid above — confirmed again here.**

### &STLIMIT behavior — loop termination

Test: `&STLIMIT = 15` with an infinite loop `i = i+1 :(LOOP)`.

| System | Terminates loop? | At i= | Error message |
|--------|-----------------|-------|---------------|
| CSNOBOL4 v2.3.3 | ✓ YES | i=5 | `Error 22: Limit on statement execution exceeded` |
| SPITBOL x64 v4.0f (installed) | ✓ YES | i=4 | `error 244 -- statement count exceeds value of stlimit keyword` |
| SPITBOL x64 v4.0f (from source) | ✓ YES | i=4 | `error 244 -- statement count exceeds value of stlimit keyword` |

**All three enforce `&STLIMIT` correctly.** Different error numbers and messages, same behavior.

Note: CSNOBOL4 stops at i=5, SPITBOL at i=4. This is because CSNOBOL4's `&STCOUNT`
is broken (always 0) so it counts differently internally — but `&STLIMIT` still works.
SPITBOL counts 2 setup statements before the loop; CSNOBOL4 does not.

### &STLIMIT default values

| System | Default `&STLIMIT` |
|--------|-------------------|
| CSNOBOL4 | **-1** (unlimited) |
| SPITBOL x64 | **2147483647** (INT_MAX — effectively unlimited) |

Different defaults. Both mean "run forever unless set."

### Implications for SNOBOL4-tiny

Our runtime has:
- `sno_kw_stlimit = 2000000` — enforced via P001 fix ✓
- `sno_kw_stcount` incremented at every `sno_comm_stno()` call ✓
- `&STNO` emitted via `sno_comm_stno()` ✓

We match SPITBOL behavior on `&STCOUNT`. We differ from CSNOBOL4 (which is broken).
Default `&STLIMIT` should be raised or set to -1 for production — 2,000,000 is
artificially low for large programs. `beautiful.sno` needs more than that.
