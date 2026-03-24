# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.**

→ Rules: [RULES.md](RULES.md) · Beauty plan: [BEAUTY.md](BEAUTY.md) · History: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — B-276 (BEAUTY) · F-217 (Prolog) concurrent
**HEAD:** `f721492` B-276 beauty / `128dd2c` F-216 prolog (main)
**B-session:** M-BEAUTY-OMEGA ❌ — driver+ref ready (10/10 CSN+SPL); SPITBOL+SO crash: strip UTF-8 from driver comments
**F-session:** M-PROLOG-R10 ❌ — rung09 PASS; rung10 puzzle_02 has spurious backtrack output
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — F-217 (M-PROLOG-R10):**

```bash
cd /home/claude/snobol4x && make -C src

# Compile and run each rung10 puzzle:
# ./sno2c -pl <puzzle.pro> -o /tmp/t.c
# gcc -g -I src/frontend/prolog -o /tmp/t /tmp/t.c \
#     src/frontend/prolog/prolog_unify.c src/frontend/prolog/prolog_atom.c \
#     src/frontend/prolog/prolog_builtin.c -lm && /tmp/t

# puzzle_01: PASS (Cashier=smith Manager=brown Teller=jones)
# puzzle_02: FAIL — doesEarnMore spurious output; only WINNER line should print
# puzzle_06: PASS (Clark=druggist Jones=grocer Morgan=butcher Smith=policeman)
# Fix puzzle_02 constraint/backtrack bug, fire M-PROLOG-R10, advance to M-PROLOG-CORPUS
```

---

## Last Two Sessions (3 lines each)

**F-216 (2026-03-23) — compound emit \\n fix; rung09 PASS:**
Bug in `prolog_emit.c` line 185: `\\n` (literal backslash-n) emitted inside GNU statement expression for compound term construction; replaced with `\n`. Rung09 builtins now compile and match expected output exactly. HEAD `128dd2c`.

**B-275 (2026-03-23) — M-BEAUTY-XDUMP ✅:**
`stmt_aref2/aset2` (2D subscripts); `PROTOTYPE` now returns `lo:hi`; `table_set_descr` preserves integer key type through SORT; `expr_flatten_str` for multi-line DEFINE. Semantic driver+ref committed; ASM segfault on DATA/`$'#N'` is B-276 blocker. HEAD `fe86477`.

---

## Beauty Subsystem Status

See [BEAUTY.md](BEAUTY.md) for full sequence. Summary:
- ✅ 1–16: global/is/FENCE/io/case/assign/match/counter/stack/tree/SR/TDump/Gen/Qize/ReadWrite/XDump
- ❌ 17: semantic ← **now**
- ❌ 18: omega
- ❌ 19: trace
