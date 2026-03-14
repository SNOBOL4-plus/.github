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
| **HEAD** | `06f4715 — fix(parse): binary ~ drops tag + emit_break wrap fix` |

---

## State at handoff (session 62)

Commits this session:
- `f74a384` — Greek port labels α/β/γ/ω watermark + UTF-8 decl_field_name fix ✅
- `e00f851` — Three-column pretty layout PLG/PL/PS/PG ✅
- `06f4715` — Binary ~ fix + emit_break wrap fix ✅

**Three sessions of progress:**

1. **emit_imm var_set fix** — `START` now passes clean (bare label lines work)
2. **Greek watermark** — all emitted C labels use α/β/γ/ω
3. **Three-column pretty layout** — generated C has label/stmt/goto columns
4. **emit_break wrap bug fixed** — `goto` inside braces, never detaches from guard
5. **Binary ~ fix** — tilde drops right-side tag (it's tree metadata, not a match target)

**Current state:** `START` passes clean. `X = 1` segfaults.

**Root cause of segfault (pinned):**
`epsilon ~ ''` in beauty.sno grammar means: match epsilon (zero chars), tag as empty-string node. After the tilde fix, the right side (`''`) is dropped, leaving the expression as just `epsilon`. That part is fine. BUT — many `~` expressions in the grammar are like `epsilon ~ '' epsilon ~ ''` chained. The parse tree for these may now have unexpected NULL nodes propagating through `byrd_emit`. The segfault is likely a NULL dereference in `byrd_emit` when `pat->left` or `pat->right` is NULL for a non-epsilon node kind.

**Simplest diagnostic:** Run under valgrind or add NULL guards:
```bash
printf 'X = 1\n' | valgrind /tmp/beauty_tramp_bin 2>&1 | head -20
```

---

## ONE NEXT ACTION — Diagnose segfault

```bash
cd /home/claude && apt-get install -y m4 libgc-dev valgrind
# rebuild repos (see Container Setup below)

printf 'X = 1\n' | valgrind --error-exitcode=1 /tmp/beauty_tramp_bin 2>&1 | head -30
```

If valgrind shows the crash site, fix it. Most likely one of:
1. `pat->left` is NULL in a node that expects a non-NULL child (E_CONCAT with NULL right
   after tilde drops right side) — byrd_emit(NULL, ...) is handled as epsilon, which is
   correct, but E_CONCAT left-recurse might not check
2. The segfault is in runtime code not pattern code — check if it's in `Reduce()` or
   `nInc()` being called with bad args

If `X = 1` passes after valgrind fix, run the full diff:
```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
SNO=/home/claude/snobol4-install/bin/snobol4

/tmp/beauty_tramp_bin < $BEAUTY > /tmp/beauty_compiled.sno
$SNO -f -P256k -I$INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_compiled.sno
# Empty diff → M-BEAUTY-FULL fires
```

---

## Artifact convention (mandatory every session touching sno2c/emit*.c)

```bash
# At END of session:
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
mkdir -p artifacts/trampoline_session63
./sno2c -trampoline -I$INC $BEAUTY > artifacts/trampoline_session63/beauty_tramp_session63.c
# Record md5, line count, gcc errors, active bug in artifacts/trampoline_session63/README.md
# Commit: artifact: beauty_tramp_session63.c — <one-line status>
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
| 2026-03-14 | beauty.sno snoXXX→XXX `d504d80` + beautifier bootstrap | oracle now self-referential |
| 2026-03-14 | S4_expression.sno→expression.sno `596cc5f` | same rename + jcooper paths fixed |
| 2026-03-14 | Technique 1 struct-passing `a3ea9ef` | re-entrancy fixed, 0 gcc errors, X=1 passes |
| 2026-03-14 | emit_imm var_set fix `dc8ad4b` | bare labels pass (START works) |
| 2026-03-14 | Greek watermark α/β/γ/ω `f74a384` | Lon's branding in all emitted labels |
| 2026-03-14 | Three-column pretty layout `e00f851` | Lon's watermark layout |
| 2026-03-14 | Binary ~ fix + wrap fix `06f4715` | START clean, X=1 segfaults |
