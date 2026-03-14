# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-runtime` (sprint 3/4 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `2379052` — fix(beauty-runtime): parse/emit/pattern fixes toward M-BEAUTY-FULL |

---

## Sprint 3 Status — T_FUNC deferred, snoParse still returns -1

All fixes committed at `2379052`. Binary compiles clean with real `engine.c`.
`snoParse` is correctly reached, correct variable name resolved, T_FUNC nodes
built — but `sno_match_pattern_at` returns -1 on `"START\n"`.

### What was fixed this session (all in `2379052`):

1. **parse.c `parse_lbin`** — `T_STAR` after WS is binary only if ALSO
   followed by WS. `POS(0) *snoParse` was parsing as `E_MUL(POS(0), snoParse)`
   because binary `*` in `parse_lbin` consumed `POS(0) *` without checking
   what follows `*`. Fix: peek one token ahead — if not WS, it's unary `*foo`.

2. **emit_byrd.c E_DEREF** — varname was `pat->sval` (always NULL for E_DEREF
   built by `unop()`). Fixed to `pat->left->sval`. Now generates
   `sno_var_get("snoParse")` not `sno_var_get("(null)")`.

3. **emit_byrd.c E_CALL** — added `nPush`/`nInc`/`nDec`/`nPop`/`nTop` cases
   calling `sno_npush()`/`sno_ninc()`/`sno_ntop()` etc. at match time.

4. **emit_byrd.c E_REDUCE** — new case calling `sno_apply("Reduce", args, 2)`
   at match time. `emit_simple_val()` helper added for arg emission.

5. **snobol4_pattern.c SPAT_USER_CALL** — now returns `T_FUNC` deferred node
   instead of firing `sno_apply()` at materialise time. Side-effect calls
   (`nPush`, `nInc`, `Reduce`) must fire when the engine REACHES the node
   during matching — not during `materialise()` which runs before the engine.
   `make_func()` helper and `deferred_call_fn()` added.

---

## One Next Action — verify T_FUNC wiring into engine

The remaining failure: `snoParse` pattern match returns -1 on `"START\n"`.
SNO_PAT_DEBUG confirms SPAT_USER_CALL is now deferred (prints "→T_FUNC").
But the engine may not be calling `deferred_call_fn`. Diagnose:

```bash
cd /home/claude/SNOBOL4-tiny
# Setup
apt-get install -y m4 libgc-dev
git config --global user.name "LCherryholmes"
git config --global user.email "lcherryh@yahoo.com"
TOKEN=TOKEN_SEE_LON
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git

cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
./configure --prefix=/home/claude/snobol4-install && make -j$(nproc)
cd ..

cd SNOBOL4-tiny && make -C src/sno2c

CORPUS=/home/claude/SNOBOL4-corpus
INC=$CORPUS/programs/inc
BEAUTY=$CORPUS/programs/beauty/beauty.sno
R=src/runtime

./src/sno2c/sno2c -I $INC $BEAUTY > /tmp/beauty_full.c
gcc -O0 -g -I $R/snobol4 -I $R \
    /tmp/beauty_full.c $R/snobol4/snobol4.c \
    $R/snobol4/snobol4_inc.c $R/snobol4/snobol4_pattern.c \
    $R/engine.c -lgc -lm -o /tmp/beauty_full_bin

timeout 10 /tmp/beauty_full_bin < $BEAUTY > /tmp/beauty_out.sno 2>&1
echo "exit=$?  lines=$(wc -l < /tmp/beauty_out.sno)"
```

**Step 1 — verify T_FUNC fires:** Add to `deferred_call_fn` in
`snobol4_pattern.c` at the top:
```c
fprintf(stderr, "DEFERRED_CALL_FN: %s\n", d->name);
```
If this never prints, the engine isn't calling `func()` on the T_FUNC nodes.

