# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-first` — fix $expr indirect read bug → M-BEAUTY-FULL |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `0113d90 — fix(emit_cnode): pat_lit strv() wrapper removed` |

---

## ⚡ SESSION 78 FIRST PRIORITY: Fix $expr indirect read bug

### The one job this session:
1. Fix E_DEREF read in `emit.c` + `emit_cnode.c` — one-liner each
2. Rebuild, verify `$name = r` → `DATATYPE(r)=tree` in compiled binary
3. Run full self-beautify diff
4. Fix every diff line until diff is empty
5. Commit: **M-BEAUTY-FULL fires**

---

## Bug: $expr indirect variable read generates deref(NULL_VAL)

### Root cause — confirmed session 78

Grammar rule (sno.y line 210):
```
DOLLAR unary_expr  →  binop(E_DEREF, NULL, $2)
```
So `$X` → `E_DEREF(left=NULL, right=X)`. Operand is in **right**.

`emit_expr` E_DEREF case (~line 292 of emit.c):
```c
E("deref("); emit_expr(e->left); E(")");
//                      ^^^^^^ WRONG — left is NULL for $X
```

**The fix — emit.c ~line 292:**
```c
/* $X → E_DEREF(left=NULL, right=X); *X → E_DEREF(left=NULL, right=X) same grammar
 * Use right when left is NULL */
Expr *_operand = e->left ? e->left : e->right;
E("deref("); emit_expr(_operand); E(")");
```

**Also check emit_cnode.c `build_expr` E_DEREF — same left/right issue likely there.**

### Verification test:
```c
cat > /tmp/test_indirect.sno << 'SNOEOF'
-INCLUDE 'global.sno'
-INCLUDE 'tree.sno'
    r = tree('Leaf', 'hello', 0,)
    name = '@S'
    $name = r
    OUTPUT = 'DATATYPE($name)=' DATATYPE($name)
SNOEOF
# Oracle should print: DATATYPE($name)=tree
# Before fix compiled prints nothing (assignment silently fails)
# After fix compiled should print: DATATYPE($name)=tree
```

### Exact grep to find the line:
```bash
grep -n "emit_expr(e->left)" /home/claude/work/SNOBOL4-tiny/src/sno2c/emit.c
grep -n "e->left\|e->right" /home/claude/work/SNOBOL4-tiny/src/sno2c/emit_cnode.c | grep -i "deref\|E_DEREF"
```

---

## Build command (every session)

```bash
# Install SNOBOL4 interpreter
cd /tmp && tar xzf /mnt/user-data/uploads/snobol4-2_3_3_tar.gz
cd /tmp/snobol4-2.3.3 && ./configure --prefix=/usr/local && make -j$(nproc) && make install
apt-get install -y libgc-dev m4

TOKEN=TOKEN_SEE_LON
# Repos already in /home/claude/work/ if container is warm
# Otherwise clone:
# git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-tiny /home/claude/work/SNOBOL4-tiny
# git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-corpus /home/claude/work/SNOBOL4-corpus
# git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/.github /home/claude/work/.github

cd /home/claude/work/SNOBOL4-tiny
git remote set-url origin https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
cd /home/claude/work/SNOBOL4-tiny/src/sno2c && make

INC=/home/claude/work/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/work/SNOBOL4-corpus/programs/beauty/beauty.sno
RT=/home/claude/work/SNOBOL4-tiny/src/runtime
SNO2C=/home/claude/work/SNOBOL4-tiny/src/sno2c

$SNO2C/sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp.c
gcc -O0 -g -I$SNO2C -I$RT -I$RT/snobol4 \
    /tmp/beauty_tramp.c $RT/snobol4/snobol4.c $RT/snobol4/snobol4_inc.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine.c -lgc -lm -w -o /tmp/beauty_tramp_bin

# Oracle
snobol4 -f -P256k -I$INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno
# Compiled
/tmp/beauty_tramp_bin < $BEAUTY > /tmp/beauty_compiled.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_compiled.sno
```

---

## Session 77 what was done

| Step | Result |
|------|--------|
| Build env | snobol4 installed, SNOBOL4-tiny/corpus cloned |
| Compile bug | `pat_lit(strv("..."))` → fixed to `pat_lit("...")` in emit_cnode.c build_pat E_STR |
| Commit | `0113d90` |
| Artifact | `beauty_tramp_session77.c` — 31773 lines, 0 errors, CHANGED |
| START test | ✅ now outputs `START` correctly |
| Full diff | oracle=162 lines, compiled=10 lines — stops after header+START |
| Root cause | `$'@S'` read → `deref(NULL_VAL)` instead of `deref(strv("@S"))` |
| Root cause | emit_expr E_DEREF uses `e->left` but grammar puts operand in `e->right` |

---

## CRITICAL Rules

- **NEVER write the token into any file**
- **NEVER link engine.c in beauty_full_bin** (trampoline path uses engine.c — that's OK for now)
- **ALWAYS run `git config user.name/email` after every clone**

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-15 | 3-column format `d5b9c3c` | emit_pretty.h shared |
| 2026-03-15 | M-CNODE CNode IR `160f69b`+`ac54bd2` | proper pp/qq architecture |
| 2026-03-15 | Return to M-BEAUTY-FULL | M-CNODE done, back to main line |
| 2026-03-14 | `0113d90` pat_lit fix | emit_cnode.c build_pat E_STR strv() removed |
