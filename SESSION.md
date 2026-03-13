# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `hand-rolled-parser` (resuming) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `bf86b4b` — milestone(M-REBUS): Rebus round-trip complete ✅ |

## Last Thing That Happened

**M-REBUS FIRED.** `rebus-roundtrip` sprint complete.

Deliverables committed (`bf86b4b`):
- `test/rebus/run_roundtrip.sh` — emit → CSNOBOL4 → diff harness
- `test/rebus/word_count.{expected,input}` — oracle files
- `test/rebus/binary_trees.{expected,input}` — oracle files
- `src/rebus/Makefile` — added `roundtrip` target

Both tests PASS: `make -C src/rebus roundtrip` → 2 passed, 0 failed.

Push pending (no GitHub credentials in container). Commit is local at `bf86b4b`.

## One Next Action

Resume `hand-rolled-parser` sprint toward M-BEAUTY-FULL:
1. Read `src/snoc/` — current state of bison/flex parser
2. Write `src/snoc/lex.c` (~200 lines) — flat `sno_charclass[256]`
3. Write `src/snoc/parse.c` (~500 lines) — `parse_expr()` and `parse_pat_expr()` as separate functions
4. Update `src/snoc/Makefile` — remove bison/flex deps
5. Build → compile `beauty.sno` → `sno_apply("snoWhite",...)` count = 0
6. Smoke tests: 0/21 → 21/21

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | M-REBUS fired → `hand-rolled-parser` resumes | `rebus-roundtrip` sprint complete, bf86b4b |
| 2026-03-13 | `rebus-emitter` complete → `rebus-roundtrip` active | Sprint finished |
| 2026-03-13 | Branding/rename session — RENAME.md created, naming rules locked | Lon pivot before public launch |
| 2026-03-13 | `hand-rolled-parser` paused → `rebus-emitter` active | Lon declared Rebus priority |
| 2026-03-12 | Bison/Flex → `hand-rolled-parser` decision | Session 53: LALR(1) unfixable (139 RR conflicts) |
| 2026-03-12 | M-BEAUTY-FULL inserted before M-COMPILED-SELF | Lon's priority: beautifier first |
