## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `crosscheck-ladder` rung 12 — beauty.sno unit tests |
| **Milestone** | M-BEAUTY-CORE → M-BEAUTY-FULL |
| **HEAD** | `4bd9050` — Revert WIP push_val (back to clean 668ce4f baseline) |

---

## ⚡ SESSION 98 FIRST ACTION — Write rung 12 crosscheck tests, run, fix

### PIVOT (Session 97)

Sprint 4 (`compiled-byrd-boxes-full`) was doing compiler internals work
disconnected from any test program. **Retired.** Lon's rule: test-driven only.

The WIP push_val commit (34489c2) has been reverted. We are back to 668ce4f baseline:
106/106 crosscheck passing, clean compiler.

### New sprint: rung 12 crosscheck tests

**The ladder rule:** add tests to `SNOBOL4-corpus/crosscheck/beauty/`, run them
against `sno2c`-compiled `beauty_full_bin`, diff against CSNOBOL4 oracle.
Fix compiler failures one test at a time. Never skip ahead.

### Session start checklist
```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3
# Verify HEAD = 4bd9050
apt-get install -y libgc-dev
make -C src/sno2c
# snobol4 oracle at /usr/local/bin/snobol4
```

### How rung 12 tests work

Each test: compile `beauty.sno` (with `-I inc/`) via `sno2c`, pipe a small
SNOBOL4 source fragment as stdin, diff output against CSNOBOL4 oracle.

The binary to test IS `beauty_full_bin` — already compiled from beauty.sno.
Tests live in `SNOBOL4-corpus/crosscheck/beauty/`.

**Build beauty_full_bin:**
```bash
cd /home/claude/SNOBOL4-tiny
RT=src/runtime
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > beauty_full.c
gcc -O0 -g beauty_full.c $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o beauty_full_bin
```

**Generate oracle for a test input:**
```bash
echo "* comment" | snobol4 -f -P256k -I$INC $BEAUTY
```

**Run compiled binary:**
```bash
echo "* comment" | ./beauty_full_bin
```

### First tests to write (rung 12, start minimal)

Write these in `SNOBOL4-corpus/crosscheck/beauty/`:

```
101_beauty_comment.sno/.ref/.input     — input: "* a comment"
102_beauty_output_stmt.sno/.ref/.input — input: "        OUTPUT = 'hello'"
103_beauty_assign.sno/.ref/.input      — input: "        X = 'foo'"
104_beauty_label.sno/.ref/.input       — input: "LOOP    X = X 1"
105_beauty_goto.sno/.ref/.input        — input: "        :(END)"
```

Each .sno just pipes the .input through beauty_full_bin.
Each .ref is the oracle output from CSNOBOL4.

### Run crosscheck for rung 12
```bash
# After adding beauty/ dir to SNOBOL4-corpus/crosscheck:
# Add "beauty" to the dir list in run_crosscheck.sh, OR run manually:
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh
```

### Crosscheck symlink fix (needed every session)
```bash
mkdir -p /home/SNOBOL4-corpus
ln -sf /home/claude/SNOBOL4-corpus/crosscheck /home/SNOBOL4-corpus/crosscheck
```

### Pivot log
- Sessions 80-89: attacked beauty.sno directly, burned sessions chasing bugs
- Session 89: pivot to corpus ladder (rung 1-11)
- Session 95: Sprint 3 complete, 106/106 rungs 1-11
- Sessions 96-97: Sprint 4 compiler internals — RETIRED (not test-driven)
- Session 97: PIVOT back to test-driven. Rung 12 beauty.sno tests next.
