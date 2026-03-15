# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-first` — fix Parse Error → M-BEAUTY-CORE |
| **Milestone** | M-BEAUTY-CORE (stubs first) → M-BEAUTY-FULL (real inc, second) |
| **HEAD** | `8676bd9` — refactor: restore proper English names — undo P4 misspelling technique |

---

## ⚡ SESSION 87 FIRST ACTION

### Active bug: Parse Error during beautification

**IMPORTANT:** `-INCLUDE` lines are handled by `sno2c` at **compile time** via
`-I inc_mock`. They never appear in the runtime input stream. The lexer ignores
them. Do NOT chase `-INCLUDE` as a runtime issue — it is not one.

beauty_core_bin is built with `sno2c -trampoline -I inc_mock beauty.sno`.
The mock `.sno` files in `inc_mock/` are comment-only — sno2c sees them and
moves on. The compiled binary has no knowledge of INCLUDE directives.

**The actual bug:** beauty_core_bin hits `Parse Error` (mainErr1 — line 796 in
beauty.sno) when fed beauty.sno as input to beautify. Simple input like
` OUTPUT = 'hello'` works. The failure is somewhere in the pattern matching
of actual beauty.sno statement forms.

**Session 87 first action:**
1. Build the binary (see Build commands below)
2. Run on simple input to confirm it works: `printf " OUTPUT = 'hello'\n" | /tmp/beauty_core_bin`
3. Run on beauty.sno itself and capture output: `/tmp/beauty_core_bin < $BEAUTY 2>&1 | head -20`
4. Narrow down which statement form triggers Parse Error

---

## Build commands

```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
apt-get install -y libgc-dev
make -C src/sno2c

RT=src/runtime
STUBS=src/runtime/inc_mock
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno

# beauty_core (stubs — USE THIS, DO NOT switch to beauty_full)
src/sno2c/sno2c -trampoline -I$STUBS $BEAUTY > /tmp/beauty_core.c
gcc -O0 -g /tmp/beauty_core.c \
    $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine_stub.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c \
    -lgc -lm -w -o /tmp/beauty_core_bin
```

⚠️ engine_stub.c — NOT engine.c  
⚠️ Test input MUST have leading space: `printf " stmt\n"` not `echo "stmt"`  
⚠️ beauty_core (stubs) FIRST — beauty_full (real inc) only after M-BEAUTY-CORE fires  

Oracle: `test/smoke/outputs/session50/beauty_oracle.sno`

---

## What was done this session (Session 85)

### Agreement breach resolved
Session 84 broke the beauty_core/beauty_full agreement. Session 85 confirmed
`inc_mock/` is intact (19 stubs), both binaries build clean.

### Rename audit — Session 84 SIL rename verified
Full word-for-word audit of 40+ renames. All clean. One bug found and fixed:
`ARRAY_VAL` macro used `.a` instead of `.arr` — dormant (never called), fixed.
Full audit written to HQ PLAN.md.

### M-BEAUTY-CORE / M-BEAUTY-FULL split written into HQ
PLAN.md, TINY.md, SESSION.md all updated. The two-phase agreement is now
a hard architectural rule in HQ, not just a session note.

### P4 misspelling technique fully undone
ALLCAPS_fn suffix provides its own namespace — misspellings no longer needed.
18 names restored to proper English:

| Old | New |
|-----|-----|
| APLY_fn | APPLY_fn |
| CONC_fn | CONCAT_fn |
| ccat (char*) | STRCONCAT_fn |
| RPLACE_fn | REPLACE_fn |
| evl | EVAL_fn |
| divyde | DIVIDE_fn |
| powr | POWER_fn |
| entr | ENTER_fn |
| xit | EXIT_fn |
| abrt | ABORT_fn |
| indx | INDEX_fn |
| replc | REPLACE_fn |
| mtch | MATCH_fn |
| strv | STRVAL_fn |
| vint | INTVAL_fn |
| ccat | CONCAT_fn |
| dupl (char*) | STRDUP_fn |
| ini | INIT_fn |

Also fixed: SNOBOL4 registration strings that had picked up `_fn` suffix
from Session 84 rename: `"SIZE"`, `"DUPL"`, `"TRIM"`, `"SUBSTR"`, `"DATA"`,
`"FAIL"`, `"DEFINE"`.

### Debug traces
Stripped bare debug traces from `_b_tree_c`, `APLY_fn(c)`, `MAKE_TREE_fn`.
Single trace added in `FIELD_GET_fn` — result: trace never fires on simple
input, meaning stmt_205 (which calls `APPLY_fn("c",...)`) is never reached.
Parse Error fires before the tree walk. Fix Parse Error first.

---

## Active bug: Parse Error during beautification

**What is known:**
- Simple ` OUTPUT = 'hello'` input works fine (output: `OUTPUT`)
- beauty_core_bin hits Parse Error on actual beauty.sno input
- `-INCLUDE` is a compile-time directive handled by sno2c — NOT a runtime issue
- The FIELD_GET_fn / _c field bug is SECONDARY — unreachable until parsing works

---

## CRITICAL Rules

- **NEVER write the token into any file**
- **NEVER link engine.c** — engine_stub.c only
- **ALWAYS run `git config user.name/email` after every clone**
- **ALWAYS use leading space in test input:** `printf " stmt\n"` not `echo "stmt"`
- **beauty_core (stubs) FIRST — beauty_full (real inc) SECOND**

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-14 | Session 80 runtime fixes | engine_stub T_FUNC/T_CAPTURE etc |
| 2026-03-14 | Session 83 diagnosis | _c = data_define overwrites _b_tree_c (later disproved) |
| 2026-03-14 | Session 84 SIL rename | DESCR_t/DTYPE_t/XKIND_t/_fn/_t throughout |
| 2026-03-14 | Session 84 build fixes | cs_alloc, computed goto, label table, inc_mock |
| 2026-03-14 | Session 84 HALT | broke beauty_core/beauty_full agreement — reverted to stubs |
| 2026-03-14 | Session 85 cleanup | agreement breach resolved, rename audit, P4 undo, M-BEAUTY-CORE split |
