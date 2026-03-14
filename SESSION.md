# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `pattern-block` (sprint 4/9 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `50ef58f — fix(emit+trampoline): DATA tree/link startup + trampoline_stno for &STLIMIT` |

---

## State at handoff (session 68)

Commits this session:
- `50ef58f` — fix(emit+trampoline): DATA tree/link startup + trampoline_stno for &STLIMIT

**CSNOBOL4 2.3.3 now installed at `/usr/local/bin/snobol4`** — oracle available every session.
Build: upload `snobol4-2_3_3_tar.gz`, extract, `./configure && make && make install`.
Or re-download: `snobol4` is Phil Budne's CSNOBOL4B 2.3.3.

### Fix 4 — emit.c: DATA('tree') and DATA('link') in dead code
`DATA('tree(t,v,n,c)')` from `tree.sno` was swallowed into `_sno_fn_Top` body
by the StackEnd boundary walk — never executed at startup. `tree()` constructor
unregistered → `Reduce('Parse', 0)` called `tree(...)` → got NULL_VAL → `Push(NULL_VAL)`
→ `Pop()` returned null → `DIFFER(sno = Pop())` failed → **Internal Error**.

Fix: emit `aply("DATA",{STR_VAL("tree(t,v,n,c)")},1)` and
`aply("DATA",{STR_VAL("link(next,value)")},1)` explicitly in `main()` before
`trampoline_run()`. Root cause (fn-body-walk swallowing tree.sno init) not yet fixed
in the emitter — tracked as a known issue.

### Fix 5 — trampoline.h + emit.c: &STLIMIT / &STCOUNT wired
`trampoline_stno(lineno)` now emitted at top of every stmt_N.
`trampoline.h` externs `kw_stlimit`/`kw_stcount` from `snobol4.c`.
Set `kw_stlimit` via constructor object to impose statement limit for hang diagnosis.

### Test results (session 68)
| Input | Compiled binary | Oracle (CSNOBOL4) |
|-------|----------------|-------------------|
| `* comment` | ✅ `* comment` | ✅ same |
| `START` | ✅ silent exit | ✅ silent exit |
| `X = 1` | ❌ infinite loop at stmt lineno=107 | ✅ `Parse Error` |

### X = 1 hang — diagnosed to stmt lineno=107
With `kw_stlimit=100`, the binary hits `** &STLIMIT exceeded at statement 107
(&STCOUNT=101 &STLIMIT=100)`. Spins at lineno=107 — tight infinite backtrack loop.
Multiple source files have line 107; need to identify which one is the hot loop.

---

## ONE NEXT ACTION

```bash
# Step 1: Identify the looping statement
grep -n "stno(107)" /tmp/beauty_tramp.c | head -20
# Each hit is a different included file's line 107.
# To find which one is hot, add a counter:
cat > /tmp/stlimit_10.c << 'EOF'
#include <stdint.h>
extern int64_t kw_stlimit;
__attribute__((constructor)) static void set_limit(void) { kw_stlimit = 10; }
EOF
# Compile with that, run X=1, look at the stno value — will be 107.
# Then grep for context around each line 3 hits of stno(107):
grep -n "trampoline_stno(107)" /tmp/beauty_tramp.c
# Read 20 lines before each hit to find which stmt/block function it's in.
# The one that's inside a pattern-matching loop (Command, Space, Parse)
# is the infinite-backtrack culprit.

# Step 2: Oracle confirms X = 1 → Parse Error (not a hang):
INC=/tmp/snobol4-corpus/programs/inc
BEAUTY=/tmp/snobol4-corpus/programs/beauty/beauty.sno
printf 'X = 1\n' | snobol4 -f -P256k -I $INC $BEAUTY
# Expected: Parse Error\nX = 1

# Step 3: Once identified, check what beauty.sno's pattern does at that
# line — likely Space or Command failing to backtrack to omega correctly.
# The fix will be in emit_byrd.c or the pattern wiring for that node type.
```

---

## Build command

```bash
# CSNOBOL4 oracle (installed):
INC=/tmp/snobol4-corpus/programs/inc
BEAUTY=/tmp/snobol4-corpus/programs/beauty/beauty.sno
printf 'INPUT\n' | snobol4 -f -P256k -I $INC $BEAUTY

# Compiled binary:
R=/home/claude/SNOBOL4-tiny   # adjust to container path
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
RT=$R/src/runtime
SNO2C=$R/src/sno2c
cd $SNO2C && make
$SNO2C/sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp.c
gcc -O0 -g -I$SNO2C -I$RT -I$RT/snobol4 \
    /tmp/beauty_tramp.c \
    $RT/snobol4/snobol4.c $RT/snobol4/snobol4_inc.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine.c \
    -lgc -lm -w -o /tmp/beauty_tramp_bin
printf '* comment\n' | /tmp/beauty_tramp_bin
printf 'START\n'     | /tmp/beauty_tramp_bin
printf 'X = 1\n'     | timeout 5 /tmp/beauty_tramp_bin
```

