# TESTING.md — Four-Paradigm TDD Protocol

**The goal:** `beauty_full_bin` reads `beauty.sno`, diff vs CSNOBOL4 oracle is empty. **M-BEAUTY-FULL.**
**The invariant:** 106/106 rungs 1–11 pass after every commit. Regression = rollback.

---

## The Corpus Ladder (all repos — TINY, JVM, DOTNET)

```
Rung 1:  output      — OUTPUT = 'hello'
Rung 2:  assign      — X = 'foo', null assign
Rung 3:  concat      — OUTPUT = 'a' 'b'
Rung 4:  arith       — OUTPUT = 1 + 2
Rung 5:  control     — goto, :S(), :F()
Rung 6:  patterns    — LIT, ANY, SPAN, ARB, ARBNO, POS, RPOS
Rung 7:  capture     — . and $ operators
Rung 8:  strings     — SIZE, SUBSTR, REPLACE, DUPL
Rung 9:  keywords    — IDENT, DIFFER, GT/LT/EQ, DATATYPE
Rung 10: functions   — DEFINE, RETURN, FRETURN, recursion
Rung 11: data        — ARRAY, TABLE, DATA types
Rung 12: beauty.sno  — tiny inputs → full self-beautification
```
Rule: stop at first failing rung. Fix. Retest. Never skip.

---

## Four Paradigms

### Paradigm 1 — Crosscheck (corpus diff)
What: compile .sno → binary, run, diff output vs .ref oracle.
Catches: wrong output — any observable bug.
Tool: `test/crosscheck/run_crosscheck.sh` (rungs 1–11), `test/crosscheck/run_beauty.sh` (rung 12).
```bash
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh   # must be 106/106
bash test/crosscheck/run_beauty.sh                       # rung 12
```

### Paradigm 2 — Probe (&STLIMIT frame-by-frame)
What: run program N times at &STLIMIT=1..N with &DUMP=2. Show what changed each step.
Catches: exactly WHERE divergence first appears — which variable, which statement.
Tool: `SNOBOL4-harness/probe/probe.py`
```bash
python3 /home/claude/SNOBOL4-harness/probe/probe.py --oracle csnobol4 --max 200 failing.sno
```
When to use: Paradigm 1 finds failure → Paradigm 2 locates statement.

### Paradigm 3 — Monitor (TRACE double-trace diff)
What: TRACE('fn','CALL'/'RETURN'/'VALUE') hooks in beauty.sno. Both oracle and compiled
emit same event stream. Diff stream event by event. First divergence = root cause.
Tool: `SNOBOL4-corpus/programs/beauty/beauty_trace.sno` + `test/crosscheck/monitor_beauty.sh`
```bash
snobol4 -f -P256k -I$INC beauty_trace.sno < input.sno 2>oracle_trace.txt
./beauty_full_bin_trace < input.sno 2>compiled_trace.txt
diff oracle_trace.txt compiled_trace.txt | head -20
```
When to use: Paradigm 2 finds divergence in recursion → Paradigm 3 shows call/return stream.

### Paradigm 4 — Triangulate (cross-engine)
What: same program through CSNOBOL4 + SPITBOL + compiled. Two oracles agree, compiled
differs → our bug. Two oracles disagree → semantic edge case, check Gimpel §7.
Tool: `test/crosscheck/triangulate_beauty.sh`
```bash
snobol4 -f -P256k -I$INC $BEAUTY < $BEAUTY > oracle_csn.txt
./beauty_full_bin < $BEAUTY > compiled_out.txt
diff oracle_csn.txt compiled_out.txt   # empty = M-BEAUTY-FULL
```
Note: SPITBOL excluded from full beauty.sno (error 021 at END). CSNOBOL4 is primary oracle.

---

## Sprint Map to M-BEAUTY-FULL

| Sprint | Paradigm | Milestone trigger |
|--------|----------|------------------|
| `monitor-scaffold` | Monitor | runner + inject_traces.py, 1 test passing |
| `monitor-value` | Monitor | assign/ + concat/ 14/14 → |
| `monitor-control` | Monitor | control_new/ 7/7 → |
| `monitor-patterns` | Monitor | patterns/ + capture/ 27/27 → |
| `monitor-functions` | Monitor | functions/ 8/8 → |
| `monitor-data` | Monitor | data/ + strings/ 17/17 → |
| `monitor-keywords` | Monitor | keywords/ 10/10 → |
| `monitor-full` | Monitor | all 152 corpus tests zero diffs → **M-MONITOR** |
| `beauty-crosscheck` | Crosscheck | beauty/140_self passes → **M-BEAUTY-CORE** |
| `beauty-triangulate` | Triangulate | Empty diff → **M-BEAUTY-FULL** |

