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
| **Icon JVM** | `main` IJ-1 — M-IJ-SCAFFOLD/HELLO/R1 ✅; rung02 12/14; t02_fact recursion + t03_locals stack-height open | `ee2810b` IJ-1 | M-IJ-CORPUS-R2 |

### Next session checklist (IJ-1)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
# Read FRONTEND-ICON-JVM.md §NOW
# Read icon_emit.h — IcnEmitter interface, IcnPorts, label conventions
# Read emit_byrd_jvm.c head — Jasmin output helpers to copy verbatim
# Create src/frontend/icon/icon_emit_jvm.c (see §Design below)
# Add -jvm flag to icon_driver.c
# Test: t01_to5.icn → foo.j → jasmin → java → 1\n2\n3\n4\n5
# Fire M-IJ-SCAFFOLD + M-IJ-HELLO
```

---

## Why JVM — Design Rationale

The x64 ASM backend emits Byrd-box four-port code as NASM labels + `jmp`.
The JVM backend is structurally identical — labels become Jasmin `label:` targets,
`jmp` becomes Jasmin `goto`. The four-port wiring (α/β/γ/ω) is unchanged.

### Oracles

| Oracle | Role |
|--------|------|
| `icon_emit.c` (x64 ASM, 49KB) | Byrd-box wiring per ICN node — authoritative logic |
| `emit_byrd_jvm.c` (187KB) | Jasmin output format, helpers, class skeleton — copy directly |
| `jcon-master/` (uploaded zip) | JCON's `gen_bc.icn` — Icon→JVM IR blueprint |

Key principle: `icon_emit_jvm.c` = `icon_emit.c` logic + `emit_byrd_jvm.c` output format.
Wherever the ASM emitter emits `E(em, "    jmp  %s\n", lbl)`,
the JVM emitter emits `J("    goto %s\n", lbl)`.

### Differences from SNOBOL4 JVM backend

| Aspect | SNOBOL4 JVM | Icon JVM |
|--------|-------------|----------|
| Value type | `java/lang/String` | `java/lang/Long` (ints) + `java/lang/String` (strings) |
| Control flow | `:S/:F` goto via `ifnull` | four-port α/β/γ/ω labels + `goto` |
| Generators | not applicable | β port resumes suspended generator via local flag |
| Variables | static String fields | JVM locals (one `long` slot per variable) |
| Suspension | not applicable | `icn_suspended` byte flag + resume address static field |

### Similarities (reuse directly from `emit_byrd_jvm.c`)

- Jasmin file skeleton: `.class public`, `.super java/lang/Object`, `.method public static main`
- Output helpers: `J()`, `JI()`, `JL()`, `JC()` — copy verbatim (10 lines each)
- `jvm_safe_name()` identifier sanitizer → rename `ij_safe_name()`
- `jasmin.jar` assembler invocation — identical
- Static helper methods emitted inline in the class

---

## Value Representation on JVM

Icon integers → JVM `long` (primitive, stored in local variable slots).
Icon strings → `java/lang/String`.
Goal-directed success/failure → boolean via a static `icn_failed` byte field
(mirrors ASM backend's `byte [rel icn_failed]`).

```jasmin
.field static icn_failed B          ; 0 = success, 1 = failure
.field static icn_suspended B       ; 0 = not suspended, 1 = suspended
.field static icn_suspend_resume J  ; resume address (method handle trick — see below)
```

For the JVM, "resume address" cannot be a raw pointer. Instead, use a
**suspend state integer**: each `ICN_SUSPEND` site gets a unique integer ID;
the β port does a `tableswitch` on the ID to jump to the right resume label.
This is the standard JVM coroutine encoding (mirrors JCON's approach in `gen_bc.icn`).

---

## Label Conventions (parallel to ASM)

| Concept | ASM label | JVM label |
|---------|-----------|-----------|
| Node α | `icon_N_a` | `icn_N_a` |
| Node β | `icon_N_b` | `icn_N_b` |
| Node γ (success) | `icon_N_g` | `icn_N_g` |
| Node ω (fail) | `icon_N_w` | `icn_N_w` |
| Extra (to.code) | `icon_N_code` | `icn_N_code` |
| Procedure entry | `icn_procname` | `icn_procname` (Jasmin label) |
| Procedure done | `icn_procname_done` | `icn_procname_done` |

---

## Design — `icon_emit_jvm.c`

### File structure

```c
/* icon_emit_jvm.c — IcnNode AST → Jasmin text emitter
 *
 * Consumes the same IcnNode* AST as icon_emit.c.
 * Emits Jasmin assembler (.j) text.
 * Assembled by: java -jar jasmin.jar foo.j -d outdir/
 *
 * Oracles:
 *   icon_emit.c        — Byrd-box wiring logic (copy four-port per node)
 *   emit_byrd_jvm.c    — Jasmin output format and helpers
 */

// Output helpers: J(), JI(), JL(), JC()         — copied from emit_byrd_jvm.c
// Safe name:      ij_safe_name()                 — adapted from jvm_safe_name()

