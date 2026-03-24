# FRONTEND-PROLOG-JVM.md — Prolog → JVM Backend (L3)

Prolog frontend targeting JVM bytecode via Jasmin.
Reuses the existing Prolog IR pipeline (lex → parse → lower) unchanged.
New layer: `prolog_emit_jvm.c` — consumes `E_CHOICE/E_CLAUSE/E_UNIFY/E_CUT/E_TRAIL_*`
and emits Jasmin `.j` files, assembled by `jasmin.jar`.

**Session trigger phrase:** `"I'm working on Prolog JVM"`
**Session prefix:** `PJ` (e.g. PJ-1, PJ-2, PJ-3)
**Driver flag:** `snobol4x -pl -jvm foo.pl → foo.j → java -jar jasmin.jar foo.j`
**Oracle:** `snobol4x -pl -c foo.pl → foo.c → gcc → ./a.out` (the C emitter, rungs 1–9 known good)

*Session state → this file §NOW. Backend reference → BACKEND-JVM.md.*

---

## §NOW — Session State

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **Prolog JVM** | `main` PJ-3 — M-PJ-FACTS ✅ M-PJ-UNIFY ✅ M-PJ-ARITH ✅; rungs 5-9 failing | `cb87932` PJ-3 | M-PJ-BACKTRACK |

### Session PJ-3 summary (2026-03-24)

- Fixed `pj_trail_unwind`: restores both `[0]="var"` AND `[1]=null` → M-PJ-FACTS ✅ (rung02: brown/jones/smith)
- Implemented compound term unification in `pj_unify`: functor+arity check, recursive arg loop → M-PJ-UNIFY ✅ (rung03: b a)
- Implemented `E_FNC` in `pj_emit_term`: flat `Object[]` `[0]="compound",[1]=functor,[2..]=args`
- Flat n-ary `,/2` and `;/2` in `prolog_lower.c`: right-spine flattened at IR level; emitter uses `goal->children` directly
- Fixed `pj_emit_goal` `;/2` + `->/2`: proper if-then-else with `cond_ok`/`cond_fail` labels; n-ary else chain
- Fixed `ldc2_w` `L` suffix (Jasmin rejects it) → M-PJ-ARITH ✅ (rung04: 6/true/false)
- HEAD: `cb87932`

### Known bugs for PJ-4 (four failures, rung05–09)

**rung05 backtrack** — `member(X,[a,b,c])` prints `_` instead of `a b c`. Variable `X` is unbound in `write`. Root cause: the Proebsting retry loop passes `cs` state but the *bound value* of `X` in the callee's environment is not visible to the caller. The retry loop allocates a fresh `rv` local for the return value but the caller writes `X` before the callee has had a chance to bind it. Fix: after the `invokestatic` call returns non-null (γ), the callee's bindings are already in the shared trail — `X` should already be bound. Check whether `pj_emit_body` for the user call actually `astore`s the result and whether the *caller's* `X` local and the *callee's* arg slot refer to the same `["var",...]` cell object.

**rung07 cut** — `differ(a,a)` returns `yes` instead of `no`. `E_CUT` is emitted but not sealing beta in the Proebsting retry loop. The `pj_emit_goal` E_CUT case sets `cs = N` (past last clause) for the *current predicate*, but in the retry loop the `cs` local belongs to the *caller's* retry scaffolding for `differ/2`. Need to verify E_CUT actually writes to the retry loop's `cs` local in the surrounding `pj_emit_body` frame.

**rung06 lists** — empty output. `write/1` of a list compound `["compound",".",H,T]` hits the default `_` branch in `pj_write`. Fix: add list-printing to `pj_write`: if tag=`"compound"` and functor=`"."`, print `[H|T]` recursively; if `[]` print `[]`.

