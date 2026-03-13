# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `space-token` (1 of 4 toward M-BEAUTY-FULL) — PIVOTED, still in progress |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `cd64930` — EMERGENCY WIP: dumb-lexer rewrite — __ token, _ optional gray nonterminal, 8 SR conflicts remain |

## Last Thing That Happened

**Sprint 1 (`space-token`) PIVOTED mid-session.** Commit `3581830` (0 conflicts) was architecturally wrong — it used `%nonassoc SUBJ` and `subject → term` as lexer-layer hacks. Lon's S4_expression.sno (beauty.sno's own expression grammar) was read and is the spec.

**Correct architecture (dumb-lexer):**
- Lexer emits `__` for ANY whitespace run (spaces/tabs). No lookahead, no state, no conditions. Continuation lines already collapsed to one space by join_file → one `__` token.
- Grammar: `__` is the mandatory-space token (terminal). `_ : __ | ε` is the optional/gray nonterminal.
- Binary operators: `expr __ OP __ term` (space required both sides, per beauty.sno `$'+'` definition)
- Concat: `expr __ term`
- Unary: `OP term` (no leading space)
- Inside parens/brackets: `_` (gray)
- Statement: `label __ subject __ pattern` — both separators are `__`

**Current state:** sno.l rewrite complete. sno.y rewrite complete. `bison -d sno.y` → **8 SR conflicts, 0 RR**. All 8 are the same: `_` (optional) next to `__` (mandatory) — bison can't decide on incoming `__` token whether it belongs to `_` or starts the next mandatory field.

**Root cause of 8 SR:** `_ : __ | ε` means wherever `_` appears adjacent to a `__`-consuming production, LALR(1) shift/reduce conflict fires. The user's last suggestion: inline `_` out of existence — duplicate every rule that uses `_` into a with-space and without-space variant.

## One Next Action

**Eliminate `_` nonterminal — inline it by rule duplication.**

Every rule using `_` gets duplicated: one with `__`, one without. For example:

```yacc
/* instead of:  LPAREN _ arglist _ RPAREN */
LPAREN arglist RPAREN
LPAREN __ arglist RPAREN
LPAREN arglist __ RPAREN
LPAREN __ arglist __ RPAREN
```

This is mechanical but eliminates all SR conflicts because there's no longer a nonterminal competing with `__`. Check: `bison -d sno.y` → 0 conflicts. Then build and run smoke tests.

Places that use `_` currently:
- `LPAREN _ arglist _ RPAREN` (function call)
- `LPAREN _ expr _ RPAREN` (grouped expr)
- `LPAREN _ expr COMMA _ arglist_ne _ RPAREN` (alternation)
- `LBRACKET _ arglist _ RBRACKET` (subscript)
- `_ COLON _ goto_clauses` (opt_goto)
- `arglist_ne _ COMMA _ expr` (arglist separator)
- `SGOTO _ LPAREN _ glabel _ RPAREN` (goto clauses)
- `FGOTO _ LPAREN _ glabel _ RPAREN`
- `_ LPAREN _ glabel _ RPAREN` (unconditional goto)

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | Sprint 1 PIVOTED: `3581830` (0 conflicts) declared wrong, dumb-lexer rewrite begun | `3581830` used `%nonassoc SUBJ` + `subject→term` hacks; S4_expression.sno is the real spec |
| 2026-03-13 | Lexer token renamed `__`; `_ : __ | ε` as optional gray | Lon suggestion: readability, `__` mandatory, `_` optional |
| 2026-03-13 | Sprint 1 (`space-token`) complete → Sprint 2 (`smoke-tests`) active | 0 conflicts achieved |
| 2026-03-13 | `_` token name (was `SPACE`) | Lon suggestion, cleaner |
| 2026-03-13 | `hand-rolled-parser` → 4-sprint `space-token` plan | SPACE token resolves LALR(1) conflicts without parser rewrite |
| 2026-03-13 | HQ PLAN.md rewritten with 4 correct sprints | Previous session had wrong 5-sprint list |
| 2026-03-13 | M-REBUS fired → `rebus-roundtrip` sprint complete, bf86b4b | Rebus milestone done |
| 2026-03-13 | `hand-rolled-parser` paused → `rebus-emitter` active | Lon declared Rebus priority |
| 2026-03-12 | Bison/Flex → `hand-rolled-parser` decision | Session 53: LALR(1) unfixable (139 RR conflicts) |
| 2026-03-12 | M-BEAUTY-FULL inserted before M-COMPILED-SELF | Lon's priority: beautifier first |
