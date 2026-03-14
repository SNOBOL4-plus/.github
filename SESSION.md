# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `trampoline` (new sprint 1/9 — see PLAN.md) |
| **Milestone** | M-TRAMPOLINE |
| **HEAD** | `bf44d5f — WIP: named pattern registry + E_DEREF compiled path (session 55)` |

---

## MAJOR ARCHITECTURE PIVOT — Session 55

Session 55 was architecture. No milestone code. But the architecture is now
**complete and correct**. Everything changed. Read PLAN.md fully before touching
any code.

### The New Model (decided with Lon Cherryholmes, 2026-03-14)

**Every SNOBOL4 statement** → its own C function `stmt_N()` returning a
`block_fn_t` — the address of the next block to execute (S-label on success,
F-label on failure). Not true/false. Actual addresses.

**Statements grouped into block functions** `block_L()` by label reachability:
- New block starts at every labeled statement
- Unlabeled statements that follow are unreachable from outside — same block
- Block function calls member stmts in sequence, returns on escape

**The trampoline IS the engine:**
```c
block_fn_t pc = block_START;
while (pc) pc = pc();
```
One loop. No interpreter. No engine.c. No dispatch table.

**GOTO / \*X / EVAL / CODE** all collapse to `block_fn_t` calls:
- `:(L42)` → return `block_L42`
- `*X` (static) → call `block_X` — compiled address
- `*X` (dynamic) → X holds a `block_fn_t` — call it
- `EVAL(str)` → TCC compile → `block_fn_t` (works for expressions AND patterns)
- `CODE(str)` → TCC compile → `block_fn_t` entry

**EVAL can eval a PATTERN.** A pattern IS a block function. EVAL and CODE
are the same mechanism — compile str via TCC, return the address.

**Flat locals struct per block** — all locals for all stmts in a block
concatenated into one struct. One allocation per block invocation.

### What was recorded in HQ this session

- `arch: iota function flat-model kludge` (e05f607)
- `arch: four techniques for Byrd box implementation` (0ef0b3f)
- `plan: full incremental sprint map — block-fn + trampoline model` (33964f5)
- `plan: CODE() via TCC in-process compile` (d173f38)
- `plan: EVAL unifies with CODE — EVAL can eval a PATTERN` (558e207)

---

## ONE NEXT ACTION — Start sprint `trampoline`

Build the proof-of-concept that the trampoline model works end-to-end
on a hello-world SNOBOL4 program.

### What to build

1. **Core types** in a new file `src/sno2c/trampoline.h`:
```c
typedef struct _block_fn (*block_fn_t)(void);
```

2. **Hand-write a hello-world** in the new model — two stmts, one block:
```c
/* SNOBOL4: OUTPUT = 'hello world' */
static block_fn_t stmt_1(void) {
    /* pure assignment — no pattern */
    sno_output(STR_VAL("hello world"));
    return NULL;  /* end of program */
}
static block_fn_t block_START(void) {
    block_fn_t next = stmt_1();
    return next;
}
int main(void) {
    block_fn_t pc = block_START;
    while (pc) pc = pc();
    return 0;
}
```

3. **Compile and run** — confirm output is `hello world`.

4. **Commit** — M-TRAMPOLINE fires.

### Then: sprint `stmt-fn`

Wire `emit.c` to emit `stmt_N()` functions instead of inline code.
Each stmt gets its own C function. setjmp guard. Returns `block_fn_t`.

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT write the TOKEN into any file
- Do NOT touch the old emit_byrd.c struct-passing code — it is superseded
- Do NOT link engine.c — ever
- Do NOT start coding before reading PLAN.md fully

---

## Container State (clone fresh each session)

```bash
apt-get install -y m4 libgc-dev
git config user.name "LCherryholmes"
git config user.email "lcherryh@yahoo.com"
TOKEN=TOKEN_SEE_LON
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
git clone https://$TOKEN@github.com/SNOBOL4-plus/.github.git dotgithub

cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
./configure --prefix=/home/claude/snobol4-install && make -j$(nproc)
cd ..

cd SNOBOL4-tiny && make -C src/sno2c
```

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | sno_output_str fix (e78d177) | linker error |
| 2026-03-14 | M-PYTHON-UNIFIED retired → M-BYRD-SPEC | Python was scaffold |
| 2026-03-14 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c + ALL OK |
| 2026-03-15 | fn_seen[] + byrd_fn_scope_reset() | static redecl errors |
| 2026-03-15 | E_DEREF with match_pattern_at() | beauty uses *varname ~100x |
| 2026-03-15 | binary links (91d097c) | 3 gcc bugs fixed |
| 2026-03-15 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c only |
| 2026-03-16 | Strip sno_/SNO_ prefix (3ea9815) | readability |
| 2026-03-14 | artifact: session54.c (35bc142) | arch-only session |
| 2026-03-14 | **PIVOT: block-fn + trampoline model** | complete architecture rethink with Lon — everything before superseded |
| 2026-03-14 | Named pattern registry added to emit_byrd.c (bf44d5f) | partial work, superseded by new model |
