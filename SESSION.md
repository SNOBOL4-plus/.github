# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `compiled-byrd-boxes` (sprint 2/4 toward M-BEAUTY-FULL) |
| **Milestone** | M-COMPILED-BYRD |
| **HEAD** | `cb3f97e` — feat(emit_byrd): compiled Byrd box emitter — LIT/SEQ/ALT/ARBNO/POS/RPOS/LEN/ANY/NOTANY/SPAN/BREAK/ARB/REM/FENCE/IMM/COND |

---

## The Architectural Decision (Lon + Claude, 2026-03-14)

**The `smoke-tests` sprint was validating the wrong runtime and has been retired.**

`sno_pat_*` / `engine.c` is a stopgap interpreter. Validating `sno2c` against it
proves nothing about the compiled Byrd box path that actually matters.

The correct test: does `sno2c` emit correct labeled-goto Byrd box C?
Validated against the hand-written sprint0–22 oracle files that already exist.

**The Python pipeline (lower.py + emit_c_byrd.py) produced correct output — 609/609
worm cases, full Chomsky hierarchy. That is the ground truth. `emit_byrd.c` is a
C port of that pipeline, wired into sno2c.**

---

## What Was Done This Session (2026-03-13)

`src/sno2c/emit_byrd.c` written from scratch and committed (`cb3f97e`).

Full C port of `src/ir/lower.py` + `src/codegen/emit_c_byrd.py`.

**Implements:** LIT, SEQ/CAT, ALT, ARBNO, POS, RPOS, LEN, TAB, RTAB, ANY, NOTANY,
SPAN, BREAK, ARB, REM, FENCE (0-arg + 1-arg), SUCCEED, FAIL, ABORT,
E_IMM ($), E_COND (.)

Two-pass generation via `open_memstream`: static declarations emitted before
`goto root_alpha` (C99 compliant). Smoke test: POS(0) ARBNO("Bird"|"Blue") RPOS(0)
on "BlueBird" generates compilable C, exits 0.

Sprint0-5 oracles: 15/15 pass. sno2c builds clean, zero errors.

---

## One Next Action — Wire emit_byrd into emit.c

`emit_byrd.c` is written and working. It is NOT yet called by `sno2c`.

**The integration step:** replace the pattern-match case in `emit_stmt()` in
`src/sno2c/emit.c` (search for `/* ---- pattern match`).

### Current (stopgap — interpreter):
```c
E("SnoVal   _s%d = ", u); emit_expr(s->subject); E(";\n");
E("SnoVal   _p%d = ", u); emit_pat(s->pattern); E(";\n");
E("SnoMatch _m%d = sno_match(&_s%d, _p%d);\n", u,u,u);
E("int      _ok%d = !_m%d.failed;\n", u, u);
```

### Target (compiled Byrd boxes):
```c
/* subject already in _s%d — get raw string for Byrd box */
int u = uid();
E("/* byrd match u%d */\n", u);
E("SnoVal _s%d = ", u); emit_expr(s->subject); E(";\n");
E("const char *_subj%d = sno_to_cstr(_s%d);\n", u, u);
E("int64_t _slen%d = (int64_t)sno_strlen(_s%d);\n", u, u);
E("int64_t _cur%d = 0;\n", u);

char root_lbl[64], ok_lbl[64], fail_lbl[64];
snprintf(root_lbl, sizeof root_lbl, "byrd_%d", u);
snprintf(ok_lbl,   sizeof ok_lbl,   "_byrd_%d_ok", u);
snprintf(fail_lbl, sizeof fail_lbl, "_byrd_%d_fail", u);

char sv[32], sl[32], cv[32];
snprintf(sv, sizeof sv, "_subj%d", u);
snprintf(sl, sizeof sl, "_slen%d", u);
snprintf(cv, sizeof cv, "_cur%d",  u);

byrd_emit_pattern(s->pattern, out, root_lbl, sv, sl, cv, ok_lbl, fail_lbl);

E("int _ok%d = 1; goto _byrd_%d_done;\n", u, u);
E("%s: _ok%d = 0;\n", fail_lbl, u);
E("_byrd_%d_done:;\n", u);
/* _ok%d now holds 1=match, 0=fail — same as old _ok%d = !_m%d.failed */
```

NOTE: `out` in emit.c is the static `FILE *out` variable. Pass it directly to
`byrd_emit_pattern`. `byrd_emit_pattern` is already declared in `snoc.h`.

