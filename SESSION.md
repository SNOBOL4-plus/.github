## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `crosscheck-ladder` — Sprint 3 of 6 toward M-BEAUTY-CORE |
| **Milestone** | M-BEAUTY-CORE (mock includes first) → M-BEAUTY-FULL (real inc, second) |
| **HEAD** | `e2ca252` — artifact: beauty_tramp_session93.c — CHANGED, 15638 lines |

---

## ⚡ SESSION 94 FIRST ACTION — Fix E_ATP bug, finish rung 8, start rung 9

### Bug 1: E_ATP capturing to `_` instead of varname

`@NH ANY(V) . CROSS` — `@NH` should capture cursor into `NH`.
E_ATP handler in emit_byrd.c uses `pat->right->sval` but the parse
stores the variable in `pat->right` (as an E_VART). Check parse.c
line 235: `case T_AT: uk=E_ATP; break;` — E_ATP is unary, so the
operand is `pat->right`. But the handler has:

```c
const char *varname = (pat->right && pat->right->sval) ? pat->right->sval : "_";
```

The bug is that `pat->right` is the child expression node (E_VART with sval="NH"),
so `pat->right->sval` SHOULD be "NH". But generated code shows `NV_SET_fn("_", ...)`.

Debug: add fprintf(stderr, "E_ATP varname=%s\n", varname) in the E_ATP case and
rerun cross. Also check parse.c line 424 — E_ATP may be parsed differently
(left-unary vs right-unary).

```bash
grep -n "E_ATP\|T_AT" src/sno2c/parse.c | head -20
```

### Bug 2: word4 — BREAKX not implemented

`BREAKX(cs)` — like BREAK but fails if no non-cs chars present (stricter).
Add to emit_byrd.c E_FNC handlers: BREAKX(cs) — same as BREAK but adds
check that cursor advanced at least 1 char (delta > 0).

### Build commands
```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
apt-get install -y libgc-dev
make -C src/sno2c

RT=src/runtime
CORPUS=/home/claude/SNOBOL4-corpus/crosscheck

bash /tmp/run_rung.sh $RT $CORPUS strings    # should be 15/17 baseline
```

### run_rung.sh (recreate if container is fresh)
```bash
cat > /tmp/run_rung.sh << 'SCRIPT'
RT=$1; CORPUS=$2; RUNG=$3
pass=0; fail=0
for sno in $CORPUS/$RUNG/*.sno; do
    name=$(basename $sno .sno)
    ref="${sno%.sno}.ref"
    inp="${sno%.sno}.input"
    [ -f "$ref" ] || continue
    src/sno2c/sno2c -trampoline "$sno" > /tmp/t.c 2>/dev/null
    gcc -O0 -g /tmp/t.c $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
        $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
        -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o /tmp/tbin 2>/dev/null
    if [ -f "$inp" ]; then
        got=$(timeout 5 /tmp/tbin < "$inp" 2>/dev/null || true)
    else
        got=$(timeout 5 /tmp/tbin </dev/null 2>/dev/null || true)
    fi
    exp=$(cat "$ref")
    if [ "$got" = "$exp" ]; then
        echo "PASS $name"; pass=$((pass+1))
    else
        echo "FAIL $name"
        diff <(echo "$exp") <(echo "$got") | head -4 | sed 's/^/  /'
        fail=$((fail+1))
    fi
done
echo "--- $RUNG: $pass pass, $fail fail ---"
SCRIPT
```

### Oracle note
The `.ref` files ARE the oracle — pre-generated from CSNOBOL4.
No need to build SPITBOL or CSNOBOL4. Never run them live.
Two executables compared:
1. `sno2c -trampoline foo.sno` → gcc → binary (with optional `.input` on stdin)
2. `cat foo.ref` — static ground truth

---

## Crosscheck ladder status (Session 93)

