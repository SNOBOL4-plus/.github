# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-first` — fix runtime bugs → M-BEAUTY-FULL |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `93e0fdb` — fix(runtime): engine_stub T_FUNC/T_CAPTURE; SPAT_USER_CALL primitive builtins; UCASE/LCASE/digits pre-init |

---

## ⚡ SESSION 84 FIRST PRIORITY

### The one job:
Fix `aply("c", {x}, 1)` — it is NOT calling `_b_tree_c`. Root cause: `data_define("tree(t,v,n,c)")` in `make_tree()` registers its own field accessors for `t`, `v`, `n`, `c` via `data_define`, which OVERWRITES the `_b_tree_c` registered in `runtime_init`. The `data_define`-generated accessor returns SSTR (type=1) instead of the ARRAY stored in field `c`.

**Exact next action — read data_define:**
```bash
sed -n '964,1000p' src/runtime/snobol4/snobol4.c
# Find: how does data_define register field accessors?
# Expected: it calls define(fieldname, some_fn) for each field
# The fn it registers is a generic field accessor that coerces to string — BUG
```

**The fix:**
Option A — Call `data_define("tree(t,v,n,c)")` in `runtime_init` BEFORE registering
`_b_tree_c`, so our registration wins (comes after, overwrites).

Option B — Remove the `if (!func_exists("t"))` guard + `data_define` call from `make_tree()`.
Register the tree type explicitly in `runtime_init` using the low-level API, then register
`_b_tree_c` manually. `make_tree` just calls `udef_new` directly.

**Option B is cleaner.** In `runtime_init`:
```c
// Register tree UDEF type
data_define("tree(t,v,n,c)");
// Now override field "c" accessor with our version that returns raw SnoVal (ARRAY)
register_fn("c", _b_tree_c, 1, 1);
```
This ensures `_b_tree_c` is always the last registration for `"c"`, wins over whatever
`data_define` installed.

**Verify fix:**
```bash
printf " OUTPUT = 'hello'\n" | /tmp/beauty_tramp_bin 2>&1
# Expected: " OUTPUT = 'hello'" (full line, not just "OUTPUT")
```

**Commit when fixed:** `fix(runtime): register tree type + override c accessor after data_define`

---

## Build command

```bash
apt-get install -y libgc-dev
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
make -C src/sno2c

RT=src/runtime
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno

src/sno2c/sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp.c
gcc -O0 -g /tmp/beauty_tramp.c \
    $RT/snobol4/snobol4.c $RT/snobol4/snobol4_inc.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine_stub.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c \
    -lgc -lm -w -o /tmp/beauty_tramp_bin
```

⚠️ engine_stub.c — NOT engine.c. engine.c is fully superseded.
⚠️ Test input MUST have a leading space: `printf " OUTPUT = 'hello'\n"` not `echo "OUTPUT = 'hello'"`
SNOBOL4 labelless statements start at column 2. Without leading space, `OUTPUT` is parsed as a label.

Oracle: `test/smoke/outputs/session50/beauty_oracle.sno` (790 lines, committed).

---

## Session 83 what was done

| Step | Result |
|------|--------|
| Parse Error investigation | NOT a regression — test input was missing leading space |
| SNOBOL4 format confirmed | Labelless stmts must start at col 2: `" OUTPUT = 'hello'"` |
| `_c` type traced | `aply("c",{x},1)` returns type=1 (SSTR), not type=6 (ARRAY) |
| `_b_tree_c` never called | Added debug trace — it never fired |
| Root cause found | `data_define("tree(t,v,n,c)")` in `make_tree()` overwrites `_b_tree_c` with a coercing accessor |
| Fix identified | In `runtime_init`: call `data_define("tree(t,v,n,c)")` first, then `register_fn("c", _b_tree_c, ...)` to override |
| No commit | Debug code reverted, working tree clean at `93e0fdb` |

---

## What works now
- Comments (`* ...`) — output correctly
- Control lines (`-INCLUDE`) — output correctly
- Simple assignment with leading space — reaches pp_Stmt, outputs label correctly
- UCASE/LCASE/digits — pre-initialized correctly

## Active bug: `aply("c",{x},1)` returns SSTR not ARRAY

**Symptom:** `_c` set in pp_Stmt has type=1 (SSTR). `indx(get(_c), {vint(2)}, 1)` fails.
ppSubj/ppPatrn/ppRepl never set → pp_Stmt outputs only label.

**Root cause:** `make_tree()` calls `data_define("tree(t,v,n,c)")` lazily on first use.
`data_define` registers its own accessor for field `"c"` which coerces the raw SnoVal
to string. This overwrites `_b_tree_c` (registered in `runtime_init`) which returns
the raw SnoVal — preserving ARRAY type.

**Fix:** In `runtime_init`, after all other registrations:
```c
data_define("tree(t,v,n,c)");   // register type first
register_fn("c", _b_tree_c, 1, 1);  // override "c" with our raw accessor
register_fn("t", _b_tree_t, 1, 1);  // same for t, v, n for consistency
register_fn("v", _b_tree_v, 1, 1);
register_fn("n", _b_tree_n, 1, 1);
```
Then remove the `if (!func_exists("t")) { data_define(...); }` guard from `make_tree()`.

---

## CRITICAL Rules

- **NEVER write the token into any file**
- **NEVER link engine.c** — engine_stub.c only, engine.c fully superseded
- **ALWAYS run `git config user.name/email` after every clone**
- **ALWAYS update TINY.md and SESSION.md at HANDOFF**
- **ALWAYS use leading space in test input:** `printf " stmt\n"` not `echo "stmt"`

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-15 | 3-column format `d5b9c3c` | emit_pretty.h shared |
| 2026-03-15 | M-CNODE CNode IR `160f69b`+`ac54bd2` | proper pp/qq architecture |
| 2026-03-15 | Return to M-BEAUTY-FULL | M-CNODE done, back to main line |
| 2026-03-14 | `0113d90` pat_lit fix | emit_cnode.c build_pat E_STR strv() removed |
| 2026-03-14 | Session 78 TINY.md/SESSION.md rewrite | both were severely stale |
| 2026-03-14 | Session 80 runtime fixes | engine_stub T_FUNC/T_CAPTURE; SPAT_USER_CALL builtins; UCASE/LCASE/digits |
| 2026-03-14 | Session 83 diagnosis | Parse Error = test format bug; _c = data_define overwrites _b_tree_c |
