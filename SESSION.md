# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `stmt-fn` (sprint 2/9 toward M-BEAUTY-FULL) |
| **Milestone** | M-STMT-FN |
| **HEAD** | `fb4915e — feat: M-TRAMPOLINE — block_fn_t + trampoline loop proven (session 56)` |

---

## M-TRAMPOLINE FIRED — Session 56 (2026-03-14)

Three hand-written proof-of-concept files committed at `fb4915e`:

- `src/sno2c/trampoline.h` — `block_fn_t = void*(*)(void)`, `trampoline_run()`, macros
- `src/sno2c/trampoline_hello.c` — two stmts, one block ✅
- `src/sno2c/trampoline_branches.c` — S/F routing, loop-back, labeled blocks ✅
- `src/sno2c/trampoline_pattern.c` — snobol4.c runtime integrated, literal pattern S/F ✅

Architecture validated:
- Every stmt = C fn returning `block_fn_t`
- Labeled stmt → new block
- S/F goto → return different block addresses
- Trampoline IS the engine: `while (pc) pc = (block_fn_t)pc()`

Also this session: CSNOBOL4 built and installed at `/home/claude/snobol4-install/bin/snobol4`
(from the snobol4-2_3_3_tar.gz upload). SNOBOL4 syntax/semantics verified hands-on.

---

## Architecture (from PLAN.md — read it fully before coding)

**Block-fn + trampoline model** (decided Session 55 with Lon):

Every SNOBOL4 statement → `stmt_N()` C function returning `block_fn_t`.
Statements grouped into `block_L()` by label reachability.
Trampoline: `while (pc) pc = pc()` — no interpreter, no engine.c, ever.
GOTO/`*X`/EVAL/CODE all = `block_fn_t` calls.
TCC for runtime CODE()/EVAL() (after M-BEAUTY-FULL).

## ONE NEXT ACTION — Sprint `stmt-fn`

Wire `emit.c` to emit `stmt_N()` functions instead of inline code.
Each stmt becomes its own C function returning `block_fn_t`.

### What to build

In `src/sno2c/emit.c`:

1. Change output preamble to `#include "trampoline.h"` (alongside runtime headers)

2. Add a new `emit_stmt_fn()` that wraps each statement as:
```c
static void *stmt_N(void) {
    /* subject eval, pattern match, replacement */
    /* α ... γ: return block_SLABEL; */
    /* ω: return block_FLABEL; */
}
```

3. Group stmts into `block_L()` functions by label reachability:
```c
static void *block_L42(void) {
    void *next;
    next = stmt_42(); if (next != (void*)block_L43) return next;
    next = stmt_43(); if (next != (void*)block_L44) return next;
    return block_L44;
}
```

4. Emit `main()` as:
```c
int main(void) {
    runtime_init();
    trampoline_run(block_START);
    return 0;
}
```

### Test sequence

```bash
cd /home/claude/SNOBOL4-tiny/src/sno2c && make

# Step 1: hello world
echo "    OUTPUT = 'hello'" > /tmp/t.sno && echo "END" >> /tmp/t.sno
./sno2c /tmp/t.sno > /tmp/t.c
gcc -O0 -I. -I../runtime/snobol4 /tmp/t.c ../runtime/snobol4/snobol4.c \
    ../runtime/snobol4/snobol4_inc.c ../runtime/snobol4/snobol4_pattern.c \
    ../runtime/engine.c -lgc -lm -w -o /tmp/t_bin && /tmp/t_bin
# expect: hello

# Step 2: S/F branch
# Step 3: beauty.sno → C → compile (0 errors) → run (may still fail semantically)
```

### Commit when
hello world compiles and runs through the new stmt-fn emitter. M-STMT-FN fires.

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
- **NEVER link engine.c in beauty_full_bin** — engine_stub.c only (for compiled path)
  - NOTE: trampoline POC tests use engine.c — that's fine (they're tests, not beauty_full_bin)
- **NEVER touch old emit_byrd.c struct-passing code** — superseded
- Read PLAN.md fully before coding

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | **PIVOT: block-fn + trampoline model** | complete architecture rethink with Lon |
| 2026-03-14 | M-TRAMPOLINE fired `fb4915e` | trampoline.h + 3 POC files proven |
| 2026-03-14 | CSNOBOL4 built from snobol4-2_3_3_tar.gz | reference interpreter available |
