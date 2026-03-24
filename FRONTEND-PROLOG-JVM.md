# FRONTEND-PROLOG-JVM.md — Prolog → JVM Backend (L3)

Prolog frontend targeting JVM bytecode via Jasmin.
Reuses the existing Prolog IR pipeline (lex → parse → lower) unchanged.
New layer: `prolog_emit_jvm.c` — consumes `E_CHOICE/E_CLAUSE/E_UNIFY/E_CUT/E_TRAIL_*`
and emits Jasmin `.j` files, assembled by `jasmin.jar`.

**Session trigger phrase:** `"I'm working on Prolog JVM"`
**Session prefix:** `PJ` (e.g. PJ-1, PJ-2, PJ-3)
**Driver flag:** `snobol4x -pl -jvm foo.pl → foo.j → java -jar jasmin.jar foo.j`
**Oracle:** `snobol4x -pl -asm foo.pl` (ASM emitter, rungs 1–9 known good)
**Design reference:** BACKEND-JVM-PROLOG.md (term encoding, runtime helpers, Jasmin patterns)

*Session state → this file §NOW. Backend reference → BACKEND-JVM-PROLOG.md.*

---

## §NOW — Session State

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **Prolog JVM** | `main` PJ-5 — M-PJ-BACKTRACK β-retry bug diagnosed | `418461a` PJ-5 | M-PJ-BACKTRACK |

### Session PJ-5 diagnosis (2026-03-24)

Two bugs confirmed by inspecting generated `/tmp/backtrack.j`:

**Bug 1 — suffix_fail misroutes to `;` else branch instead of retrying ucall:**
In `pj_emit_body` user-call block, `suffix_fail` does `goto lbl_omega`. But when the
ucall is inside a `;/2` left branch, `lbl_omega` resolves to `disj2_alt1` (else branch)
not back to `call_try`. Output: only `a` printed; `b`/`c` skipped.
Fix: change `JI("goto", lbl_omega)` → `JI("goto", call_try)` in suffix_fail block.

**Bug 2 — `main()` does not loop:**
`pj_emit_main()` calls `p_main_0(0)` once and discards result. No β-retry loop.
Fix: emit retry loop — extract cs from `result[0]`, call again while non-null.

### CRITICAL NEXT ACTION (PJ-6)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x && make -C src
# Fix 1: pj_emit_body suffix_fail: JI("goto", lbl_omega) → JI("goto", call_try)
# Fix 2: pj_emit_main(): add β-retry loop (load cs from result[0], loop while non-null)
BASE=backtrack; PRO=test/frontend/prolog/corpus/rung05_${BASE}/${BASE}.pro
./sno2c -pl -jvm $PRO -o /tmp/$BASE.j && java -jar src/backend/jvm/jasmin.jar /tmp/$BASE.j -d /tmp/
java -cp /tmp Backtrack   # expected: a\nb\nc
```

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
# Read §NOW for current milestone. Start at first ❌.
```

---

*FRONTEND-PROLOG-JVM.md = L3. ~3KB sprint content max. Archive ✅ milestones to MILESTONE_ARCHIVE.md on session end.*
