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
| **Prolog JVM** | `main` PJ-76 — 8 plunit bugs fixed; SWI baseline partial | `d6c63ad` PJ-76 | M-PJ-SWI-BASELINE |

### CRITICAL NEXT ACTION (PJ-77)

**PJ-76 findings: 8 bugs fixed in plunit linker/shim. Corpus 107/107. SWI files now run.**

**What was done PJ-76 (commits PJ-76a/b/c):**
1. `pj_linker_emit_main_assertz` moved before directive loop (was after → 0/0/0 always)
2. `pj_linker_emit_bridge` now dispatches `p_test_2(name,opts,cs)` for `test/2` (was always `p_test_1`)
3. Bare-goal opts (`X==3`) wrapped as `true(Opts)` compound at assertz time
4. `run_suite/1` — added `!` after format to prevent double header print on retry
5. `E_VART` as goal → now calls `pj_call_goal` with `-1`-as-fail convention (was `goto lbl_γ`)
6. `p_main_0` call guarded by `has_main0` scan (plunit-only files crashed)
7. Auto `run_tests` emitted in `main()` when plunit file has no `:- run_tests` directive
8. `run_tests/1` accepts list of suites; `run_suites_list/1` helper added to shim
- **Corpus:** 107/107 pass, 0 regressions throughout
- SWI test files provided as `swipl-devel-master.zip` — extract to `/tmp/swipl-devel-master/`

**SWI baseline so far (tests/core/):**
- `test_list.pl` — 0p/1f: `true(Expr)` opts variable-sharing gap (known, see below)
- `test_exception.pl` — 0p/5f/1s: throw/catch semantics gaps
- `test_arith/unify/dcg/misc.pl` — compile errors (parse gaps)

**PJ-77 task: M-PJ-SWI-BASELINE continued**

**Task 1 — CRITICAL: `true(Expr)` opts variable-sharing fix**
- Problem: `test(name, X==y) :- Body` — bridge emits `pj_term_var()` for `X` in opts, disconnected from `X` in `Body`
- Fix: store body `EXPR_t*` in `PjTestInfo`; in bridge emit body+check in same JVM scope
- In `pj_linker_scan`: add `body_expr` field to `PjTestInfo`, set to `cl->body` (the clause body nodes)
- In `pj_linker_emit_bridge` for test/2 with `true(Expr)` opts: instead of calling `p_test_2`, inline `pj_emit_body(body_exprs) + pj_emit_goal(expr_check)`
- This gives X the same JVM local slot in body and check

**Task 2 — parse gaps (fix one at a time, re-run after each)**
- `:- else. / :- endif.` — add to meta-directive skip list in `pj_emit_main` (line ~7230)
- `f()` zero-arity compound — parser sees `f(` then `)` → fix `prolog_parse.c` term parser
- `-atom` as arg (e.g. `style_check(-no_effect)`) — prefix `-` on atom: fix unary minus to accept atoms
- `div` as infix in opts (`A == X div Y`) — `div` already in op table? check `prolog_lex.c`
- `:-` inside list term — parser needs to handle `:-` as a term functor

**Task 3 — throw/catch semantics (test_exception.pl)**
- `throw:error` (no exception) — inspect what test body does; likely `catch` swallowing
- `throw:ground/unbound/not/non_unify` — likely same root cause

```bash
git clone https://TOKEN@github.com/snobol4ever/snobol4x
git clone https://TOKEN@github.com/snobol4ever/.github
apt-get install -y --fix-missing default-jdk nasm libgc-dev swi-prolog
make -C snobol4x/src
# SWI test files: unzip swipl-devel-master.zip to /tmp/
# Read §NOW above. Start at CRITICAL NEXT ACTION.
```