**Step 2 — check Pattern struct compatibility:** The `Pattern` struct in
`engine.h` has `func` and `func_data` fields at lines ~80-81. The
`pattern_alloc()` in `snobol4_pattern.c` must zero-init the full struct so
`func` and `func_data` are accessible. Verify:
```bash
grep -n "func\|func_data" src/runtime/engine.h
grep -n "pattern_alloc\|calloc\|memset" src/runtime/snobol4/snobol4_pattern.c | head -10
```
If `pattern_alloc` doesn't zero the `func`/`func_data` fields, add
`memset(p, 0, sizeof(*p))` or use `calloc`.

**Step 3 — if T_FUNC fires but Reduce fails:** Add trace to `deferred_call_fn`
for the reduce branch to print `t_arg` and `n_arg` values after processing.

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — they're now used correctly.
- Do NOT rewrite emit_byrd.c wholesale — only targeted fixes.
- Do NOT reset byrd_uid_ctr — continuity fix is intentional.
- Do NOT write the TOKEN into any file.
- Link `engine.c` (NOT `engine_stub.c`) — *snoParse needs real engine.

---

## Container State (clone fresh each session)

    apt-get install -y m4 libgc-dev
    git config --global user.name "LCherryholmes"
    git config --global user.email "lcherryh@yahoo.com"
    TOKEN=TOKEN_SEE_LON
    git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
    git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
    git clone https://$TOKEN@github.com/SNOBOL4-plus/.github.git dotgithub

    # Build CSNOBOL4 — tarball in uploads as snobol4-2_3_3_tar.gz
    cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
    tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
    ./configure --prefix=/home/claude/snobol4-install && make -j$(nproc)
    cd ..

    cd SNOBOL4-tiny && make -C src/sno2c

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | sno_output_str fix — linker error resolved (e78d177) | sno_output(str_t) does not exist |
| 2026-03-14 | M-PYTHON-UNIFIED retired → M-BYRD-SPEC (HQ 0959202) | Python was scaffold, not destination |
| 2026-03-14 | JCON lessons recorded in MISC.md (HQ a120be6) | bounded flag, temp liveness, jvm guidance |
| 2026-03-15 | fn_seen[] + byrd_fn_scope_reset() — cross-pattern decl dedup | static redecl errors in Gen/Qize |
| 2026-03-15 | E_DEREF implemented with sno_match_pattern_at() | beauty uses *varname ~100x |
| 2026-03-14 | binary links and exits 0 (91d097c) | 3 gcc bugs fixed |
| 2026-03-15 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c + ALL OK |
| 2026-03-15 | uid continuity fix (735c456) | duplicate labels across multiple patterns |
| 2026-03-15 | ARB scan wrap (735c456) | substring scan semantics |
| 2026-03-15 | emit.c wired — byrd_emit_pattern() called (1c2062a) | compiled path active |
| 2026-03-13 | emit_byrd.c written committed (cb3f97e) | C port of Python pipeline complete |
| 2026-03-16 | parse_lbin T_STAR fix + E_DEREF varname + E_REDUCE + SPAT_USER_CALL→T_FUNC (2379052) | *snoParse was E_MUL; USER_CALL fired at materialise not match time |

---

## Sprint 3 Status — linker error fixed, gcc compile + run is next

The `sno_output` linker error is resolved. `sno2c` builds clean. `beauty_full.c`
was generated successfully this session. Next Claude picks up at the gcc compile step.

What was done this session (all committed):

1. **`sno_output` fix** (`e78d177`) — COND handler OUTPUT case in `emit_byrd.c` ~line 826.
   Replaced `sno_output(str_t)` (nonexistent) with malloc+memcpy → `sno_output_str(const char*)`.
   sno2c links clean. beauty_full.c generates without error.

2. **HQ: M-PYTHON-UNIFIED retired** (`0959202`) — replaced with M-BYRD-SPEC.
   Python was the scaffold. `emit_byrd.c` is the real implementation. No Python in pipeline.
   M-BYRD-SPEC = language-agnostic written spec of α/β/γ/ω wiring rules, all backends
   implement independently against it.

