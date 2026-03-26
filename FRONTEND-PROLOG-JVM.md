# FRONTEND-PROLOG-JVM.md — Prolog → JVM Backend (L4)

Prolog frontend targeting JVM bytecode via Jasmin.
Reuses the existing Prolog IR pipeline (lex → parse → lower) unchanged.
New layer: `prolog_emit_jvm.c` — consumes `E_CHOICE/E_CLAUSE/E_UNIFY/E_CUT/E_TRAIL_*`
and emits Jasmin `.j` files, assembled by `jasmin.jar`.

**Session trigger phrase:** `"I'm working on Prolog JVM"`
**Session prefix:** `PJ` (e.g. PJ-7, PJ-8, ...)
**Driver flag:** `snobol4x -pl -jvm foo.pl → foo.j → java -jar jasmin.jar foo.j`
**Oracle:** `snobol4x -pl -asm foo.pl` (ASM emitter, rungs 1–9 known good)
**Design reference:** BACKEND-JVM-PROLOG.md (term encoding, runtime helpers, Jasmin patterns)

*Session state → this file §NOW. Backend reference → BACKEND-JVM-PROLOG.md.*

---

## §NOW — Session State

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **Prolog JVM** | `main` PJ-53 — M-PJ-ASSERTZ ✅ 5/5 rung13 | `8929f4e` PJ-53 | M-PJ-RETRACT |

### CRITICAL NEXT ACTION (PJ-54)

**Baseline: 5/5 rung11 ✅. 5/5 rung12 ✅. 5/5 rung13 ✅. snobol4x HEAD `8929f4e`.**

**Next milestone: M-PJ-RETRACT — implement `retract/1`, get 5/5 rung14.**

**Plan:**
- Add `pj_db_retract(String key, int idx)` helper: removes entry at index `idx` from the ArrayList for `key` in `pj_db`. Returns the removed `Object[]` term, or null if out-of-range.
- Add `retract/1` dispatch in `pj_emit_goal`: call `pj_db_retract_key` to get key, call `pj_db_retract`, unify head with returned term.
- Backtracking: `retract/1` is choice-point-aware — on retry, increment idx and try next entry.
- Create rung14 corpus: 5 `.pro` + `.expected` covering: basic retract, retract with unification, retract-all via backtracking, retract from mixed DB, retract nonexistent (fail).
- Verify 5/5 rung11–rung13 no regressions.

**Bootstrap PJ-54:**
```bash
git clone https://TOKEN@github.com/snobol4ever/snobol4x
git clone https://TOKEN@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev swi-prolog
make -C snobol4x/src
# Read §NOW above. Implement retract/1.
# bash test/frontend/prolog/run_prolog_jvm_rung.sh test/frontend/prolog/corpus/rung14_retract
# Confirm rung11–rung13 no regressions
# Commit snobol4x, update §NOW + PLAN.md + SESSIONS_ARCHIVE.md, push both repos
```
## Milestone Table

| ID | Trigger | Status |
|----|---------|--------|
| **M-PJ-SCAFFOLD** | `-pl -jvm null.pl → null.j` assembles + exits 0 | ✅ |
| **M-PJ-HELLO** | `write('hello'), nl.` → JVM output `hello` | ✅ |
| **M-PJ-FACTS** | Rung 2: deterministic fact lookup | ✅ |
| **M-PJ-UNIFY** | Rung 3: head unification, compound terms | ✅ |
| **M-PJ-ARITH** | Rung 4: `is/2` arithmetic | ✅ |
| **M-PJ-BACKTRACK** | Rung 5: `member/2` — β port, all solutions | ✅ |
| **M-PJ-LISTS** | Rung 6: `append/3`, `length/2`, `reverse/2` | ✅ |
| **M-PJ-CUT** | Rung 7: `differ/N`, closed-world `!, fail` | ✅ |
| **M-PJ-RECUR** | Rung 8: `fibonacci/2`, `factorial/2` | ✅ |
| **M-PJ-BUILTINS** | Rung 9: `functor/3`, `arg/3`, `=../2`, type tests | ✅ |
| **M-PJ-CORPUS-R10** | Rung 10: Lon's puzzle corpus PASS | ✅ |
| **M-PJ-NEQ** | `\=/2` emit missing in `pj_emit_goal` | ✅ |
| **M-PJ-STACK-LIMIT** | Dynamic `.limit stack` via term depth walker | ✅ |
| **M-PJ-NAF-TRAIL** | `\+` trail: save mark before inner goal, unwind both paths | ✅ |
| **M-PJ-BODYFAIL-TRAIL** | Body-fail trail unwind: `bodyfail_N` trampoline per clause | ✅ |
| **M-PJ-BETWEEN** | `between/3` — synthetic p_between_3 method | ✅ |
| **M-PJ-DISJ-ARITH** | Plain `;` retry loop — tableswitch dispatch; puzzle_12 PASS | ✅ |
| **M-PJ-CUT-UCALL** | `!` + ucall body sentinel propagation | ✅ |
| **M-PJ-NAF-INNER-LOCALS** | NAF helper method — fix frame aliasing; puzzle_18 PASS | ✅ |
| **M-PJ-DISPLAY-BT** | puzzle_03 over-generation workaround; 20/20 | ✅ |
| **M-PJ-PZ-ALL-JVM** | All 20 puzzle solutions pass JVM | ✅ |
| **M-PJ-FINDALL** | `findall/3` — collect all solutions into list | ✅ |
| **M-PJ-ATOM-BUILTINS** | atom_chars/length/concat/codes/char_code etc. | ✅ |
| **M-PJ-ASSERTZ** | `assertz/1`, `asserta/1` — dynamic DB (Scripten dep) | ❌ **NEXT** |

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

**Sprint order:** ASSERTZ → RETRACT → SORT → SUCC-PLUS → FORMAT → STRING-OPS → AGGREGATE → COPY-TERM → EXCEPTIONS → NUMBER-OPS → DCG.



*FRONTEND-PROLOG-JVM.md = L4. §NOW = ONE bootstrap block only — current session's next action. Prior session findings → SESSIONS_ARCHIVE.md only. Completed milestones → MILESTONE_ARCHIVE.md. Size target: ≤8KB total.*
