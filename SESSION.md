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
| **HEAD** | `560c56a` — feat(runtime): engine_stub.c — compiled path links without engine.c |

---

## M-COMPILED-BYRD — FIRED ✅ (`560c56a`, 2026-03-15)

What was proven:
- `sno2c` emits correct labeled-goto Byrd box C for all pattern types
- ARB scan wrap added: bare patterns wrapped in SEQ(ARB, pat) for substring scan semantics
- uid continuity fix: multiple Byrd blocks in one .sno file no longer collide on labels
- `engine_stub.c` added: compiled binaries link with stub instead of engine.c
- Integration test (hello/world/xyz substring scan): ALL OK
- Sprint oracles: 28/28 pass

---

## What Sprint 3 (`beauty-runtime`) Means

Run `beauty_full_bin < beauty.sno` to completion without crash, hang, or abort.
Now using compiled Byrd boxes (not the interpreter). Sprint 3 catches runtime
issues in `snobol4.c` / `snobol4_inc.c` that surface when beauty actually runs.

**Commit when:** Binary exits cleanly on beauty.sno input.

---

## One Next Action — Build beauty_full_bin and Run It

Steps:
1. Clone repos (see Container State below)
2. Build CSNOBOL4 from tarball (see Container State)
3. Build sno2c: `make -C src/sno2c`
4. Compile beauty.sno:
   ```bash
   CORPUS=/home/claude/SNOBOL4-corpus
   INC=$CORPUS/programs/inc
   BEAUTY=$CORPUS/programs/beauty/beauty.sno
   ./src/sno2c/sno2c $BEAUTY > /tmp/beauty_full.c
   gcc -O0 -g -I src/runtime/snobol4 -I src/runtime \
       /tmp/beauty_full.c src/runtime/snobol4/snobol4.c \
       src/runtime/snobol4/snobol4_inc.c \
       src/runtime/snobol4/snobol4_pattern.c \
       src/runtime/engine_stub.c \
       -lgc -lm -o /tmp/beauty_full_bin
   ```
5. Run it: `/tmp/beauty_full_bin < $BEAUTY > /tmp/beauty_out.sno`
6. If it crashes or hangs: debug. The compiled C is in /tmp/beauty_full.c — read it.
7. When it exits cleanly: commit `feat(runtime): beauty-runtime — binary exits clean on beauty.sno`
8. Then Sprint 4: diff against CSNOBOL4 oracle output.

**Known risk:** beauty.sno uses many SNOBOL4 features. Parser or codegen may reject
some constructs. sno2c itself may fail to parse beauty.sno — that's a parser issue,
not a runtime issue. Check `./src/sno2c/sno2c $BEAUTY` output first before linking.

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — retired from compiled path.
- Do NOT rewrite emit_byrd.c — it works (28/28 oracles pass).
- Do NOT rewrite the ARB scan wrap in emit.c — it works.
- Do NOT reset byrd_uid_ctr — the continuity fix is intentional.

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
    # Binary at: /home/claude/snobol4-2.3.3/snobol4 (use directly)
    cd ..

    cd SNOBOL4-tiny && make -C src/sno2c

---

## Rebuild and Test Commands

    cd /home/claude/SNOBOL4-tiny
    make -C src/sno2c

    # Sprint oracle tests (28/28 pass; 4 exit 1 intentionally):
    R=src/runtime
    for c in test/sprint*/*.c; do
        gcc -O0 -g "$c" $R/runtime.c -I$R -o /tmp/t 2>/dev/null
        /tmp/t; echo "exit=$? $c"
    done

    # Integration test (substring scan — expect ALL OK):
    cat > /tmp/pat_test.sno << 'EOF'
*  Pattern match integration test
        X = "hello world"
        X  "hello"                    :S(S1)F(S3)
S1      X  "world"                    :S(S2)F(S3)
S2      X  "xyz"                      :S(S3)F(S3OK)
S3      OUTPUT = "FAIL"               :(END)
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
| src/sno2c/emit_byrd.c | Compiled Byrd box emitter | Keeper |
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
| 2026-03-15 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c + ALL OK |
| 2026-03-15 | uid continuity fix (735c456) | duplicate labels across multiple patterns |
| 2026-03-15 | ARB scan wrap (735c456) | substring scan semantics |
| 2026-03-15 | emit.c wired — byrd_emit_pattern() called (1c2062a) | compiled path active |
| 2026-03-13 | emit_byrd.c written committed (cb3f97e) | C port of Python pipeline complete |
| 2026-03-14 | compiled-byrd-boxes sprint opened; smoke-tests retired | validated wrong runtime |
| 2026-03-14 | VarCache + INPUT redirect committed (be4fbb1) | Keeper work |
| 2026-03-13 | Architecture recorded: sno_pat_* stopgap, M-COMPILED-BYRD locked | Agreement with Lon |