**rung08 recursion** — `NumberFormatException: Cannot parse null string` in `p_fib_2`. `pj_emit_arith` E_VART: after `pj_deref`, the result is `["int","6"]`. But `fib/2` head arg slot `N` is the *caller-passed* term. After deref the array `[1]` should be `"6"`. Hypothesis: the head arg local holds a `["var", ref]` cell and `pj_deref` is returning the var cell itself when `ref` is null (i.e. deref doesn't follow into the term properly). Check `pj_deref` — it follows `tag="ref"` chains but stops at `tag="var"`. If the caller passes an atom-int `["int","6"]` directly (not wrapped in a var), deref returns it immediately and `[1]="6"` is correct. But if `N` was unified via a var chain, the deref result might still be a var cell.

**rung09 builtins** — empty output. Likely `functor/3`, `arg/3`, `=../2` not implemented. Check `pj_emit_goal` for these atoms — they fall through to the user-call path and there is no matching predicate class.

### Next session checklist (PJ-4)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x/src && make
# Read FRONTEND-PROLOG-JVM.md §NOW — start here
INC=src/frontend/prolog
# Helper to test one rung:
# BASE=backtrack; PRO=test/frontend/prolog/corpus/rung05_${BASE}/${BASE}.pro
# ./sno2c -pl $PRO -o /tmp/$BASE.c && gcc /tmp/$BASE.c $INC/prolog_atom.c $INC/prolog_unify.c $INC/prolog_builtin.c -I $INC -o /tmp/${BASE}_c && /tmp/${BASE}_c
# ./sno2c -pl -jvm $PRO -o /tmp/$BASE.j && java -jar src/backend/jvm/jasmin.jar /tmp/$BASE.j -d /tmp/ && java -cp /tmp/ Backtrack
# Fix in order: rung08 (arith deref) → rung06 (list write) → rung07 (cut) → rung05 (backtrack vars) → rung09 (builtins)
```

---

## Why JVM — Design Rationale

The ASM backend emits Byrd-box four-port code as NASM labels + `jmp`.
The JVM backend does the same thing — but labels become Jasmin `label:` targets
and `jmp` becomes Jasmin `goto`. The four-port model is identical.

Structural oracle: `emit_byrd_jvm.c` (187KB, snobol4x `src/backend/jvm/`).
That file already handles SNOBOL4 → JVM via the same Byrd-box IR.
`prolog_emit_jvm.c` is its sibling: same output format, different input nodes.

Key differences from SNOBOL4 JVM backend:
- No `sno_var_*` static String fields — Prolog uses per-clause local variables
- Trail + unification replace pattern matching primitives
- Choice points replace pattern ALT/ARBNO nodes
- Terms (TERM_t) replace SNOBOL4 strings as the value type

Key similarities (reuse directly):
- Jasmin file skeleton (class header, main method, static helpers)
- `J()` / `JI()` / `JL()` output helpers — copy verbatim
- `jvm_safe_name()` identifier sanitization
- Arithmetic helpers: `sno_arith`, `Long.parseLong` / `ddiv` patterns
- `:S/:F` routing via `ifnull` / `ifnonnull` — exact same trick

---

## Term Representation on JVM

SNOBOL4 JVM uses `java/lang/String` for all values.
Prolog JVM uses `java/lang/Object[]` (boxed term arrays) for all values.

```
TERM encoding on JVM heap (Object[] of length 2+):
  [0] = tag string: "atom", "int", "float", "var", "compound", "ref"
  [1] = value:
        atom      → String (atom name)
        int       → String (decimal, as in SNOBOL4 JVM)
        float     → String (decimal)
        var       → Object[] (points to binding cell, initially null slot)
        compound  → Object[] { tagStr, functor, arity, arg0, arg1, ... }
        ref       → Object[] (points to bound-to term)
  null = unbound variable
```

This keeps all terms as Java objects — no native library needed.
Unification and trail are emitted as static helper methods on the class,
same pattern as `sno_arith` in the SNOBOL4 JVM backend.

---

## Design — `prolog_emit_jvm.c`

### File structure (mirrors `emit_byrd_jvm.c`)

```c
/* prolog_emit_jvm.c — Prolog IR → Jasmin text emitter */

// Output helpers: J(), JI(), JL(), JC(), JSep()  — copy from emit_byrd_jvm.c
// Safe name:      pj_safe_name()                 — like jvm_safe_name()

// Sections:
//   pj_emit_class_header()    — .class public, .super, .method main
//   pj_emit_runtime_helpers() — unify(), trail_push/unwind(), term constructors
//   pj_emit_atom_table()      — static String[] for interned atoms
//   pj_emit_choice()          — E_CHOICE → α/β/ω label chain
//   pj_emit_clause()          — E_CLAUSE → head unify + body goals
//   pj_emit_goal()            — E_FNC / E_UNIFY / E_CUT / arithmetic
//   pj_emit_term()            — E_QLIT / E_ILIT / E_VART / E_FNC (term context)
//   pj_emit_main_init()       — initialization directive call
//   prolog_emit_jvm(prog, out) — entry point
```

### Label conventions (parallel to ASM emitter)

| Concept | ASM label | JVM label |
|---------|-----------|-----------|
| Choice α | `P_FOO_1_alpha` | `p_foo_1_alpha` |
| Choice β (clause N) | `P_FOO_2_alpha` | `p_foo_2_alpha` |
| Choice ω | `P_FOO_omega` | `p_foo_omega` |
| Goal succeed | `goal_N_gamma` | `goal_N_gamma` |
| Goal fail | `goal_N_omega` | `goal_N_omega` |
| Trail mark | `trail_mark_N` | `trail_mark_N` |

### Byrd box wiring — clause selection (Jasmin)

```jasmin
; foo/1 with two clauses
; local 0 = arg0 (Object[])
; local 1 = trail Object[] (growable array reference)
; local 2 = trail mark (int)

p_foo_alpha:
    ; trail mark
    invokestatic  ThisClass/trail_mark()I
    istore 2
    ; try clause 1: unify arg0 with head
    aload 0
    ldc "atom_a"
    invokestatic  ThisClass/unify(Ljava/lang/Object;Ljava/lang/String;)Z
    ifeq p_foo_beta   ; unify failed → try next clause
    ; body of clause 1 ...
    goto p_foo_gamma

p_foo_beta:
    ; unwind trail to mark
    iload 2
    invokestatic  ThisClass/trail_unwind(I)V
    ; trail mark again for clause 2
    invokestatic  ThisClass/trail_mark()I
    istore 2
    ; try clause 2: unify arg0 with head
    aload 0
    ldc "atom_b"
    invokestatic  ThisClass/unify(Ljava/lang/Object;Ljava/lang/String;)Z
    ifeq p_foo_omega
    ; body of clause 2 ...
    goto p_foo_gamma

p_foo_omega:
    iload 2
    invokestatic  ThisClass/trail_unwind(I)V
    goto caller_omega

p_foo_gamma:
    goto caller_gamma
```

### Runtime helpers (emitted inline in class)

```
trail_mark()I          — returns trail.size() as int
trail_unwind(I)V       — restores trail to saved mark, unbinds vars
unify(Object,Object)Z  — WAM-style unification, returns boolean
term_atom(String)      — allocate atom term
term_int(long)         — allocate integer term
term_var()             — allocate unbound variable cell
term_compound(String,int,Object[]) — allocate compound
deref(Object)Object    — dereference chain
write_term(Object)V    — write/1 builtin, mirrors sno_write in JVM backend
```

All implemented as `static` methods in the emitted class — same pattern as
`sno_arith`, `sno_write`, `sno_output_assign` in `emit_byrd_jvm.c`.

---

## Milestone Table

| ID | Trigger | Depends on | Status |
|----|---------|-----------|--------|
| **M-PJ-SCAFFOLD** | `prolog_emit_jvm.c` exists; `-pl -jvm null.pl → null.j` assembles and exits 0; driver wired | — | ✅ |
| **M-PJ-HELLO** | `hello.pl` → `write('hello'), nl.` → JVM output `hello` | M-PJ-SCAFFOLD | ✅ |
| **M-PJ-FACTS** | Rung 2: deterministic fact lookup, `write(answer)` | M-PJ-HELLO | ❌ |
| **M-PJ-UNIFY** | Rung 3: head unification, compound terms | M-PJ-FACTS | ❌ |
| **M-PJ-ARITH** | Rung 4: `is/2` arithmetic — reuse JVM `sno_arith` helpers | M-PJ-UNIFY | ❌ |
| **M-PJ-BACKTRACK** | Rung 5: `member/2` — first backtracking via β port | M-PJ-ARITH | ❌ |
| **M-PJ-LISTS** | Rung 6: `append/3`, `length/2`, `reverse/2` | M-PJ-BACKTRACK | ❌ |
| **M-PJ-CUT** | Rung 7: `differ/N`, closed-world `!, fail` pattern | M-PJ-LISTS | ❌ |
| **M-PJ-RECUR** | Rung 8: `fibonacci/2`, `factorial/2` | M-PJ-CUT | ❌ |
| **M-PJ-BUILTINS** | Rung 9: `functor/3`, `arg/3`, `=../2`, type tests | M-PJ-RECUR | ❌ |
| **M-PJ-CORPUS-R10** | Rung 10: Lon's puzzle corpus — all solved puzzles PASS | M-PJ-BUILTINS | ❌ |

---

## Sprint Map

| Sprint | Milestones | Key work |
|--------|-----------|---------|
| **PJ-S1** | M-PJ-SCAFFOLD, M-PJ-HELLO | Create file, wire driver, emit class header + `write` helper |
| **PJ-S2** | M-PJ-FACTS, M-PJ-UNIFY | Atom table, `unify()` helper, deterministic clause dispatch |
| **PJ-S3** | M-PJ-ARITH | `is/2` → `sno_arith` pattern (reuse from JVM backend) |
| **PJ-S4** | M-PJ-BACKTRACK | Trail, `trail_mark/unwind`, β port wiring, `member/2` |
| **PJ-S5** | M-PJ-LISTS | List term encoding, recursive clause chains |
| **PJ-S6** | M-PJ-CUT, M-PJ-RECUR | `E_CUT` seals β → ω; recursive per-frame locals |
| **PJ-S7** | M-PJ-BUILTINS | `functor/3`, `arg/3`, `=../2`, type tests |
| **PJ-S8** | M-PJ-CORPUS-R10 | Puzzle corpus; may expose constraint/arithmetic gaps |

---

## Key Files

| File | Role |
|------|------|
| `src/frontend/prolog/prolog_emit_jvm.c` | **TO CREATE** — this sprint's deliverable |
| `src/frontend/prolog/prolog_lower.c` | IR producer — consumed unchanged |
| `src/frontend/prolog/prolog_emit.c` | C emitter — structural oracle for Byrd-box logic |
| `src/backend/jvm/emit_byrd_jvm.c` | JVM emitter oracle — Jasmin output format |
| `src/backend/jvm/jasmin.jar` | Assembler — `java -jar jasmin.jar foo.j -d outdir/` |
| `driver/main.c` | Add `-pl -jvm` → `prolog_emit_jvm()` branch |
| `test/frontend/prolog/corpus/rung01_hello/` | First test — same `.pl` files, new `.j` oracle output |

---

## Oracle Comparison Strategy

For each rung, the C emitter is the correctness oracle:
```bash
snobol4x -pl -c   foo.pl -o /tmp/foo.c && gcc /tmp/foo.c -o /tmp/foo_c && /tmp/foo_c
snobol4x -pl -jvm foo.pl -o /tmp/foo.j && java -jar jasmin.jar /tmp/foo.j -d /tmp/ && java -cp /tmp/ FooClass
diff <(/tmp/foo_c) <(java -cp /tmp/ FooClass)
```
Both must produce identical output for the milestone to fire.

---

## Session Bootstrap (every PJ-session)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
make -C snobol4x/src
# Read FRONTEND-PROLOG-JVM.md §NOW for current milestone
# Start at first ❌ in milestone table
```

---

*FRONTEND-PROLOG-JVM.md = L3. ~3KB sprint content max per section. Archive completed milestones to MILESTONE_ARCHIVE.md on session end.*
