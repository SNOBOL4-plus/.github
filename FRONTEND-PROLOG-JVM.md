# FRONTEND-PROLOG-JVM.md — Prolog → JVM Backend (L3)

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
| **Prolog JVM** | `main` PJ-49 — 5/5 rung11 ✅; 4/5 rung12; pj_is_user_call whitelist fixed; atom_codes reverse ClassCastException WIP | `7e31f3a` PJ-49 | M-PJ-ATOM-BUILTINS |

### CRITICAL NEXT ACTION (PJ-50)

**Baseline: 5/5 rung11 PASS. 20/20 puzzle corpus PASS. snobol4x HEAD `7e31f3a`.**

**Next milestone: M-PJ-ATOM-BUILTINS — fix nil-check bug, then green rung12**

**THE BUG:** `pj_code_list_to_string` reverse path — `atom_codes(A, [104,101,...])` crashes with ClassCastException: String cannot be cast to Long.

Root cause: nil-check in `colts_loop` uses `iconst_1 aaload` (head slot) but nil term `{"[]"}` only has index 0. Check `pj_char_list_to_string` (which works) — uses `iconst_0 aaload` (the tag). Fix `pj_code_list_to_string` nil-check to use `iconst_0`. Also verify head is at [1] vs [2] to match `pj_char_list_to_string`.

Fix: `make -C src`, then `5/5 rung12` → **M-PJ-ATOM-BUILTINS ✅**. Then 20/20 puzzle sweep.

**Bootstrap PJ-50:**
```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y --fix-missing default-jdk nasm libgc-dev swi-prolog
make -C snobol4x/src && cd snobol4x
# Confirm 5/5 rung11 + 20/20 puzzle baseline
for f in test/frontend/prolog/corpus/rung11_findall/*.pro; do
  base=$(basename $f .pro); ./sno2c -pl -jvm $f -o /tmp/${base}.j 2>/dev/null
  java -jar src/backend/jvm/jasmin.jar /tmp/${base}.j -d /tmp 2>/dev/null
  cls=$(grep "^\.class" /tmp/${base}.j | awk '{print $3}')
  got=$(timeout 10 java -cp /tmp $cls 2>&1 | grep -v "Picked up")
  want=$(cat ${f%.pro}.expected)
  [ "$got" = "$want" ] && echo "$base: PASS" || echo "$base: FAIL"
done
# Fix pj_code_list_to_string nil-check: iconst_0 aaload, not iconst_1
# Build, run rung12 sweep, expect 5/5
# Then run 20/20 puzzle sweep to confirm no regressions
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
| **M-PJ-ATOM-BUILTINS** | atom_chars/length/concat/codes/char_code etc. | ❌ **NEXT** |
| **M-PJ-ASSERTZ** | `assertz/1`, `asserta/1` — dynamic DB (Scripten dep) | ❌ |

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

*FRONTEND-PROLOG-JVM.md = L4. §NOW = ONE bootstrap block only — current session's next action. Prior session findings → SESSIONS_ARCHIVE.md only. Completed milestones → MILESTONE_ARCHIVE.md. Size target: ≤8KB total.*

---

## Tiny-Prolog Enhancement Roadmap — ISO Gap Closure

**Oracle:** `swipl -q -g halt -t main file.pro`
**Tests:** `test/frontend/prolog/corpus/rung11_*/` onward
**Impl:** `prolog_emit_jvm.c` (emit) + `prolog_builtin.c` (runtime helpers)

### Tier 1 — High Impact

| Milestone | Feature | Rung | Status |
|-----------|---------|------|--------|
| M-PJ-FINDALL | `findall/3` | rung11 | ✅ |
| M-PJ-ATOM-BUILTINS | atom_chars/codes/length/concat/char_code, upcase/downcase | rung12 | ❌ **NEXT** |
| M-PJ-ASSERTZ | `assertz/1`, `asserta/1` dynamic DB — **Scripten Demo dep** | rung13 | ❌ |
| M-PJ-RETRACT | `retract/1`, `retractall/1` (depends: ASSERTZ) | rung14 | ❌ |
| M-PJ-SORT | `sort/2`, `msort/2`, `keysort/2` | rung14b | ❌ |

### Tier 2 — Medium Impact

| Milestone | Feature | Rung | Status |
|-----------|---------|------|--------|
| M-PJ-SUCC-PLUS | `succ/2`, `plus/3` reversible arithmetic | rung15 | ❌ |
| M-PJ-FORMAT | `format/1`, `format/2`, `format(atom(A),...)` | rung16 | ❌ |
| M-PJ-STRING-OPS | `split_string/4`, `string_concat/3`, etc. | rung17 | ❌ |
| M-PJ-AGGREGATE | `bagof/3`, `setof/3` (depends: FINDALL+SORT) | rung18 | ❌ |
| M-PJ-COPY-TERM | `copy_term/2` deep copy with fresh vars | rung19 | ❌ |
| M-PJ-EXCEPTIONS | `catch/3`, `throw/1`, ISO error terms | rung20 | ❌ |
| M-PJ-NUMBER-OPS | Extended `is/2`: trig, round, abs, max, min | rung21 | ❌ |

### Tier 3 — Future

| Milestone | Feature | Status |
|-----------|---------|--------|
| M-PJ-DCG | DCG `-->`, `phrase/2` (depends: COPY-TERM) | 💭 |

**Sprint order:** ATOM-BUILTINS → **ASSERTZ** → SORT → SUCC-PLUS → RETRACT → FORMAT → STRING-OPS → AGGREGATE → COPY-TERM → EXCEPTIONS → NUMBER-OPS → DCG.
