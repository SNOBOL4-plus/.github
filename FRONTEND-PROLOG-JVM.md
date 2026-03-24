# FRONTEND-PROLOG-JVM.md — Prolog → JVM Backend (L3)

Prolog frontend targeting JVM bytecode via Jasmin.
Reuses the existing Prolog IR pipeline (lex → parse → lower) unchanged.
New layer: `prolog_emit_jvm.c` — consumes `E_CHOICE/E_CLAUSE/E_UNIFY/E_CUT/E_TRAIL_*`
and emits Jasmin `.j` files, assembled by `jasmin.jar`.

**Session trigger phrase:** `"I'm working on Prolog JVM"`
**Session prefix:** `PJ` (e.g. PJ-5, PJ-6, ...)`
**Driver flag:** `snobol4x -pl -jvm foo.pl → foo.j → java -jar jasmin.jar foo.j`
**Oracle:** `snobol4x -pl -asm foo.pl` (ASM emitter, rungs 1–9 known good)
**Design reference:** BACKEND-JVM-PROLOG.md (term encoding, runtime helpers, Jasmin patterns)

*Session state → this file §NOW. Backend reference → BACKEND-JVM-PROLOG.md.*

---

## §NOW — Session State

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **Prolog JVM** | `main` PJ-5 — two fixes applied, rung05 still outputs `a` only | `8f60b6f` PJ-5 | M-PJ-BACKTRACK |

### CRITICAL NEXT ACTION (PJ-6)

**Bug:** `member/2` β-retry outputs `a` only; `b` and `c` not printed.

**What was done in PJ-5:**
- Fix 1: `pj_emit_body` suffix_fail: `goto lbl_omega` → `goto call_try` (β-retry loop now wired)
- Fix 3: per-call trail mark saved at `call3_try`; `call3_sfail` now unwinds trail before looping
- Result: `call3_sfail → pj_trail_unwind → goto call3_try` is structurally correct
- But output is still `a` only — trail unwind is not restoring `X` properly

**Suspected remaining bug:** `pj_trail_unwind` may not be correctly restoring the var cell.
The unwind loop does `checkcast [Ljava/lang/Object;` then sets `[0]="var"`, `[1]=null`.
But the trail stores the **cell itself** via `pj_trail_push` — need to verify the cell stored
on trail is the same object as `var_locals[0]` (X, local 4 in main's clause).

**Debug steps for PJ-6:**
```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x && make -C src

# Step 1: inspect generated Jasmin to verify trail_unwind is called at sfail
BASE=backtrack; PRO=test/frontend/prolog/corpus/rung05_backtrack/${BASE}.pro
./sno2c -pl -jvm $PRO -o /tmp/$BASE.j
grep -A8 "call3_sfail" /tmp/$BASE.j

# Step 2: add stderr debug to pj_trail_unwind in the generated .j manually:
#   before the unwind loop, emit: getstatic System/err; ldc "UNWIND"; invokevirtual println
#   confirm it fires on 2nd iteration

# Step 3: check pj_unify — when X (var cell) is bound to atom 'a', is the cell
#   actually pushed onto pj_trail? Verify pj_trail_push is reached in pj_unify.

# Step 4: if trail is empty on retry, the bind was never trailed.
#   Check pj_unify: the "bind a → b" path calls pj_trail_push([Ljava/lang/Object;)V
#   but the stack manipulation may be wrong — verify with javap -c on Backtrack.class

# Expected after fix: a\nb\nc
java -jar src/backend/jvm/jasmin.jar /tmp/$BASE.j -d /tmp/
timeout 5 java -cp /tmp Backtrack
```

**If trail is confirmed empty on retry:**
The issue is in `pj_unify` — the `aload_0 checkcast [Ljava/lang/Object;` / trail_push sequence.
The cell pushed must be the var cell (`a` = `{tag,ref}` Object[]), not something else.
Compare with BACKEND-JVM-PROLOG.md §Trail + Unification Runtime for the correct Java model.

---

## Milestone Table

| ID | Trigger | Status |
|----|---------|--------|
| **M-PJ-SCAFFOLD** | `-pl -jvm null.pl → null.j` assembles + exits 0 | ✅ |
| **M-PJ-HELLO** | `write('hello'), nl.` → JVM output `hello` | ✅ |
| **M-PJ-FACTS** | Rung 2: deterministic fact lookup | ✅ |
| **M-PJ-UNIFY** | Rung 3: head unification, compound terms | ✅ |
| **M-PJ-ARITH** | Rung 4: `is/2` arithmetic | ✅ |
| **M-PJ-BACKTRACK** | Rung 5: `member/2` — β port, all solutions | ❌ |
| **M-PJ-LISTS** | Rung 6: `append/3`, `length/2`, `reverse/2` | ❌ |
| **M-PJ-CUT** | Rung 7: `differ/N`, closed-world `!, fail` | ❌ |
| **M-PJ-RECUR** | Rung 8: `fibonacci/2`, `factorial/2` | ❌ |
| **M-PJ-BUILTINS** | Rung 9: `functor/3`, `arg/3`, `=../2`, type tests | ❌ |
| **M-PJ-CORPUS-R10** | Rung 10: Lon's puzzle corpus PASS | ❌ |

---

## Session Bootstrap (every PJ-session)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
make -C snobol4x/src
# Read §NOW above. Start at CRITICAL NEXT ACTION.
```

---

*FRONTEND-PROLOG-JVM.md = L3. ~3KB sprint content max. Archive ✅ milestones to MILESTONE_ARCHIVE.md on session end.*