---

## STLIMIT hang diagnosis helper

```bash
# Build with statement limit to catch infinite loops:
cat > /tmp/stlimit.c << 'EOF'
#include <stdint.h>
extern int64_t kw_stlimit;
__attribute__((constructor)) static void set_limit(void) { kw_stlimit = 200; }
EOF
RT=$R/src/runtime; SNO2C=$R/src/sno2c
gcc -O0 -g -I$SNO2C -I$RT -I$RT/snobol4 \
    /tmp/beauty_tramp.c /tmp/stlimit.c \
    $RT/snobol4/snobol4.c $RT/snobol4/snobol4_inc.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine.c \
    -lgc -lm -w -o /tmp/beauty_tramp_limited
printf 'X = 1\n' | /tmp/beauty_tramp_limited 2>&1
# Reports: ** &STLIMIT exceeded at statement NNN
# Then: grep -n "trampoline_stno(NNN)" /tmp/beauty_tramp.c
# Read context around each hit to find the loop.
```

---

## Artifact convention (mandatory every session touching sno2c/emit*.c)

Session 68 artifact: `artifacts/beauty_tramp_session68.c`
- Lines: 29926
- MD5:   3e070e50a1936983f91f1d5064ece1de
- Compile: 0 errors
- Tests: comment ✅  START ✅  X=1 ❌ (infinite loop at stmt 107)

Next artifact: `beauty_tramp_session69.c`

---

## Container Setup (fresh session)

```bash
apt-get install -y m4 libgc-dev
# CSNOBOL4 — upload snobol4-2_3_3_tar.gz then:
tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
./configure --prefix=/usr/local && make -j$(nproc) && make install
cd /tmp
TOKEN=TOKEN_SEE_LON
git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-tiny /home/claude/SNOBOL4-tiny
git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-corpus /home/claude/SNOBOL4-corpus
git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/.github /home/claude/snobol4-hq
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
cd src/sno2c && make
```

---

## CRITICAL Rules (no exceptions)

- **NEVER write the token into any file**
- **NEVER link engine.c in beauty_full_bin** — engine_stub.c only (beauty_tramp_bin still uses engine.c — ok for now)
- **ALWAYS run `git config user.name/email` after every clone**
- Read PLAN.md fully before coding

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-14 | M-TRAMPOLINE fired `fb4915e` | trampoline.h + 3 POC files |
| 2026-03-14 | M-STMT-FN fired `4a6db69` | trampoline emitter in sno2c, beauty 0 gcc errors |
| 2026-03-14 | block grouping bug fixed `98ec305` | first_block flag |
| 2026-03-14 | pattern-block sprint `373d939` | 112 named pat fns, 0 gcc errors |
| 2026-03-14 | E_COND/E_IMM E_STR fix `6d09bfa` | binary compiles, runs, fails on static re-entrancy |
| 2026-03-14 | beauty.sno snoXXX→XXX `d504d80` | oracle now self-referential |
| 2026-03-14 | Technique 1 struct-passing `a3ea9ef` | re-entrancy fixed, 0 gcc errors |
| 2026-03-14 | emit_imm var_set fix `dc8ad4b` | bare labels pass |
| 2026-03-14 | Greek watermark α/β/γ/ω `f74a384` | Lon's branding |
| 2026-03-14 | Three-column pretty layout `e00f851` | Lon's watermark layout |
| 2026-03-14 | Binary ~ fix + wrap fix `06f4715` | START clean, X=1 segfaults |
| 2026-03-14 | Compile named pats from fn bodies `6467ff2` | 196 compiled pats |
| 2026-03-14 | E_DEREF + sideeffect + C-static `09e5a5d` `5e90712` | 33→9 match_pattern_at |
| 2026-03-15 | E_VAR implicit deref `bc8a520` | infinite ARBNO loop eliminated |
| 2026-03-15 | ~ emits Shift() `70e5d89` | 48 Shift calls wired; Internal Error persisted |
| 2026-03-15 | parse_expr13 E_COND fix `f05d3c4` | 7 Shifts now fire for START; * comment passes |
| 2026-03-15 | nl/tab init + nTop() fix + nPush β `5d0e584` | 3 bugs fixed; untested — container crashed |
| 2026-03-15 | DATA tree/link startup + &STLIMIT `50ef58f` | START passes; X=1 loops at stmt 107 |
