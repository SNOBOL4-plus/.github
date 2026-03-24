# FRONTEND-ICON-JVM.md — Tiny-ICON → JVM Backend (L3)

Tiny-ICON frontend targeting JVM bytecode via Jasmin.
Reuses the existing Icon pipeline (lex → parse → AST) unchanged.
New layer: `icon_emit_jvm.c` — consumes `IcnNode*` AST and emits Jasmin `.j` files,
assembled by `jasmin.jar` into `.class` files.

**Session trigger phrase:** `"I'm working on Icon JVM"`
**Session prefix:** `IJ` (e.g. IJ-1, IJ-2, IJ-3)
**Driver flag:** `icon_driver -jvm foo.icn -o foo.j` → `java -jar jasmin.jar foo.j -d .` → `java FooClass`
**Oracle:** `icon_driver foo.icn -o foo.asm -run` (the x64 ASM backend, rungs 1–2 known good)

*Session state → this file §NOW. Backend reference → BACKEND-JVM.md.*

---

## §NOW — Session State

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **Icon JVM** | `main` IJ-6 — Fix1 slot-type VerifyError ✅; Fix2 bdone drain ✅; Fix3 sdrain ✅; remaining: body ω path routes through pop2 (empty stack) — one-line fix needed | `d169d6f` IJ-6 | M-IJ-CORPUS-R3 |

### Next session checklist (IJ-7)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x
gcc -Wall -Wextra -g -O0 -I. src/frontend/icon/icon_driver.c src/frontend/icon/icon_lex.c     src/frontend/icon/icon_parse.c src/frontend/icon/icon_ast.c     src/frontend/icon/icon_emit.c src/frontend/icon/icon_emit_jvm.c     src/frontend/icon/icon_runtime.c -o /tmp/icon_driver
# Read FRONTEND-ICON-JVM.md §NOW — HEAD = d169d6f
# rung01 6/6 + rung02 14/14 already pass
# One fix remains — see IJ-6 findings below — then fire M-IJ-CORPUS-R3
```

### IJ-6 findings

**Bug 1 — FIXED (d169d6f):** `lconst_0; lstore N` preamble for all `ij_nlocals` long
slots emitted in `ij_emit_proc` before the `getstatic icn_suspend_id / ifne beta`
dispatch. Slot-type VerifyError `Register pair 2/3 contains wrong type` is dead.

**Bug 3 (inter-stmt drain) — FIXED (d169d6f):** Top-level statement γ port now routes
through `icn_sN_sdrain: pop2; goto next_a` instead of jumping directly to the next
statement's α, which was also entered empty-stack from other paths.

**Bug 2 (body drain) — FIXED structure, one refinement needed (IJ-7):**
`icn_N_bdone: pop2; goto ports.γ` correctly drains the body γ result.
**BUT:** `bp.ω` (body failure, empty stack) is also routed through `icn_N_bdone`,
causing `pop2` on an empty stack → `Inconsistent stack height 0 != 2` persists.

**IJ-7 one-line fix** in `ij_emit_suspend`, in the `if (body_node)` block:
```c
// WRONG (current):
strncpy(bp.ω, body_done, 63);  /* routes empty-stack failure through pop2 */

// CORRECT:
strncpy(bp.ω, ports.γ, 63);   /* failure has empty stack — jump direct, no pop */
```
After this change rebuild → jasmin → java T01_gen should print 1 2 3 4 and rung03 passes.
The exact location is `ij_emit_suspend`, the `IjPorts bp` block inside `if (body_node)`,
around line 520 of `icon_emit_jvm.c`.

---


## Key Files

| File | Role |
|------|------|
| `src/frontend/icon/icon_emit_jvm.c` | **TO CREATE** — this sprint's deliverable |
| `src/frontend/icon/icon_emit.c` | x64 ASM emitter — Byrd-box logic oracle (49KB) |
| `src/frontend/icon/icon_driver.c` | Add `-jvm` flag → `ij_emit_file()` branch |
| `src/backend/jvm/emit_byrd_jvm.c` | JVM output format oracle — copy helpers verbatim |
| `src/backend/jvm/jasmin.jar` | Assembler — `java -jar jasmin.jar foo.j -d outdir/` |
| `test/frontend/icon/corpus/` | Same `.icn` tests; oracle = ASM backend output |

---

## Oracle Comparison Strategy

```bash
# ASM oracle
icon_driver foo.icn -o /tmp/foo.asm -run   # produces output via nasm+ld

# JVM candidate
icon_driver -jvm foo.icn -o /tmp/foo.j
java -jar src/backend/jvm/jasmin.jar /tmp/foo.j -d /tmp/
java -cp /tmp/ FooClass

diff <(icon_driver foo.icn -o /tmp/foo.asm -run 2>/dev/null) \
     <(java -cp /tmp/ FooClass 2>/dev/null)
```

Both must produce identical output for each milestone to fire.

---

## Session Bootstrap (every IJ-session)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
make -C snobol4x/src
# Read FRONTEND-ICON-JVM.md §NOW → start at first ❌
```

---

*FRONTEND-ICON-JVM.md = L3. ~3KB sprint content max per active section.*
*Completed milestones → MILESTONE_ARCHIVE.md on session end.*