// Sections (parallel to icon_emit.c):
//   ij_emit_file_header()    — .class, .super, .method main, static fields
//   ij_emit_runtime_helpers()— icn_write_int/str(), icn_failed field ops
//   ij_emit_proc()           — procedure: prologue + body + epilogue
//   ij_emit_expr()           — dispatch per IcnKind → four-port wiring
//   ij_emit_to()             — ICN_TO: inline counter (paper §4.4)
//   ij_emit_every()          — ICN_EVERY: pump generator to exhaustion
//   ij_emit_if()             — ICN_IF: indirect goto gate (paper §4.5)
//   ij_emit_call()           — ICN_CALL: user procedure + generator suspend
//   ij_emit_suspend()        — ICN_SUSPEND: yield + tableswitch resume
//   ij_emit_file()           — entry point: emit full .j file
```

### `to` generator — inline counter in Jasmin

```jasmin
; every write(1 to 5);  →  ICN_EVERY( ICN_CALL(write, ICN_TO(1,5)) )
; Node 3 = ICN_TO.  Locals: slot 0 = to_I (long), slot 1 = to_limit (long)
icn_3_a:                        ; α — evaluate bounds, init counter
    ldc2_w 1
    lstore 0                    ; to_I = 1
    ldc2_w 5
    lstore 1                    ; to_limit = 5
icn_3_code:                     ; check and yield
    lload 0
    lload 1
    lcmp
    ifgt icn_3_w                ; to_I > to_limit → fail
    lload 0                     ; push to_I as value
    ; (pass to parent γ — write call, which pops and prints)
    goto icn_3_g
icn_3_b:                        ; β — resume: increment and loop
    lload 0
    ldc2_w 1
    ladd
    lstore 0
    goto icn_3_code
icn_3_w:                        ; ω — exhausted
    getstatic  ThisClass/icn_failed B
    ; already set by convention — just goto parent ω
    goto <parent_omega>
icn_3_g:                        ; γ — success, value in slot 0
    goto <parent_gamma>
```

### Suspend/resume via tableswitch

Each `ICN_SUSPEND` (user-defined generator) gets a unique ID `S`.
The procedure's β entry does:

```jasmin
proc_beta:
    getstatic ThisClass/icn_suspend_id I   ; which suspend point?
    tableswitch 1 ... N
        icn_S1_resume
        icn_S2_resume
        ...
    default: proc_omega
```

On `suspend E`: push value, store suspend ID, `goto proc_gamma`.
On β resume: tableswitch dispatches to correct `icn_SN_resume` label.

This encodes the ASM backend's `[rel icn_suspend_resume]` indirect jmp
in pure JVM bytecode without raw pointers.

---

## Milestone Table

| ID | Trigger | Depends on | Status |
|----|---------|-----------|--------|
| **M-IJ-SCAFFOLD** | `icon_emit_jvm.c` exists; `-jvm null.icn → null.j` assembles and exits 0; `-jvm` flag in driver | — | ❌ |
| **M-IJ-HELLO** | `every write(1 to 5);` → JVM output `1\n2\n3\n4\n5` (rung01 t01) | M-IJ-SCAFFOLD | ❌ |
| **M-IJ-CORPUS-R1** | All 6 rung01 tests PASS vs ASM oracle | M-IJ-HELLO | ❌ |
| **M-IJ-PROC** | rung02_proc: user procedures with `return`, local variables | M-IJ-CORPUS-R1 | ❌ |
| **M-IJ-CORPUS-R2** | All rung02 tests PASS (arith_gen 5/5 + proc 3/3) | M-IJ-PROC | ❌ |
| **M-IJ-SUSPEND** | `suspend E` user-defined generators via tableswitch resume | M-IJ-CORPUS-R2 | ❌ |
| **M-IJ-CORPUS-R3** | rung03_suspend: all tests PASS | M-IJ-SUSPEND | ❌ |
| **M-IJ-STRING** | `ICN_STR`, `\|\|` concat; string locals as `java/lang/String` slots | M-IJ-CORPUS-R3 | ❌ |
| **M-IJ-SCAN** | `E ? E` string scanning; cursor threading via static fields | M-IJ-STRING | ❌ |
| **M-IJ-CSET** | Cset literals; `upto`→BREAK, `many`→SPAN, membership→ANY | M-IJ-SCAN | ❌ |
| **M-IJ-CORPUS-R4** | Rung 4: string operations and scanning PASS | M-IJ-CSET | ❌ |

---

## Sprint Map

| Sprint | Milestones | Key work |
|--------|-----------|---------|
| **IJ-S1** | M-IJ-SCAFFOLD, M-IJ-HELLO, M-IJ-CORPUS-R1 | Class skeleton, `icn_failed` field, `to` generator, `every`, arithmetic, relational, `if` |
| **IJ-S2** | M-IJ-PROC, M-IJ-CORPUS-R2 | Procedure call/return, local variable slots, `while`/`until` loops |
| **IJ-S3** | M-IJ-SUSPEND, M-IJ-CORPUS-R3 | `suspend` + tableswitch resume encoding |
| **IJ-S4** | M-IJ-STRING, M-IJ-SCAN, M-IJ-CSET, M-IJ-CORPUS-R4 | String type, concat, scanning, csets |

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
