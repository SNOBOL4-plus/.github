# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.**

→ Rules: [RULES.md](RULES.md) · Beauty plan: [BEAUTY.md](BEAUTY.md) · History: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — B-276 (BEAUTY) · F-220 (Prolog) concurrent
**HEAD:** `5e6b872` F-220 prolog (main) / `f721492` B-276 beauty (main)
**B-session:** M-BEAUTY-OMEGA ❌ — driver+ref ready (10/10 CSN+SPL); SPITBOL+SO crash: strip UTF-8 from driver comments
**F-session:** M-PROLOG-R10 ✅ — all 4 puzzles PASS (01/02/05/06); \+ NAF implemented
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — F-222 (M-PROLOG-CORPUS):**

```
BUG: rung05 backtrack FAIL — prints a\nb instead of a\nb\nc.
ROOT CAUSE: prolog_emit.c emit_body, last-goal user-call branch (~line 692).
  When last body goal is a user call, emitter does PG(γ) — jumps to clause γ
  returning the clause index (e.g. 1). Discards inner _cs9 counter.
  On retry, _start=1 resets _cs9=0, re-finds b instead of advancing to c.

FIX (two parts):
  1. emit_body last-goal branch: instead of PG(γ), emit
       *_tr = _trail; return <clause_idx> + _cs<N>;
     (encode inner _cs into return value)
  2. emit_choice switch: add default: case that re-enters last clause's
     retry loop with _start - nclauses as inner _cs.

After fix: all 10 rungs PASS → M-PROLOG-CORPUS fires.
```

---

## Last Two Sessions (3 lines each)

**F-221 (2026-03-23) — bug diagnosis only, no commit:**
Ran all rungs: 1–4 and 6–9 PASS, rung 5 FAIL. Root-caused to `emit_body` last-goal user-call discarding inner `_cs`. Context exhausted before fix applied. HEAD unchanged `5e6b872`.

**F-220 (2026-03-23) — \+ NAF implemented; puzzle_05 PASS:**
`\+` in `prolog_emit.c` was a stub (always succeeded). Fixed: copies trail, tries subgoal via `_r` call, unwinds, succeeds iff subgoal failed. M-PROLOG-R10 ✅. HEAD `5e6b872`.

---

## Beauty Subsystem Status

See [BEAUTY.md](BEAUTY.md) for full sequence. Summary:
- ✅ 1–16: global/is/FENCE/io/case/assign/match/counter/stack/tree/SR/TDump/Gen/Qize/ReadWrite/XDump
- ❌ 17: semantic ← **now**
- ❌ 18: omega
- ❌ 19: trace
