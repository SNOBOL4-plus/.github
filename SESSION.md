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
| **HEAD** | `373d939 — feat(pattern-block): byrd_emit_named_pattern + two-pass pre-registration` |

---

## State at handoff (session 57)

Three commits this session:
- `98ec305` — fix(block-fn): correct block grouping — first_block flag ✅
- `373d939` — feat(pattern-block): byrd_emit_named_pattern + two-pass pre-registration ✅

112 named pattern functions compiled for beauty.sno (26197 lines C, 0 gcc errors).
Binary runs (exit 0) but outputs only 10 lines — Parse Error on first statement.

---

## ONE NEXT ACTION — Fix E_COND in byrd_emit for string RHS

**File:** `src/sno2c/emit_byrd.c`

**The bug:** The `~` operator (E_COND) compiles `BREAK(...) ~ 'snoLabel'` wrong.
- `pat->right->kind == E_STR` (the string `'snoLabel'`)
- E_COND case checks `pat->right->kind == E_VAR` → false → uses `"OUTPUT"` as varname
- `emit_cond` then goes wrong, OR the `~` is parsing as E_CONCAT inside emit_pat
  and the E_STR literal becomes a `memcmp` instead of a capture

**Root cause confirmed** by inspecting `pat_snoLabel` output:
```c
if (memcmp(_subj_np + _cur_np, "snoLabel", 8) != 0) goto cat_l_585_beta;
```
It's treating `~ 'snoLabel'` as a literal string match, not a capture.

**The fix — in emit_byrd.c E_COND case (around line 1308):**
```c
case E_COND: {
    /* ~ varname  OR  ~ 'string-as-varname' */
    const char *varname = NULL;
    if (pat->right && pat->right->kind == E_VAR)
        varname = pat->right->sval;
    else if (pat->right && pat->right->kind == E_STR)
        varname = pat->right->sval;   /* 'name' evaluated as varname */
    if (!varname || !*varname) varname = "OUTPUT";
    emit_cond(pat->left, varname,
              alpha, beta, gamma, omega,
              subj, subj_len, cursor, depth);
    return;
}
```

Also check E_IMM ($ operator) — same issue likely exists there.

**Test after fix:**
```bash
cd /home/claude/SNOBOL4-tiny/src/sno2c && make -B

INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
R=/home/claude/SNOBOL4-tiny/src/runtime
SNO=/home/claude/snobol4-install/bin/snobol4

./sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp.c

gcc -O0 -g -I. -I$R -I$R/snobol4 /tmp/beauty_tramp.c \
    $R/snobol4/snobol4.c $R/snobol4/snobol4_inc.c \
    $R/snobol4/snobol4_pattern.c $R/engine_stub.c \
    -lgc -lm -w -o /tmp/beauty_tramp_bin
echo "gcc exit: $?"

# Quick smoke test
printf 'X = 1\n' | /tmp/beauty_tramp_bin

# Full diff
/tmp/beauty_tramp_bin < $BEAUTY > /tmp/beauty_tramp_out.sno
$SNO -f -P256k -I$INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_tramp_out.sno
# Expect: empty → M-BEAUTY-FULL fires
```

**Commit when diff is empty:**
```
feat: M-BEAUTY-FULL — beauty.sno self-beautifies through trampoline compiled binary
```

---

## Artifact convention (mandatory every session touching sno2c/emit*.c)

```bash
# At END of session:
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
mkdir -p artifacts/trampoline_sessionN
./sno2c -trampoline -I$INC $BEAUTY > artifacts/trampoline_sessionN/beauty_tramp_sessionN.c
# Record md5, line count, gcc errors, active bug in artifacts/trampoline_sessionN/README.md
# Commit: artifact: beauty_tramp_sessionN.c — <one-line status>
```

---

## Container Setup (fresh session)

```bash
apt-get install -y m4 libgc-dev
git config user.name "LCherryholmes"
git config user.email "lcherryh@yahoo.com"
TOKEN=TOKEN_SEE_LON
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/.github.git dotgithub

cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
./configure --prefix=/home/claude/snobol4-install && make -j$(nproc) && make install
cd ..

cd SNOBOL4-tiny/src/sno2c && make
```

---

## CRITICAL Rules (no exceptions)

- **NEVER write the token into any file**
- **NEVER link engine.c in beauty_full_bin** — engine_stub.c only
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
| 2026-03-14 | Parse Error root cause found | E_COND ~ 'str' treated as literal match |