**Key files:**
- `snobol4x/src/frontend/prolog/prolog_emit_jvm.c` — linker ~line 6995 (`pj_linker_*`)
- `snobol4x/test/frontend/prolog/plunit.pl` — shim (keep in sync with C string literal)
- SWI tests: `swipl-devel-master/tests/core/test_*.pl` (58 files)

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
| **M-PJ-ASSERTZ** | `assertz/1`, `asserta/1` — dynamic DB (Scrip dep) | ✅ |
| **M-PJ-RETRACT** | `retract/1` — peek-then-remove, 5/5 rung14 | ✅ |
| **M-PJ-ATOP** | `@<`/`@>`/`@=<`/`@>=` as parser infix operators — Scrip dep | ✅ |
| **M-PJ-SORT** | `sort/2`, `msort/2` — insertion sort, optional dedup | ✅ |
| **M-PJ-SUCC-PLUS** | `succ/2`, `plus/3` — successor/addition builtins | ✅ |
| **M-PJ-FORMAT** | `format/1`, `format/2` — ~w ~a ~n ~d ~i directives | ✅ |
| **M-PJ-NUMBER-VARS** | `numbervars/3` — name unbound vars as A,B,...Z,A1,...; `$VAR` write support | ✅ |
| **M-PJ-CHAR-TYPE** | `char_type/2` — alpha/alnum/digit/space/upper/lower/to_upper/to_lower/ascii | ✅ |
| **M-PJ-WRITE-CANONICAL** | `writeq/1`, `write_canonical/1`, `print/1`; atom quoting + symbolic token rules | ✅ |
| **M-PJ-SUCC-ARITH** | `max/min/sign/truncate/msb`; bitwise `/\ \/ xor >> <<`; `** ^`; prefix `\`; parser op table | ✅ |
| **M-PJ-STRING-IO** | `atom_string/2`, `number_string/2`, `string_concat/3`, `string_length/2`, `string_lower/2`, `string_upper/2`; rung24 5/5 | ✅ |
| **M-PJ-TERM-STRING** | `term_to_atom/2`, `term_string/2` (forward); rung25 3/3 | ✅ |
| **M-PJ-COPY-TERM** | `copy_term/2`, `string_to_atom/2`, `atomic_list_concat/2,3`, `concat_atom/2`; rung26 5/5 | ✅ |
| **M-PJ-AGGREGATE** | `aggregate_all/3` (count/sum/max/min/bag/set), `nb_setval/2`, `nb_getval/2`, `succ_or_zero/2`; rung27 5/5 | ✅ |
| **M-PJ-EXCEPTIONS** | `catch/3`, `throw/1` — ISO exception machinery; rung28 5/5 | ✅ |
| **M-PJ-NUMBER-OPS** | `sqrt/sin/cos/tan/exp/log/atan/atan2/float/float_integer_part/float_fractional_part/pi/e`; `truncate/ceiling/floor/round` float→int; `gcd/2`; rung29 5/5 | ✅ |
| **M-PJ-DCG** | DCG `-->` rules, `phrase/2,3`, `{}/1` inline goals, pushback notation; rung30 5/5 | ✅ |
| **M-PJ-PLUNIT-SHIM** | SWI `tests/core/` converted to standalone `.pro` (564 tests); loads+runs under SWI | ✅ |
| **M-PJ-LINKER** | plunit linker in prolog_emit_jvm.c — raw SWI .pl files compile directly; test_list 10/11 | ✅ |
| **M-PJ-SWI-BASELINE** | Run all 564 converted tests against JVM backend; record pass/fail baseline | ❌ |

---

## Session Bootstrap (every PJ-session)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y --fix-missing default-jdk nasm libgc-dev swi-prolog
make -C snobol4x/src
# Read §NOW above. Start at CRITICAL NEXT ACTION.
```

---

**Sprint order:** ASSERTZ → RETRACT → SORT → SUCC-PLUS → FORMAT → STRING-OPS → AGGREGATE → COPY-TERM → EXCEPTIONS → NUMBER-OPS → DCG.



*FRONTEND-PROLOG-JVM.md = L4. §NOW = ONE bootstrap block only — current session's next action. Prior session findings → SESSIONS_ARCHIVE.md only. Completed milestones → MILESTONE_ARCHIVE.md. Size target: ≤8KB total.*