| Rung | Dir | Tests | Status |
|------|-----|-------|--------|
| 1 output | output/ | 8 | ✅ 8/8 |
| 2 assign | assign/ | 8 | ✅ 8/8 |
| 3 concat | concat/ | 6 | ✅ 6/6 |
| 4 arith | arith_new/ | 8 | ✅ 8/8 |
| 5 control | control_new/ | 7 | ✅ 7/7 |
| 6 patterns | patterns/ | 20 | ✅ 20/20 |
| 7 capture | capture/ | 7 | ✅ 7/7 |
| 8 strings | strings/ | 17 | ⏳ 15/17 — 2 failures |
| 9 keywords | keywords/ | 11 | ❌ not started |
| 10 functions | functions/ | 8 | ❌ |
| 11 data | data/ | 6 | ❌ |
| 12 beauty.sno | TBD | TBD | ❌ |

**Total so far: 71/73 pass**

Rung 8 remaining failures:
- `cross` — E_ATP `@NH` captures to `_` instead of `NH`
- `word4` — BREAKX not implemented

---

## Benchmarks — sno2c vs SPITBOL/CSNOBOL4

Benchmarks live in `SNOBOL4-corpus/benchmarks/`.
Run with: `sno2c -trampoline bench.sno` → gcc -O2 → binary (no stdin).
Reference numbers in `SNOBOL4-corpus/BENCHMARKS.md`.
**Do NOT build SPITBOL or CSNOBOL4** — use ref numbers from BENCHMARKS.md.

arith_loop, var_access, string_concat, string_manip all compile and run.
All produce correct integer output (coerce_numeric fix landed this session).
TIME() returns 0 (no real timer implementation yet — acceptable).

---

## Fixes made this session (Session 93)

| Fix | File | What |
|-----|------|------|
| ? stmt operator | parse.c | S ? P and S  ?  P both parse; = after ? allowed |
| E_NAM cond capture | emit_byrd.c | deferred via pending-cond list; flushed at _byrd_ok and _PAT_γ |
| Null replacement | emit.c | X pat = deletes match; has_eq+NULL replacement emits splice |
| E_ATP stub | emit_byrd.c | @VAR emits NV_SET of cursor as int — bug: → _ not varname |
| coerce_numeric | snobol4.c | integer-string args in arith coerced to DT_I; null → 0 |
| run_rung.sh | /tmp | pipes .input file to binary when present |

---

## CRITICAL Rules

- **NEVER write the token into any file**
- **NEVER link engine.c** — mock_engine.c only
- **ALWAYS run `git config user.name/email` after every clone**
- **beauty_core (mock includes) FIRST — beauty_full (real inc) SECOND**
- **beauty.sno is NEVER modified — it is syntactically perfect**
- **-INCLUDE is a noop in sno2c lexer — no -I flag needed**
- **Do NOT build SPITBOL or CSNOBOL4 — .ref files ARE the oracle**

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-14 | Session 84 SIL rename | DESCR_t/DTYPE_t/XKIND_t/_fn/_t throughout |
| 2026-03-14 | Session 85 cleanup | agreement breach resolved, rename audit |
| 2026-03-14 | Session 87 renames | inc_stubs→inc_mock, snobol4_inc→mock_includes |
| 2026-03-14 | Session 88 bug fix | nInc beta now emits NDEC_fn() — ntop leak resolved |
| 2026-03-15 | Session 89 PIVOT | crosscheck-ladder replaces smoke test |
| 2026-03-15 | Session 90 | Rungs 1-5 37/37; -INCLUDE noop; mock_engine renamed; 5 bugs fixed |
| 2026-03-15 | Session 91 | Rung 6 20/20; bare builtins as E_VART; dynamic POS/TAB args |
| 2026-03-15 | Session 92 | Rung 7 7/7; SNO_MSTART; null replace; pat_is_anchored POS(0) only |
| 2026-03-15 | Session 93 | Rung 8 15/17; ? op; E_NAM cond; coerce_numeric; E_ATP stub |