## Rung 12 Test Format

Tests live in `SNOBOL4-corpus/crosscheck/beauty/`:
- `NNN_name.input` — SNOBOL4 snippet to pipe to beauty_full_bin
- `NNN_name.ref` — oracle output: `snobol4 -f -P256k -I$INC $BEAUTY < NNN_name.input`

Test progression: 101_comment → 102_output → 103_assign → 104_label → 105_goto →
109_multi → 120_real_prog → 130_inc_file → 140_self (M-BEAUTY-CORE).

## Sprint: `oracle-verify` — Verify the Keyword Grid (Session 124)

**Goal:** Every `?` and every unverified cell in the keyword grid below becomes ✅ or ❌, confirmed by live test. Every oracle must have ≥1 working probe statement counter.

**Deliverables:**
1. CSNOBOL4 built from source (tarball already uploaded)
2. SPITBOL x64 built from source (needs `x64-main.zip` upload)
3. SNOBOL5 located, installed if available, or documented as unavailable
4. `oracles/verify.sno` — single test program that probes all keywords and emits a result line per keyword
5. All `?` cells in the grid below replaced with live-tested ✅ or ❌
6. `&STEXEC` tested on CSNOBOL4 as substitute for broken `&STCOUNT`
7. SNOBOL5 probe counter situation resolved: `&STNO`, `&LASTNO`, or neither?
8. Commit to SNOBOL4-harness with updated grid

**verify.sno — probe program:**
```snobol4
*       verify.sno — oracle keyword verification
*       Run: snobol4 -f verify.sno  (or spitbol -b, or snobol5)
*       Each line of output: KEYWORD = value  OR  KEYWORD = FAIL

        &STLIMIT = 100000

*       &STCOUNT
        X = &STCOUNT
        OUTPUT = '&STCOUNT = ' X

*       &STEXEC (CSNOBOL4 extension)
        X = &STEXEC                                         :F(NO_STEXEC)
        OUTPUT = '&STEXEC = ' X                             :(DONE_STEXEC)
NO_STEXEC
        OUTPUT = '&STEXEC = FAIL'
DONE_STEXEC

*       &STNO
        X = &STNO                                           :F(NO_STNO)
        OUTPUT = '&STNO = ' X                               :(DONE_STNO)
NO_STNO
        OUTPUT = '&STNO = FAIL'
DONE_STNO

*       &LASTNO
        X = &LASTNO                                         :F(NO_LASTNO)
        OUTPUT = '&LASTNO = ' X                             :(DONE_LASTNO)
NO_LASTNO
        OUTPUT = '&LASTNO = FAIL'
DONE_LASTNO

*       &DUMP=2 — tested by checking &DUMP is writable
        &DUMP = 2
        OUTPUT = '&DUMP = ' &DUMP

*       &ANCHOR, &TRIM, &FULLSCAN defaults
        OUTPUT = '&ANCHOR = ' &ANCHOR
        OUTPUT = '&TRIM = ' &TRIM
        OUTPUT = '&FULLSCAN = ' &FULLSCAN

        END
```

**Pass condition:** every keyword row in the grid has a live result. No `?` remaining. Each oracle has ≥1 cell in {`&STCOUNT`, `&STEXEC`, `&STNO`, `&LASTNO`} that returns a non-zero-always value.

**Build steps** (see `oracles/csnobol4/BUILD.md` and `oracles/spitbol/BUILD.md`):
```bash
# CSNOBOL4 — tarball already at /mnt/user-data/uploads/snobol4-2_3_3_tar.gz
apt-get install -y build-essential libgmp-dev m4
mkdir -p /home/claude/csnobol4-src
tar xzf /mnt/user-data/uploads/snobol4-2_3_3_tar.gz -C /home/claude/csnobol4-src/ --strip-components=1
cd /home/claude/csnobol4-src
sed -i '/if (!chk_break(0))/{N;/goto L_INIT1;/d}' snobol4.c isnobol4.c
./configure --prefix=/usr/local && make -j4 && make install

# SPITBOL x64 — needs x64-main.zip uploaded by Lon
apt-get install -y nasm
unzip -q /mnt/user-data/uploads/x64-main.zip -d /home/claude/spitbol-src/
# apply systm.c patch from oracles/spitbol/BUILD.md
cd /home/claude/spitbol-src/x64-main && make && cp sbl /usr/local/bin/spitbol

# SNOBOL5 — prebuilt binary, no build required
wget -O /usr/local/bin/snobol5 https://snobol5.org/snobol5
chmod +x /usr/local/bin/snobol5
# verify: echo "OUTPUT = 'hello'" | snobol5

# SPITBOL x32 — our fork (not yet built; 32-bit not runnable in this container)
# https://github.com/SNOBOL4-plus/x32  (forked from hardbol/spitbol)
```

