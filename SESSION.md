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
| **HEAD** | `1c2062a` — feat(emit): wire byrd_emit_pattern into emit_stmt — compiled Byrd box path active |

---

## The Architectural Decision (Lon + Claude, 2026-03-14)

**The `smoke-tests` sprint was validating the wrong runtime and has been retired.**

`sno_pat_*` / `engine.c` is a stopgap interpreter. Validating `sno2c` against it
proves nothing about the compiled Byrd box path that actually matters.

The correct test: does `sno2c` emit correct labeled-goto Byrd box C?
Validated against the hand-written sprint0-22 oracle files that already exist.

**The Python pipeline (lower.py + emit_c_byrd.py) produced correct output — 609/609
worm cases, full Chomsky hierarchy. That is the ground truth. emit_byrd.c is a
C port of that pipeline, wired into sno2c.**

---

## What Was Done This Session (2026-03-15)

emit_byrd.c wired into emit_stmt() in emit.c — committed 1c2062a.

What works:
- byrd_emit_pattern() called from the pattern-match statement case
- Subject extracted: _subj%d (const char*), _slen%d (int64_t), _cur%d cursor
- Static decls emitted before goto root_alpha; full Byrd box C inline in function
- gamma (_byrd_%d_ok) sets _ok%d=1; omega sets _ok%d=0
- Replacement (= repl) path: cursor-range memcpy with GC_malloc
- _ok%d declared before Byrd block (no C jump-over-declaration errors)
- Clean build, zero gcc errors
- Oracle C files: 28/28 pass (4 intentional-fail tests correctly exit 1)
- End-to-end .sno compile: emits correct C, links clean, Byrd box fires

Known gap discovered — NEXT ACTION:

Bare LIT("world") pattern is anchored at cursor=0 — but SNOBOL4 pattern
matching is a substring scan. X "world" on "hello world" must find
"world" anywhere in X, not just at position 0.

Fix: In emit_stmt(), before calling byrd_emit_pattern(), wrap s->pattern
in an implicit SEQ(ARB, s->pattern) — unless the pattern already anchors
with POS(0) as its leftmost node.

The oracle C files do not expose this because they hardcode cursor=0 and
always wrap with POS(0)/RPOS(0) in the test pattern itself. The gap only
shows up in real .sno compilation.

---

## One Next Action — Add ARB Scan Wrap

The fix is in src/sno2c/emit.c, in the if (s->pattern) block.

Add a pat_is_anchored() static helper before emit_stmt():

    static int pat_is_anchored(Expr *e) {
        if (!e) return 0;
        if (e->kind == E_CALL && e->sval && strcasecmp(e->sval, "POS") == 0) return 1;
        if (e->kind == E_CONCAT) return pat_is_anchored(e->left);
        return 0;
    }

Then in the if (s->pattern) block, before byrd_emit_pattern():

    Expr *scan_pat = s->pattern;
    if (!pat_is_anchored(s->pattern)) {
        Expr *arb = expr_new(E_CALL);
        arb->sval = strdup("ARB");
        arb->nargs = 0;
        Expr *seq = expr_new(E_CONCAT);
        seq->left = arb;
        seq->right = s->pattern;
        scan_pat = seq;
    }
    byrd_emit_pattern(scan_pat, out, root_lbl, sv, sl, cv, ok_lbl, fail_lbl);

Steps:
1. cd /home/claude/SNOBOL4-tiny && git log --oneline -3 — confirm HEAD 1c2062a
2. grep -n "E_CALL\|E_CONCAT\|expr_new\|Expr\b" src/sno2c/snoc.h — check Expr API
3. grep -n "ARB\b" src/sno2c/emit_byrd.c — confirm ARB is E_CALL with sval="ARB"
4. Add pat_is_anchored() and scan_pat wrap
5. make -C src/sno2c — confirm clean build
6. Run integration test (see Rebuild section below) — expect "ALL OK"
7. Run sprint oracles — still 28/28
8. Commit: "feat(emit): ARB scan wrap — SNOBOL4 substring scan semantics"

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — retired from compiled path.
- Do NOT chase sno_match_pattern / materialise bugs — irrelevant to Byrd boxes.
- Do NOT run test_snoCommand_match.sh — validates the wrong runtime.
- Do NOT rewrite emit_byrd.c — it works.
- Do NOT rewrite the wiring in emit.c — it works. One gap: ARB scan wrap.

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
    # Binary at: /home/claude/snobol4-2.3.3/snobol4 (use directly, install unreliable)
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

    # Integration test (after ARB wrap fix, expect "ALL OK"):
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
        src/runtime/engine.c -lgc -lm -o /tmp/pat_test
    /tmp/pat_test   # should print ALL OK

---

## What Is Keeper Work

| File | What it is | Status |
|------|-----------|--------|
| src/sno2c/emit_byrd.c | Compiled Byrd box emitter | Keeper |
| src/sno2c/emit.c | Wired — byrd_emit_pattern called from emit_stmt | Keeper |
| src/sno2c/snoc.h | IR + public API | Keeper |
| src/runtime/snobol4/snobol4.c | Value runtime, builtins, var table, I/O | Keeper |
| src/runtime/snobol4/snobol4_inc.c | Gen, Qize, Shift/Reduce, stack, counter | Keeper |
| src/runtime/engine.c | Byrd box interpreter | EVAL only — do not modify |
| src/runtime/snobol4/snobol4_pattern.c | SnoPattern tree + materialise | EVAL only |
| src/codegen/emit_c_byrd.py | Python emitter — ground truth | Do not delete |
| src/ir/lower.py | Python lowering pass — ground truth | Do not delete |
| src/ir/byrd_ir.py | Python IR — ground truth | Do not delete |

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-15 | ARB scan wrap gap identified | bare LIT anchored at 0 not substring scan |
| 2026-03-15 | emit.c wired — byrd_emit_pattern() called (1c2062a) | compiled path active |
| 2026-03-13 | emit_byrd.c written committed (cb3f97e) | C port of Python pipeline complete |
| 2026-03-14 | compiled-byrd-boxes sprint opened; smoke-tests retired | validated wrong runtime |
| 2026-03-14 | VarCache + INPUT redirect committed (be4fbb1) | Keeper work |
| 2026-03-13 | Architecture recorded: sno_pat_* stopgap, M-COMPILED-BYRD locked | Agreement with Lon |
