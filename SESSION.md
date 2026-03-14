# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `pattern-block` (sprint 4/9 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `c5d5c2b — fix(emit+parse): computed goto inline dispatch` |

---

## State at handoff (session 71)

Commits this session:
- `c5d5c2b` — fix(emit+parse): computed goto inline dispatch — $COMPUTED:expr now works in fn-body mode

**CSNOBOL4 2.3.3 NOT pre-installed** — must build from tarball at session start.
Tarball is at `/mnt/user-data/uploads/snobol4-2_3_3_tar.gz` (uploaded by Lon).
Install: `cd /tmp && tar xzf /mnt/user-data/uploads/snobol4-2_3_3_tar.gz && cd snobol4-2.3.3 && ./configure --prefix=/usr/local && make -j$(nproc) && make install`
Then oracle is at `/usr/local/bin/snobol4`.

### Three bugs fixed this session

**Bug 1 — emit.c:** `emit_computed_goto_inline()` was missing. Classic fn-body
mode `$COMPUTED:expr` labels fell through silently. Now emits a full strcmp
chain over all labels in the function's body (pp_Parse, pp_Stmt, pp_Label, etc).

**Bug 2 — parse.c capture:** `parse_goto_label()` captured one character too
many — the closing `)` of `$(...)` was included in the expr text. Fixed by
stripping trailing `)` and whitespace in `emit_computed_goto_inline()` before
calling `parse_expr_from_str()`.

**Bug 3 — parse.c prime:** `parse_expr_from_str()` called `lex_next()` to
"prime" the lexer, discarding the first token. `lex_peek()` is self-priming —
no prime needed. Removed the call.

### Current symptom

`$('pp_' t)` now correctly dispatches into `pp()` sub-labels. The dispatch
chain is emitted and correct. However:

| Input | Compiled | Oracle | Status |
|-------|----------|--------|--------|
| `* comment` | `* comment` | `* comment` | ✅ MATCH |
| `START` | `Parse Error\nSTART` | `START` | ❌ wrong |
| `X = 1` | `Parse Error\nX = 1` | `Parse Error\nX = 1` | ✅ MATCH |
| `label OUTPUT = 'hello'` | `Parse Error\nlabel OUTPUT = 'hello'` | `label          OUTPUT         =  'hello'` | ❌ wrong |

Self-beautification: oracle 162 lines, compiled 30 lines. Binary stops after
the -INCLUDE block and START.

### Root cause of remaining failure — SUSPECTED

`START` is a label-only statement (no pattern, no replacement). Its parse tree
node has type `"Label"`. `pp()` dispatches `$('pp_' t)` → `pp_Label`.

The `Parse Error` output is coming from the **parser** (`snoParse`), not from
`pp()`. The binary is outputting `Parse Error` for the `START` line because
`snoParse` fails to parse it as a SNOBOL4 statement, not because `pp()` fails.

**Evidence:** `X = 1` also outputs `Parse Error\nX = 1` and the oracle does too
— so Parse Error for X=1 is **correct** (beauty.sno prints "Parse Error" for
lines it can't parse, then falls back). But for `START` the oracle outputs just
`START` with no `Parse Error`. So snoParse correctly parses `START` as a label
statement in the oracle, but our compiled binary's snoParse fails on it.

### ONE NEXT ACTION — Session 72

Add a `printf` to the trampoline entry of `snoParse` block function to trace
what input it receives for the `START` line:

**Step 1:** Search generated `beauty_tramp.c` for `block_snoParse` and find
where it processes the input line. Add:
```c
fprintf(stderr, "snoParse input: [%s]\n", to_str(get(_INPUT)));
```
just after the block entry. Recompile and run `printf 'START\n' | ./beauty_tramp_bin`.

**Step 2:** Check what `snoParse` does differently for `START` vs `* comment`.
`* comment` succeeds (type=Comment). `START` should succeed (type=Label).
If snoParse fails on `START`, it means the SNOBOL4 pattern for label-only
statements isn't matching.

**Step 3:** The issue is likely in `snoLabel` pattern. In beauty.sno, `snoLabel`
matches `IDENT ':'` or similar. `START` has no `:`. A bare identifier at the
start of a line IS a label in SNOBOL4. Check that `snoLabel` pattern handles
bare label (no colon suffix) — it should match `START` as a valid label.

**Alternative approach:** Compare `snoParse` Byrd box behavior directly.
Run `snobol4 -f -P256k -I $INC $BEAUTY` interactively with debug output.
Or: add trampoline_stno calls to trace which block_snoParse* block runs.

---

## Build command

```bash
# Install oracle (ONCE per container)
cd /tmp && tar xzf /mnt/user-data/uploads/snobol4-2_3_3_tar.gz
cd /tmp/snobol4-2.3.3 && ./configure --prefix=/usr/local && make -j$(nproc) && make install

# Clone repos
apt-get install -y m4 libgc-dev
TOKEN=TOKEN_SEE_LON
git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-tiny /home/claude/SNOBOL4-tiny
git clone https://x-access-token:${TOKEN}@github.com/SNOBOL4-plus/SNOBOL4-corpus /home/claude/SNOBOL4-corpus
git -C /home/claude/SNOBOL4-tiny config user.name "LCherryholmes"
git -C /home/claude/SNOBOL4-tiny config user.email "lcherryh@yahoo.com"

# Build sno2c
cd /home/claude/SNOBOL4-tiny/src/sno2c && make

# Generate + compile beauty_tramp_bin
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
RT=/home/claude/SNOBOL4-tiny/src/runtime
SNO2C=/home/claude/SNOBOL4-tiny/src/sno2c
$SNO2C/sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp.c
gcc -O0 -g -I$SNO2C -I$RT -I$RT/snobol4 \
    /tmp/beauty_tramp.c $RT/snobol4/snobol4.c $RT/snobol4/snobol4_inc.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine.c -lgc -lm -w -o /tmp/beauty_tramp_bin

# Test
printf '* comment\n' | /tmp/beauty_tramp_bin
printf 'START\n'     | /tmp/beauty_tramp_bin
printf 'X = 1\n'     | /tmp/beauty_tramp_bin

# Oracle self-beautify
/usr/local/bin/snobol4 -f -P256k -I$INC $BEAUTY < $BEAUTY > /tmp/oracle_out.sno
/tmp/beauty_tramp_bin < $BEAUTY > /tmp/compiled_out.sno
diff /tmp/oracle_out.sno /tmp/compiled_out.sno | head -40
```

---

## Artifact convention

Next artifact: `beauty_tramp_session71.c` (generate after snoParse/Label bug fixed)

---

## CRITICAL Rules

- **NEVER write the token into any file**
- **NEVER link engine.c in beauty_full_bin**
- **ALWAYS run `git config user.name/email` after every clone**

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-15 | DATA tree/link startup `50ef58f` | START passes; X=1 loops |
| 2026-03-15 | nPush β→ω `6abfdf6` | X=1 infinite loop eliminated |
| 2026-03-15 | ARBNO beta nhas_frame `27325b6` | ntop counts correctly |
| 2026-03-15 | @S checkpoint ARBNO `emit_arbno` | @S stack pollution fixed |
| 2026-03-15 | @S checkpoint per-stmt `emit.c` | per-stmt @S save/restore |
| 2026-03-15 | computed goto infrastructure `e8f9e5d` | $COMPUTED:expr preserved; dispatch TODO |
| 2026-03-15 | computed goto inline dispatch `c5d5c2b` | $COMPUTED now dispatches correctly; snoParse/Label next |
