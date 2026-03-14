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
| **HEAD** | `91d097c` — fix(emit): beauty-runtime — gcc clean, binary exits 0 on beauty.sno |

---

## Sprint 3 Status — binary exits 0, but "Internal Error"

`beauty_full_bin < beauty.sno` exits 0. No crash, no hang, no abort. ✅
But output is only 10 lines then "Internal Error" from beauty's own error path.

CSNOBOL4 oracle produces 790 lines. diff is not empty. Sprint 4 not yet fireable.

Root cause identified: **`E_DEREF` (`*varname` indirect pattern) is stubbed as epsilon in `emit_byrd.c`.**

beauty.sno uses indirect patterns ~100+ times:
- `*snoParse`, `*snoWhite` (42x), `*assign` (21x), `*snoExpr14` (17x), etc.
- These are central to the recursive grammar — beauty can't parse without them.

---

## One Next Action — Implement E_DEREF in emit_byrd.c

The fix lives entirely in `emit_byrd.c` around line 1129 (`case E_DEREF:`).

### What E_DEREF must do

`*varname` in pattern position means: at runtime, get the value of `varname`,
treat it as a pattern, and match it against the subject starting at the current cursor.

### Runtime API available (snobol4.h line ~367)

```c
int sno_match_pattern(SnoVal pat, const char *subject);
```

But this takes a full subject string — doesn't take cursor offset. Need to understand
what it returns (match length? bool?) before wiring.

### Steps

1. Read `src/runtime/snobol4/snobol4_pattern.c` lines 791–840 — understand
   `sno_match_pattern` return value and how cursor advance works.
2. Check if there's a cursor-aware variant or if we need to pass `subject + cursor`.
3. Implement `E_DEREF` in `emit_byrd.c`:
   - alpha: get var value via `sno_var_get(varname)`
   - call match API against subject at current cursor
   - on success: advance cursor by match length, goto gamma
   - on failure: goto omega
4. Rebuild sno2c, regenerate beauty_full.c, relink, run.
5. Watch "Internal Error" disappear and output grow toward 790 lines.

### Key file locations (in container, clone fresh each session)

```
src/sno2c/emit_byrd.c       — E_DEREF case ~line 1129
src/runtime/snobol4/snobol4_pattern.c  — sno_match_pattern ~line 791
src/runtime/snobol4/snobol4.h          — API declarations
```

### Known: `sno_match_pattern` signature

```c
int sno_match_pattern(SnoVal pat, const char *subject);
```

Located in `snobol4_pattern.c` line 791. Return value and cursor semantics
must be read before use — do NOT guess.

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — retired from compiled path.
- Do NOT rewrite emit_byrd.c wholesale — only add E_DEREF implementation.
- Do NOT reset byrd_uid_ctr — continuity fix is intentional.
- Do NOT remove fn_seen[] / byrd_fn_scope_reset() — fixes real gcc errors.

---

## Container State (as of this handoff)

These will NOT be present in next Claude's container. Clone fresh:

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

## Rebuild and Test Commands

    cd /home/claude/SNOBOL4-tiny
    make -C src/sno2c

    # Full beauty pipeline:
    CORPUS=/home/claude/SNOBOL4-corpus
    INC=$CORPUS/programs/inc
    BEAUTY=$CORPUS/programs/beauty/beauty.sno
    ./src/sno2c/sno2c -I $INC $BEAUTY > /tmp/beauty_full.c
    gcc -O0 -g -I src/runtime/snobol4 -I src/runtime \
        /tmp/beauty_full.c src/runtime/snobol4/snobol4.c \
        src/runtime/snobol4/snobol4_inc.c \
        src/runtime/snobol4/snobol4_pattern.c \
        src/runtime/engine_stub.c \
        -lgc -lm -o /tmp/beauty_full_bin
    /tmp/beauty_full_bin < $BEAUTY > /tmp/beauty_out.sno
    echo "exit=$?  lines=$(wc -l < /tmp/beauty_out.sno)"

    # Oracle (CSNOBOL4):
    /home/claude/snobol4-2.3.3/snobol4 -f -P256k -I $INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno

    # Sprint 4 trigger:
    diff /tmp/beauty_oracle.sno /tmp/beauty_out.sno

    # Sprint oracle tests (must stay 28/28):
    R=src/runtime
    for c in test/sprint*/*.c; do
        gcc -O0 -g "$c" $R/runtime.c -I$R -o /tmp/t 2>/dev/null
        /tmp/t; echo "exit=$? $c"
    done

    # Integration test (substring scan — expect ALL OK):
    cat > /tmp/pat_test.sno << 'EOF'
        X = "hello world"
        X  "hello"   :S(S1)F(S3)
S1      X  "world"   :S(S2)F(S3)
S2      X  "xyz"     :S(S3)F(S3OK)
S3      OUTPUT = "FAIL"   :(END)
S3OK    OUTPUT = "ALL OK"
END
EOF
    ./src/sno2c/sno2c /tmp/pat_test.sno > /tmp/pat_test.c
    gcc -O0 -g -I src/runtime/snobol4 -I src/runtime \
        /tmp/pat_test.c src/runtime/snobol4/snobol4.c \
        src/runtime/snobol4/snobol4_inc.c \
        src/runtime/snobol4/snobol4_pattern.c \
        src/runtime/engine_stub.c -lgc -lm -o /tmp/pat_test
    /tmp/pat_test   # should print ALL OK

---

## What Is Keeper Work

| File | What it is | Status |
|------|-----------|--------|
| src/sno2c/emit_byrd.c | Compiled Byrd box emitter | Keeper — E_DEREF needs impl |
| src/sno2c/emit.c | Wired — ARB scan wrap + byrd_emit_pattern | Keeper |
| src/sno2c/snoc.h | IR + public API | Keeper |
| src/runtime/snobol4/snobol4.c | Value runtime, builtins, var table, I/O | Keeper |
| src/runtime/snobol4/snobol4_inc.c | Gen, Qize, Shift/Reduce, stack, counter | Keeper |
| src/runtime/engine_stub.c | Linker stub — compiled path only | Keeper |
| src/runtime/engine.c | Byrd box interpreter | EVAL only — do not modify |
| src/runtime/snobol4/snobol4_pattern.c | SnoPattern tree + materialise | EVAL only |
| src/codegen/emit_c_byrd.py | Python emitter — ground truth | Do not delete |
| src/ir/lower.py | Python lowering pass — ground truth | Do not delete |
| src/ir/byrd_ir.py | Python IR — ground truth | Do not delete |

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | E_DEREF identified as root cause of Internal Error | beauty uses *var ~100x |
| 2026-03-14 | sno_output sig fixed (str_t→ptr+len), decl dedup across fn scope | gcc clean |
| 2026-03-14 | binary links and exits 0 (91d097c) | 3 gcc bugs fixed |
| 2026-03-15 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c + ALL OK |
| 2026-03-15 | uid continuity fix (735c456) | duplicate labels across multiple patterns |
| 2026-03-15 | ARB scan wrap (735c456) | substring scan semantics |
| 2026-03-15 | emit.c wired — byrd_emit_pattern() called (1c2062a) | compiled path active |
| 2026-03-13 | emit_byrd.c written committed (cb3f97e) | C port of Python pipeline complete |
| 2026-03-14 | compiled-byrd-boxes sprint opened; smoke-tests retired | validated wrong runtime |
| 2026-03-14 | VarCache + INPUT redirect committed (be4fbb1) | Keeper work |
| 2026-03-13 | Architecture recorded: sno_pat_* stopgap, M-COMPILED-BYRD locked | Agreement with Lon |
