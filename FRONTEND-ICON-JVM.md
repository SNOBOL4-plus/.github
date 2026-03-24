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
| **Icon JVM** | `main` IJ-5 — Bug3 write(long) stack + Bug2 tableswitch label fixed; rung01 6/6 + rung02 14/14 PASS; rung03 VerifyError slot 2/3 open | `e590c4f` IJ-5 | M-IJ-CORPUS-R3 |

### Next session checklist (IJ-6)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x/src/frontend/icon
gcc -Wall -Wextra -g -O0 -I. icon_driver.c icon_lex.c icon_parse.c icon_ast.c \
    icon_emit.c icon_emit_jvm.c icon_runtime.c -o /tmp/icon_driver
# Read FRONTEND-ICON-JVM.md §NOW
# HEAD = e590c4f; rung01 6/6 + rung02 14/14 already pass
# Fix rung03 VerifyError: slot 2/3 type merge — see IJ-5 findings below
# Run from repo root: cd snobol4x && /tmp/icon_driver -jvm ... (oracle also needs repo root)
# Fire M-IJ-CORPUS-R2 (already earned: 14/14) then M-IJ-CORPUS-R3 when rung03 passes
```

### IJ-5 findings

**Bug 3 — FIXED (e590c4f):** `write(long)` after `invokevirtual println(J)V` now
reloads the scratch slot (`lload scratch`) so the value stays on stack for the γ port
caller. `write()` returns its argument; `gbfwd: pop2` needs it there.

**Bug 2 — FIXED (e590c4f):** `ij_suspend_ids[k] = id` (node id, not `susp_id`).
The resume label is `icn_{node_id}_resume`; `susp_id` is only the tableswitch ordinal.

**Rung03 Bug — OPEN for IJ-6:** VerifyError `Register pair 2/3 contains wrong type`
in `icn_upto()V`. Root cause: the beta dispatch block (`getstatic icn_suspend_id; ifne
icn_upto_beta`) creates a control-flow join at `icn_upto_beta` where the JVM verifier
sees slot 2/3 as uninitialized on the fall-through path but long on paths that went
through the body. The slot type merge fails.

Fix strategy: initialize all param/local long slots to 0L **before** the suspend_id
dispatch check, so every path into `icn_upto_beta` has slot 2/3 typed as long:
```
lconst_0; lstore 0   ; lconst_0; lstore 2   ; ...   (all used long slots)
getstatic icn_suspend_id I
ifne icn_upto_beta
icn_upto_fresh:
  getstatic icn_arg_0 J; lstore 0   ; (real param load overwrites)
  goto first_stmt
icn_upto_beta:
  tableswitch ...
```
In `ij_emit_proc`, before the suspend_id dispatch, emit `lconst_0; lstore N` for each
long slot 0..2*(nlocals-1). This ensures verifier type consistency.

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
