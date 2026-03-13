# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-runtime` (3 of 4 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `8f68962` — fix(sno2c): emit_pat E_DEREF dangling-if, unop left/right contract |

## Last Thing That Happened

**Sprint 2 (`smoke-tests`) COMPLETE — 21/21 PASS.**

Full hand-rolled lex.c + parse.c replacing flex/bison (sno.l + sno.y):
- `lex.c`: Pass 1 joins continuation lines and splits each logical line into
  `(label, body, goto_str)` fields. Pass 2 tokenises with T_WS as the explicit
  binary-vs-unary discriminator. Handles `;` inline comments, computed gotos
  `$('expr')`, FENCE-style label-only statements.
- `parse.c`: Recursive descent, one function per snoExpr0–17 level from
  beauty.sno. Irony architecture: each field parsed independently.
- Binary renamed `snoc` → `sno2c` (SNOBOL4 to C). Directory `src/snoc` → `src/sno2c`.
- Fixed emit.c dangling-if in E_DEREF that silently swallowed `sno_pat_ref()`.
- `beauty_full_bin` builds clean from 12,292 lines of generated C.

## One Next Action

**Sprint 3 (`beauty-runtime`): run `beauty_full_bin < beauty.sno` to completion.**

```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
/tmp/beauty_full_bin < $BEAUTY > /tmp/beauty_compiled.sno 2>&1 | head -30
```

Diagnose with `SNO_PAT_DEBUG=1` and `SNOC_DEBUG=1`. Known runtime fixes already
landed: `DATA()` (e4595a7), `NRETURN`→success (66b7eab), `sno_inc_init()` (627a030).
Sprint 3 ends when binary exits cleanly on beauty.sno input — output may differ
from oracle. Sprint 4 (`beauty-full-diff`) is the empty-diff milestone.

## Rebuild Commands

```bash
cd /home/claude/SNOBOL4-tiny
make -C src/sno2c

# Compile beauty.sno
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
```

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-13 | Sprint 2 (`smoke-tests`) complete — 21/21 | hand-rolled lex/parse works |
| 2026-03-13 | `snoc` renamed `sno2c`; src/snoc → src/sno2c | name reflects function |
| 2026-03-13 | hand-rolled lex.c + parse.c replace flex/bison | grammar confirmed LALR(1) |
| 2026-03-13 | Sprint 1 (`space-token`) PIVOTED, dumb-lexer rewrite begun | architecture fix |
| 2026-03-13 | Sprint 1 (`space-token`) complete → Sprint 2 (`smoke-tests`) active | 0 conflicts |
| 2026-03-13 | M-REBUS fired → `rebus-roundtrip` sprint complete, bf86b4b | Rebus milestone done |
| 2026-03-12 | Bison/Flex → `hand-rolled-parser` decision | Session 53: LALR(1) unfixable (139 RR) |
| 2026-03-12 | M-BEAUTY-FULL inserted before M-COMPILED-SELF | Lon's priority: beautifier first |
