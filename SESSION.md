# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-full-diff` (4 of 4 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `6665384` — EMERGENCY WIP: LexMark save/restore — concat fixed but parse_expr0 restore breaks pattern parse |

## Last Thing That Happened

**Root cause of concat truncation found and partially fixed.**

All binary op levels in parse.c (`parse_lbin`, `parse_rbin`, `parse_expr12`,
`parse_expr13`, `parse_expr3`, `parse_expr0`) were calling `skip_ws()` after
consuming WS for lookahead. `skip_ws` calls `lex_peek` which calls `raw_next`,
advancing `lx->pos` past the next token. When the next token wasn't the expected
operator and the level "put WS back" via synthetic peek injection, `pos` was already
past that token. Each level stacked this drift — 8 levels deep meant pos=EOF by the
time `parse_expr4`'s concat loop fired. `'a' 'b' 'c'` compiled to just `"a"`.

**Fix applied:** Added `LexMark` (pos+peek+peeked save/restore) to `lex.h`. Fixed
all lookahead sites: `parse_lbin`, `parse_rbin`, `parse_expr12`, `parse_expr13`,
`parse_expr3`. Basic concat `'a' 'b' 'c'` now emits `concat(concat("a","b"),"c")`.

**Regression introduced:** Also applied `LexMark` to `parse_expr0`, which was wrong.
`parse_expr0` is called inside `parse_arglist` and pattern contexts. Its restore
fires when WS is followed by anything other than `=` or `?` — which is legal in
those contexts (e.g., before `,` or `)`). Result: all 21 smoke tests fail with
`Parse Error`. `beauty_full_bin < beauty.sno` outputs only 10 lines then Parse Error.

## One Next Action

**Remove LexMark from `parse_expr0` only.** `parse_expr0` is NOT a lookahead site
like the others — it handles top-level `=` and `?` which `parse_body_field` already
handles explicitly at the statement level. The WS+non-op case in `parse_expr0` should
just put WS back via the old synthetic injection (or simply not consume it at all),
NOT restore the full LexMark.

```c
// In parse_expr0 — REVERT to old behavior:
static Expr *parse_expr0(Lex *lx) {
    Expr *l = parse_expr2(lx);
    if (!l) return NULL;
    if (lex_peek(lx).kind != T_WS) return l;
    lex_next(lx);  // consume WS tentatively
    TokKind k = lex_peek(lx).kind;
    if (k == T_EQ) {
        lex_next(lx); skip_ws(lx);
        Expr *r = parse_expr0(lx);
        return binop(E_ASSIGN, l, r);
    }
    if (k == T_QMARK) {
        lex_next(lx); skip_ws(lx);
        Expr *r = parse_expr0(lx);
        return binop(E_COND, l, r);
    }
    // Put WS back — but do NOT use LexMark (would restore past real token)
    // Instead: inject synthetic T_WS. pos is only one peek ahead here
    // because we did NOT call skip_ws after lex_next(WS).
    lx->peek = (Token){T_WS, NULL, 0, 0, lx->lineno};
    lx->peeked = 1;
    return l;
}
```

This is safe because `parse_expr0` only peeks ONE token ahead (no `skip_ws` call),
so the synthetic T_WS injection only loses one position step — and `pos` is only
past the ONE token we peeked (not cascaded through 8 levels).

After reverting parse_expr0:
1. Rebuild sno2c
2. Run smoke tests → should be 21/21 again
3. Recompile beauty.sno, rebuild binary
4. Run diff against oracle

## Rebuild Commands

```bash
cd /home/claude/SNOBOL4-tiny

# Rebuild sno2c
make -C src/sno2c

# Smoke test
bash test/smoke/test_snoCommand_match.sh /tmp/beauty_full_bin

# Recompile beauty.sno
src/sno2c/sno2c \
  /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
  -I /home/claude/SNOBOL4-corpus/programs/inc \
  > /tmp/beauty_full.c

# Build binary
R=src/runtime/snobol4
gcc -O0 -g /tmp/beauty_full.c \
    $R/snobol4.c $R/snobol4_inc.c $R/snobol4_pattern.c \
    src/runtime/engine.c \
    -I$R -Isrc/runtime -lgc -lm -w \
    -o /tmp/beauty_full_bin

# Oracle
SNO=/home/claude/snobol4-2.3.3/snobol4
$SNO -f -P256k -I /home/claude/SNOBOL4-corpus/programs/inc \
    /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
    < /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno \
    > /tmp/beauty_oracle.sno 2>/dev/null

# Diff
diff /tmp/beauty_oracle.sno /tmp/beauty_compiled.sno
```

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | EMERGENCY: LexMark fixes concat but parse_expr0 breaks patterns | context full |
| 2026-03-13 | Sprint 3 (`beauty-runtime`) complete — clean exit | first run worked |
| 2026-03-13 | Sprint 2 (`smoke-tests`) complete — 21/21 | hand-rolled lex/parse works |
| 2026-03-13 | `snoc` renamed `sno2c`; src/snoc → src/sno2c | name reflects function |
| 2026-03-13 | hand-rolled lex.c + parse.c replace flex/bison | grammar confirmed LALR(1) |
| 2026-03-13 | Sprint 1 (`space-token`) PIVOTED, dumb-lexer rewrite begun | architecture fix |
| 2026-03-13 | Sprint 1 (`space-token`) complete → Sprint 2 (`smoke-tests`) active | 0 conflicts |
| 2026-03-13 | M-REBUS fired → `rebus-roundtrip` sprint complete, bf86b4b | Rebus milestone done |
| 2026-03-12 | Bison/Flex → `hand-rolled-parser` decision | Session 53: LALR(1) unfixable (139 RR) |
| 2026-03-12 | M-BEAUTY-FULL inserted before M-COMPILED-SELF | Lon's priority: beautifier first |