**Runtime accessors needed** — check `src/runtime/snobol4/snobol4.h` for:
- `sno_to_cstr(SnoVal v)` — returns `const char*`
- `sno_strlen(SnoVal v)` — returns length

If those don't exist by those names, grep for what does. The SnoVal struct has
`.type` / `.u.str.ptr` / `.u.str.len` fields — use those directly if needed.

### Steps:
1. `cd /home/claude/SNOBOL4-tiny && git log --oneline -3` — confirm HEAD cb3f97e
2. `grep -n "pattern match" src/sno2c/emit.c` — find the block
3. Read `src/runtime/snobol4/snobol4.h` for string accessor names
4. Edit emit.c — replace the sno_match block with byrd_emit_pattern call
5. `make -C src/sno2c` — confirm clean build
6. Sprint oracle check: all sprint0-5 should still pass
7. Test on a simple .sno: `echo 'OUTPUT = "hello"' | src/sno2c/sno2c /dev/stdin`
8. Iterate until sprint0-22 pass

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in `sno_pat_*` / `engine.c` — retired from compiled path.
- Do NOT chase `sno_match_pattern` / `materialise` bugs — irrelevant to Byrd boxes.
- Do NOT run `test_snoCommand_match.sh` — validates the wrong runtime.
- Do NOT rewrite `emit_byrd.c` — it works. Wire it in.

---

## Container State (as of this handoff)

These will NOT be present in next Claude's container. Clone fresh:
```bash
apt-get install -y m4 libgc-dev

git config --global user.name "LCherryholmes"
git config --global user.email "lcherryh@yahoo.com"

TOKEN=TOKEN_SEE_LON

git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/.github.git snobol4-plus-github

# Build CSNOBOL4 — tarball is in uploads as snobol4-2_3_3_tar.gz
cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
./configure --prefix=/home/claude/snobol4-install && make -j$(nproc) && make install
cd ..

# Build sno2c
cd SNOBOL4-tiny && make -C src/sno2c
```

---

## Rebuild Commands

```bash
cd /home/claude/SNOBOL4-tiny

make -C src/sno2c

# Sprint oracle tests (sprint0-5 should all pass):
R=src/runtime
gcc -O0 -g test/sprint1/lit_hello.c $R/runtime.c -Isrc/runtime -o /tmp/t && /tmp/t && echo PASS
gcc -O0 -g test/sprint2/cat_pos_lit_rpos.c $R/runtime.c -Isrc/runtime -o /tmp/t && /tmp/t && echo PASS
gcc -O0 -g test/sprint3/alt_first.c $R/runtime.c -Isrc/runtime -o /tmp/t && /tmp/t && echo PASS
gcc -O0 -g test/sprint5/arbno_match.c $R/runtime.c -Isrc/runtime -o /tmp/t && /tmp/t && echo PASS
```

---

## What Is Keeper Work

| File | What it is | Status |
|------|-----------|--------|
| `src/sno2c/emit_byrd.c` | Compiled Byrd box emitter — written this session | ✅ Keeper |
| `src/sno2c/emit.c` | C compiler emitter — needs wiring to emit_byrd | ✅ Keeper |
| `src/sno2c/snoc.h` | IR + public API (byrd_emit_pattern declared) | ✅ Keeper |
| `src/runtime/snobol4/snobol4.c` | Value runtime, builtins, var table, I/O | ✅ Keeper |
| `src/runtime/snobol4/snobol4_inc.c` | Gen, Qize, Shift/Reduce, stack, counter | ✅ Keeper |
| `src/runtime/engine.c` | Byrd box interpreter | ⚠️ EVAL only — do not modify |
| `src/runtime/snobol4/snobol4_pattern.c` | SnoPattern tree + materialise | ⚠️ EVAL only |
| `src/codegen/emit_c_byrd.py` | Python emitter — ground truth | ✅ Do not delete |
| `src/ir/lower.py` | Python lowering pass — ground truth | ✅ Do not delete |
| `src/ir/byrd_ir.py` | Python IR — ground truth | ✅ Do not delete |

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | `emit_byrd.c` written, smoke-tested, committed (`cb3f97e`) | C port of Python pipeline complete |
| 2026-03-14 | `compiled-byrd-boxes` sprint opened; `smoke-tests` retired | smoke-tests validated wrong runtime |
| 2026-03-14 | VarCache + INPUT redirect committed (`be4fbb1`) | Keeper work |
| 2026-03-13 | Architecture recorded: sno_pat_* stopgap, M-COMPILED-BYRD locked | Agreement with Lon |
| 2026-03-13 | materialise-once fix | SPAT_USER_CALL called N times per match |
