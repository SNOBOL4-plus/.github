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
| **HEAD** | `6d09bfa — fix(emit_byrd): E_COND/E_IMM accept E_STR varname + sanitize special chars` |

---

## State at handoff (session 58)

Commits this session:
- `6d09bfa` (TINY) — E_COND/E_IMM E_STR varname fix + sanitize special chars ✅
- `9efd628` (CORPUS) — restore full 801-line beauty.sno after truncated re-beautify ✅
- `d504d80` (CORPUS) — rename snoXXX → XXX (42 names, 207 lines) ✅

Also: beauty.sno self-beautified itself (beautifier bootstrap moment noted).

**Current state:** Binary compiles (gcc 0 errors), runs exit 0, outputs comments only.
Parse Error on first real statement (START, X=1, anything non-comment).

**Root cause pinned:** `pat_Stmt` and all named pattern functions use `static` local
variables. Static locals are shared across all calls — re-entrant calls (e.g.
`*Stmt` inside `*Command` inside `*Parse`) stomp each other's saved cursors.
This is the fundamental re-entrancy bug. The E_COND fix was correct and did unblock
the previous blocker — the new blocker is the static locals issue.

---

## ONE NEXT ACTION — Fix static re-entrancy in named pattern functions

**The bug:** Every `pat_Xxx` function declares its locals as `static`:
```c
static int64_t deref_589_saved_cur;   // ← shared across ALL calls to pat_Stmt
```
When `pat_Stmt` calls `pat_Label` which calls back into `pat_Stmt` (or any
re-entrant path), the inner call overwrites the outer call's saved cursor.

**The fix — Technique 1 (struct-passing) from PLAN.md:**

Each named pattern function gets a locals struct. The struct is heap-allocated
on first call (entry==0) and threaded through re-entry (entry==1).

```c
typedef struct pat_Stmt_t {
    int64_t deref_589_saved_cur;
    int64_t deref_593_saved_cur;
    /* ... all locals ... */
} pat_Stmt_t;

static SnoVal pat_Stmt(pat_Stmt_t **zz, const char *_subj, int64_t _slen,
                       int64_t *_cur_ptr, int _entry) {
    if (_entry == 0) { *zz = calloc(1, sizeof(pat_Stmt_t)); }
    pat_Stmt_t *z = *zz;
    /* use z->deref_589_saved_cur instead of static deref_589_saved_cur */
    ...
}
```

Child pattern calls embed a pointer field in the parent struct:
```c
typedef struct pat_Stmt_t {
    ...
    struct pat_Label_t *Label_z;   /* child frame for *Label */
} pat_Stmt_t;
```

**Where to make the change:** `src/sno2c/emit_byrd.c`
- `byrd_emit_named_pattern()` — emit the struct typedef + function signature
- `emit_imm` / `emit_cond` — use `z->varname` instead of `static str_t var_xxx`
- `byrd_emit` E_DEREF case — call `pat_X(&z->X_z, subj, slen, cur, entry)`
- All `decl_add()` calls — collect into struct fields, not static locals

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

printf 'X = 1\n' | /tmp/beauty_tramp_bin   # expect: X = 1

/tmp/beauty_tramp_bin < $BEAUTY > /tmp/beauty_tramp_out.sno
$SNO -f -P256k -I$INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_tramp_out.sno
# Expect: empty → M-BEAUTY-FULL fires
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
| 2026-03-14 | E_COND/E_IMM E_STR fix `6d09bfa` | binary compiles, runs, fails on static re-entrancy |
| 2026-03-14 | beauty.sno rename snoXXX→XXX `d504d80` | beautifier bootstrap noted |
| 2026-03-14 | Next blocker: static locals in pat_Xxx | re-entrant calls stomp saved cursors |
