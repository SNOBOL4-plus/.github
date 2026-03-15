# FRONTEND-BEAUTY.md — beauty.sno (L3)

`beauty.sno` is the shared SNOBOL4 frontend: a self-contained parser and
pretty-printer written in SNOBOL4. It is the proof-of-concept for the whole
project — if a backend can run beauty.sno correctly, it can run anything.

*Session state → TINY.md. Testing paradigms → TESTING.md. sno2c compiler → FRONTEND-SNO2C.md.*

---

## What beauty.sno Does

Reads SNOBOL4 source on stdin, pretty-prints it to stdout.
Uses a Shift/Reduce parse tree built by pattern actions (nPush/nPop/~/&).
Two-pass: `pp(sno)` walks the tree and emits; `qq(sno)` measures width for line-break decisions.

Key functions:
- `snoCommand` — ARBNO pattern matching one token
- `snoStmt` — one full statement
- `snoLabel` — optional label
- `snoParse` — top-level: `Src POS(0) *Parse *Space RPOS(0) → pp(sno)`
- `pp(sno)` — walk tree, emit beautified output
- `qq(sno)` — measure flat width (lookahead for pp line-break decisions)
- `pp_Parse` — parse tree walker for pp

Includes 19 helper libraries via `-INCLUDE` from `SNOBOL4-corpus/programs/inc/`.

---

## Four-Paradigm TDD Protocol (toward M-BEAUTY-FULL)

| Sprint | Paradigm | What it catches | Milestone trigger |
|--------|----------|-----------------|-------------------|
| `beauty-crosscheck` | Crosscheck — diff vs oracle | Wrong output | beauty/140_self → **M-BEAUTY-CORE** |
| `beauty-probe` | Probe — &STLIMIT frame-by-frame | WHERE divergence first appears | All failures diagnosed |
| `beauty-monitor` | Monitor — TRACE double-trace | Deep recursion / call-return bugs | Trace streams match |
| `beauty-triangulate` | Triangulate — cross-engine | Edge cases, CSNOBOL4 quirks | Empty diff → **M-BEAUTY-FULL** |

---

## Rung 12 Test Format

Tests live in `SNOBOL4-corpus/crosscheck/beauty/`:
- `NNN_name.input` — SNOBOL4 snippet to pipe into beauty_full_bin
- `NNN_name.ref` — oracle: `snobol4 -f -P256k -I$INC $BEAUTY < NNN_name.input`

Runner: `SNOBOL4-tiny/test/crosscheck/run_beauty.sh` (pre-compiled binary, not sno2c).

Test progression (add one at a time, never skip):
```
101_comment     * a comment
102_output          OUTPUT = 'hello'
103_assign          X = 'foo'
104_label       LOOP    X = X 1
105_goto            :(END)
106_pattern         X Y = 'ab'
107_define          DEFINE('fn(a)','fn_label')
108_sf_goto         X = Y   :S(A)F(B)
109_multi       5-line program
110_expression      X = (1 + 2) * 3
120_real_prog   20-line program
130_inc_file    program using -INCLUDE
140_self        full beauty.sno → M-BEAUTY-CORE
```

Generate oracle for a new test:
```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
snobol4 -f -P256k -I$INC $BEAUTY < NNN_name.input > NNN_name.ref
```

---

## Probe Script (Paradigm 2)

When crosscheck fails — find exactly which statement diverges:
```bash
python3 /home/claude/SNOBOL4-harness/probe/probe.py \
    --oracle csnobol4 --max 200 failing.sno
```
Probe targets in priority order: `pp`, `snoCommand`, `snoLabel`, `qq`, `pp_Parse`.

Manual binary comparison against compiled binary:
```bash
for N in $(seq 1 200); do
    echo "=== STEP $N ==="
    printf "&STLIMIT = $N\n&DUMP = 2\n" > /tmp/pre.sno
    cat /tmp/pre.sno $BEAUTY | ./beauty_full_bin < input.sno 2>&1 | grep -v "^Normal"
done > compiled_frames.txt
```

---

## Monitor Script (Paradigm 3)

When probe finds divergence inside recursion — trace every call/return:

`beauty_trace.sno` — prepend to beauty.sno or patch inline:
```snobol4
        TRACE('pp','CALL')
        TRACE('pp','RETURN')
        TRACE('qq','CALL')
        TRACE('qq','RETURN')
        TRACE('pp_Parse','CALL')
        TRACE('pp_Parse','RETURN')
        TRACE('snoCommand','CALL')
        TRACE('snoStmt','CALL')
        TRACE('snoLabel','CALL')
        TRACE('snoLine','VALUE')
        TRACE('snoSrc','VALUE')
```

```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
snobol4 -f -P256k -I$INC beauty_trace.sno < input.sno 2>oracle_trace.txt
./beauty_full_bin_trace < input.sno 2>compiled_trace.txt
diff oracle_trace.txt compiled_trace.txt | head -20
```

TRACE gotcha: `TRACE(...,'KEYWORD')` non-functional on CSNOBOL4 and SPITBOL.
Use `TRACE('varname','VALUE')` instead. `&STCOUNT` broken in CSNOBOL4 (always 0).

---

## Triangulate Script (Paradigm 4)

Full self-beautification — M-BEAUTY-FULL trigger:
```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
snobol4 -f -P256k -I$INC $BEAUTY < $BEAUTY > oracle_csn.txt
./beauty_full_bin < $BEAUTY > compiled_out.txt
diff oracle_csn.txt compiled_out.txt   # empty = M-BEAUTY-FULL FIRES
```

SPITBOL note: disqualified for full beauty.sno (error 021 at END — indirect function
call semantic difference). Use for individual sub-programs only.
Two oracles agree, compiled differs → our bug.
Two oracles disagree → semantic edge case, check Gimpel §7.

---

## compiler.sno Strategy (post-M-BEAUTY-FULL)

`compiler.sno` = `beauty.sno` + `compile(sno)` replacing `pp(sno)`.
Same grammar, same Shift/Reduce tree. Final action emits C Byrd boxes instead of
pretty-printed SNOBOL4. One new function walking the proven tree. Minimal delta.

Architecture A (future, harder): sprinkle emit actions into pattern alternations
using `epsilon . *action(...)` — like `iniParse` in `programs/inc/ini.sno`.
Eliminates pp/qq post-processing. Pursue after M-BOOTSTRAP.
