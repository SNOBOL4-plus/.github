## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-crosscheck` — Sprint A of 4 — rung 12 crosscheck tests |
| **Milestone** | M-BEAUTY-CORE → M-BEAUTY-FULL |
| **HEAD** | `4bd9050` — Revert WIP push_val (back to clean 668ce4f baseline) |

---

## ⚡ SESSION 99 FIRST ACTION — Build beauty_full_bin, write rung 12 tests, run Sprint A

### Four-Sprint Plan (Session 98, decided 2026-03-15)

**See PLAN.md §"Session 98 — Four-Paradigm TDD Plan"** for full detail.

Sprint A: **Crosscheck** — corpus diff tests for beauty.sno (rung 12)  
Sprint B: **Probe** — &STLIMIT frame-by-frame for failing tests  
Sprint C: **Monitor** — TRACE double-trace diff for deep recursion bugs  
Sprint D: **Triangulate** — cross-engine CSNOBOL4 vs compiled, full self-beautify  

**The goal:** `diff oracle_csn.txt compiled_out.txt` is empty. M-BEAUTY-FULL fires.

**The invariant:** 106/106 rungs 1–11 must pass after every commit.

### Session start checklist

```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3
# Verify HEAD = 4bd9050

apt-get install -y libgc-dev
make -C src/sno2c

# Invariant check
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh
# Must be 106/106

# Symlink
mkdir -p /home/SNOBOL4-corpus
ln -sf /home/claude/SNOBOL4-corpus/crosscheck /home/SNOBOL4-corpus/crosscheck

# Build beauty_full_bin
RT=src/runtime
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > beauty_full.c
gcc -O0 -g beauty_full.c $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o beauty_full_bin
```

### Sprint A — First action

1. Create `SNOBOL4-corpus/crosscheck/beauty/` directory
2. Write `101_beauty_comment.input` + generate `.ref` from CSNOBOL4 oracle
3. Write `SNOBOL4-tiny/test/crosscheck/run_beauty.sh` (pre-compiled binary runner)
4. Run: `bash test/crosscheck/run_beauty.sh`
5. If PASS → write 102, 103, 104... escalate
6. If FAIL → drop to probe.py (Paradigm 2) to locate the statement

### Pivot log

- Sessions 80–89: attacked beauty.sno directly — burned chasing bugs
- Session 89: pivot to corpus ladder (rungs 1–11)
- Session 95: Sprint 3 complete, 106/106 rungs 1–11 ✅
- Sessions 96–97: Sprint 4 compiler internals — RETIRED (not test-driven)
- Session 97: PIVOT — test-driven only
- Session 98: PIVOT — four-paradigm TDD plan written to PLAN.md
- Session 99: Sprint A begins — rung 12 crosscheck