3. **HQ: JCON lessons recorded** (`a120be6`) — six actionable items in MISC.md:
   - `bounded` flag (highest value post-M-BEAUTY-FULL optimization)
   - Temp liveness lattice (worth revisiting at M-COMPILED-SELF)
   - Materialized IR vs. streaming (required for future optimizer)
   - Deep similarity confirmed
   - Cursor model — ours is better for SNOBOL4
   - SNOBOL4-jvm guidance (what to copy / avoid from JCON)

---

## One Next Action — gcc compile beauty_full.c and run

```bash
cd /home/claude/SNOBOL4-tiny
CORPUS=/home/claude/SNOBOL4-corpus
INC=$CORPUS/programs/inc
BEAUTY=$CORPUS/programs/beauty/beauty.sno

# Regenerate (sno2c is clean at e78d177)
./src/sno2c/sno2c -I $INC $BEAUTY > /tmp/beauty_full.c

# Compile
R=src/runtime
gcc -O0 -g -I $R/snobol4 -I $R \
    /tmp/beauty_full.c $R/snobol4/snobol4.c \
    $R/snobol4/snobol4_inc.c $R/snobol4/snobol4_pattern.c \
    $R/engine_stub.c -lgc -lm -o /tmp/beauty_full_bin

# Run
/tmp/beauty_full_bin < $BEAUTY > /tmp/beauty_out.sno
echo "exit=$?  lines=$(wc -l < /tmp/beauty_out.sno)"
# Target: exit 0, ~790 lines
```

### Oracle comparison (sprint 4 trigger)

```bash
/home/claude/snobol4-2.3.3/snobol4 -f -P256k -I $INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_out.sno
# Sprint 4 fires when diff is empty → M-BEAUTY-FULL fires
```

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — retired from compiled path.
- Do NOT rewrite emit_byrd.c wholesale — only targeted fixes.
- Do NOT reset byrd_uid_ctr — continuity fix is intentional.
- Do NOT remove fn_seen[] / byrd_fn_scope_reset() — fixes real gcc errors.
- Do NOT write the TOKEN into any file.

---

## Container State (clone fresh each session)

    apt-get install -y m4 libgc-dev
    git config --global user.name "LCherryholmes"
    git config --global user.email "lcherryh@yahoo.com"
    TOKEN=TOKEN_SEE_LON
    git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
    git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
    git clone https://$TOKEN@github.com/SNOBOL4-plus/.github.git dotgithub

    # Build CSNOBOL4 — tarball in uploads as snobol4-2_3_3_tar.gz
    cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
    tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
    ./configure --prefix=/home/claude/snobol4-install && make -j$(nproc)
    cd ..

    cd SNOBOL4-tiny && make -C src/sno2c

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | sno_output_str fix — linker error resolved (e78d177) | sno_output(str_t) does not exist |
| 2026-03-14 | M-PYTHON-UNIFIED retired → M-BYRD-SPEC (HQ 0959202) | Python was scaffold, not destination |
| 2026-03-14 | JCON lessons recorded in MISC.md (HQ a120be6) | bounded flag, temp liveness, jvm guidance |
| 2026-03-15 | fn_seen[] + byrd_fn_scope_reset() — cross-pattern decl dedup | static redecl errors in Gen/Qize |
| 2026-03-15 | E_DEREF implemented with sno_match_pattern_at() | beauty uses *varname ~100x |
| 2026-03-14 | binary links and exits 0 (91d097c) | 3 gcc bugs fixed |
| 2026-03-15 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c + ALL OK |
| 2026-03-15 | uid continuity fix (735c456) | duplicate labels across multiple patterns |
| 2026-03-15 | ARB scan wrap (735c456) | substring scan semantics |
| 2026-03-15 | emit.c wired — byrd_emit_pattern() called (1c2062a) | compiled path active |
| 2026-03-13 | emit_byrd.c written committed (cb3f97e) | C port of Python pipeline complete |