---

## Oracle Keyword & TRACE Reference

Every cell proven by live test on 2026-03-10. SPITBOL-x32 not runnable in container (32-bit execution disabled) — values inferred from source.

### Keywords

| Keyword | CSNOBOL4 | SPITBOL-x64 | SPITBOL-x32 | SNOBOL5 | Use for portability |
|---------|:--------:|:-----------:|:-----------:|:-------:|---------------------|
| `&STLIMIT` | ✅ -1 (unlimited) | ✅ MAX_INT | ✅ (inferred) | ✅ | ✅ primary probe/abort tool |
| `&STCOUNT` | ❌ **always 0** | ✅ increments | ✅ (inferred) | ✅ | ⚠️ use `&STEXEC` or avoid on CSNOBOL4 |
| `&STEXEC` | ❓ unverified | ❌ (CSNOBOL4-only) | ❌ | ❌ | ⚠️ CSNOBOL4-only if it works — needs live test |
| `&STNO` | ✅ | ❌ | ❌ | ? | ❌ CSNOBOL4-only; use `&LASTNO` elsewhere |
| `&LASTNO` | ❌ | ✅ | ✅ (inferred) | ? | ❌ not portable either; avoid |
| `&DUMP=2` fires at `&STLIMIT` | ✅ | ✅ | ? | ✅ | ✅ safe to use |
| `&ANCHOR` default | 0 | **1** | **1** | ? | ⚠️ set explicitly — defaults differ |
| `&TRIM` default | 0 | **1** | **1** | ? | ⚠️ set explicitly — defaults differ |
| `&FULLSCAN` default | 0 | 1 | 1 | ? | ⚠️ set explicitly |
| `&CASE` | 0 even with `-f` | 0 | 0 | ? | `-f` ≠ `&CASE=1` in CSNOBOL4 |
| `&MAXLNGTH` | 4G | 16M | 16M | 64-bit | ⚠️ all differ |
| TRACE output stream | stderr | **stdout** | stdout | stderr | ⚠️ redirect per oracle |

### TRACE types

| TRACE call | CSNOBOL4 | SPITBOL-x64 | SPITBOL-x32 | SNOBOL5 | Use for portability |
|-----------|:--------:|:-----------:|:-----------:|:-------:|---------------------|
| `TRACE(var,'VALUE')` | ✅ | ✅ | ✅ (inferred) | ✅ | ✅ primary monitor tool |
| `TRACE(fn,'CALL')` | ✅ | ✅ | ✅ (inferred) | ✅ | ✅ |
| `TRACE(fn,'RETURN')` | ✅ | ✅ | ✅ (inferred) | ✅ | ✅ |
| `TRACE(fn,'FUNCTION')` | ✅ | ✅ | ✅ (inferred) | ✅ | ✅ |
| `TRACE(label,'LABEL')` | ✅ | ✅ | ✅ (inferred) | ✅ | ✅ |
| `TRACE('STCOUNT','KEYWORD')` | ✅ | ✅ | ? | ✅ | ✅ portable per-statement trace |
| `TRACE('STNO','KEYWORD')` | ✅ at `BREAKPOINT(n,1)` stmts only | ❌ error 198 | ❌ | ❌ silent | ❌ CSNOBOL4-only, avoid |
| `TRACE(...,'KEYWORD')` (general) | non-functional | error 198 | error 198 | ? | ❌ never use |

### TRACE output format

| Oracle | Format |
|--------|--------|
| CSNOBOL4 | `file:LINE stmt N: EVENT, time = T.` |
| SPITBOL-x64 | `****N*******  event` |
| SNOBOL5 | `    STATEMENT N: EVENT,TIME = T` |

Monitor pipe reader must normalize per oracle — all carry statement number and event description.

---

## Session Start Checklist

```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
apt-get install -y libgc-dev && make -C src/sno2c
mkdir -p /home/SNOBOL4-corpus
ln -sf /home/claude/SNOBOL4-corpus/crosscheck /home/SNOBOL4-corpus/crosscheck
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh   # must be 106/106 before any work

# Build beauty_full_bin
RT=src/runtime; INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > beauty_full.c
gcc -O0 -g beauty_full.c $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o beauty_full_bin
```
