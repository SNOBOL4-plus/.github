# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `smoke-tests` (2 of 4 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `3581830` — feat(snoc): space-token — 0 bison conflicts, unified grammar ✅ |

## Last Thing That Happened

**Sprint 1 (`space-token`) COMPLETE.** 159 conflicts → 0. Clean build. Committed and pushed.

Key decisions made this session:
- `SPACE` token renamed to `_` (Lon's suggestion — bison allows it, zero warnings)
- Subject in stmt rules restricted to `term` (not `expr`) — eliminates the core 106 RR conflicts. First space always separates subject from pattern per SNOBOL4 semantics. Concat in subject requires parens.
- `bstack`/`last_was_callable`/`PAT_BUILTIN`/`is_pat_builtin` fully eliminated from sno.l
- `opt_expr → empty` removed — arglist empty handled by `arglist → empty` only
- `IDENT[...]` array syntax kept only in `atom` rule (removed from `primary`)
- `%nonassoc SUBJ` precedence resolves remaining SR conflicts
- Unreachable `<GT>.` catch-all removed from sno.l

## One Next Action

**Start Sprint 2 (`smoke-tests`):**

1. Clone SNOBOL4-corpus (needed for beauty.sno and inc/):
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-corpus.git
```

2. Find and build `beauty_full_bin`:
```bash
find SNOBOL4-tiny -name Makefile | xargs grep -l beauty 2>/dev/null
```

3. Run the smoke test harness:
```bash
bash SNOBOL4-tiny/test/smoke/test_snoCommand_match.sh /tmp/beauty_full_bin
```

4. Drive from 0/21 → 21/21. Each failure = one parser/emitter routing bug.

5. **Commit when:** All 21 pass. Zero "Parse Error" lines.

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | Sprint 1 (`space-token`) complete → Sprint 2 (`smoke-tests`) active | 0 conflicts achieved |
| 2026-03-13 | `_` token name (was `SPACE`) | Lon suggestion, cleaner |
| 2026-03-13 | `hand-rolled-parser` → 4-sprint `space-token` plan | SPACE token resolves LALR(1) conflicts without parser rewrite |
| 2026-03-13 | HQ PLAN.md rewritten with 4 correct sprints | Previous session had wrong 5-sprint list |
| 2026-03-13 | M-REBUS fired → `rebus-roundtrip` sprint complete, bf86b4b | Rebus milestone done |
| 2026-03-13 | `hand-rolled-parser` paused → `rebus-emitter` active | Lon declared Rebus priority |
| 2026-03-12 | Bison/Flex → `hand-rolled-parser` decision | Session 53: LALR(1) unfixable (139 RR conflicts) |
| 2026-03-12 | M-BEAUTY-FULL inserted before M-COMPILED-SELF | Lon's priority: beautifier first |
