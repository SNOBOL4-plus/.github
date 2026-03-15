## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `compiled-byrd-boxes-full` — Sprint 4 of 6 toward M-BEAUTY-CORE |
| **Milestone** | M-BEAUTY-CORE → M-BEAUTY-FULL |
| **HEAD** | `34489c2` — WIP(sprint4): ~ capture push_val fix — partial, inline push needed in emit_cond |

---

## ⚡ SESSION 97 FIRST ACTION — Fix inline ~ push in emit_cond

Sprint 4 is in progress. Root cause of beauty_full_bin parse failure fully diagnosed.

### The bug

`~` (E_NAM / conditional assignment) in named pattern functions needs to push the
captured value to the nInc stack (`push_val`) inline at the moment of capture —
not deferred to `_PAT_γ`. `Reduce("Stmt", 7)` is called by the CALLER (`pat_Command`)
immediately after `pat_Stmt` returns. The pushes must happen inside `pat_Stmt` before
it returns, and only for the TAKEN alternation branch.

### What was tried (Session 96, commit 34489c2)

Added `push_val(STRVAL(tmpvar))` inside `byrd_cond_emit_assigns` — called at `_Stmt_γ`.
**Wrong because:** pushes ALL cond_ vars (18 items) including epsilon ~ '' from UNTAKEN
alternation branches. `Reduce("Stmt",7)` only pops 7. Stack gets corrupted.

### The correct fix

**In `src/sno2c/emit_byrd.c`, function `emit_cond` (around line 1320):**

1. At `alpha` entry: save stack — emit `DESCR_t save_stk_UID = NV_GET_fn("@S");`
   Add this to `decl_add` so it goes in the struct for named patterns.

2. At `do_capture`: push inline — after copying span into `tmp_var`, emit:
   `push_val(STRVAL(tmp_var));`

3. At `omega` (the backtrack path that bypasses this capture): restore stack —
   `NV_SET_fn("@S", save_stk_UID);`
   This undoes any push from a prior do_capture attempt on backtrack.

4. **Remove** the `push_val` added to `byrd_cond_emit_assigns` — leave it as
   NV_SET_fn only (the variable assignment still matters for semantics).

This matches the standard Gimpel pattern: ~ pushes inline, backtrack restores @S,
Reduce pops exactly what the taken path pushed.

### Key code location

`emit_cond` in `src/sno2c/emit_byrd.c` lines ~1320-1375:
- `do_capture` section: add `push_val(STRVAL(tmp_var));` after the memcpy
- `alpha` section: add stack-save declaration + emit
- `beta`/backtrack: add `NV_SET_fn("@S", save_stk_UID);` to undo pushes

### Verification

After fix, regenerate and test:
```bash
cd /home/claude/SNOBOL4-tiny
make -C src/sno2c
RT=src/runtime
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > beauty_full.c
gcc -O0 -g beauty_full.c $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o beauty_full_bin

# Expected: "               OUTPUT         =  'hello'"
echo "        OUTPUT = 'hello'" | ./beauty_full_bin

# Oracle comparison:
echo "        OUTPUT = 'hello'" | snobol4 -f -P256k -I$INC $BEAUTY

# Crosscheck must still be 106/106:
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh
```

### Build command
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

### Session start checklist
```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -5
# Verify HEAD = 34489c2
make -C src/sno2c
apt-get install -y libgc-dev  # if needed
# snobol4 is installed at /usr/local/bin/snobol4
```

### Pivot log
- Session 95: Sprint 3 complete. 106/106 crosscheck.
- Session 96: Sprint 4 diagnosis. Root cause: ~ capture does not push to nInc stack.
  byrd_cond_emit_assigns push_val attempt PARTIAL — wrong timing/count. Fix spec above.
  Crosscheck symlink: ln -sf /home/SNOBOL4-corpus /home/SNOBOL4-corpus (need /home/SNOBOL4-corpus/crosscheck → /home/claude/SNOBOL4-corpus/crosscheck for run_crosscheck.sh).
  Also: run_crosscheck.sh expects SNOBOL4-tiny and SNOBOL4-corpus as siblings two levels up.
  Workaround used: `ln -sf /home/claude/SNOBOL4-corpus/crosscheck /home/SNOBOL4-corpus/crosscheck`
