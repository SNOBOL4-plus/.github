# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.**

→ Rules: [RULES.md](RULES.md) · Beauty plan: [BEAUTY.md](BEAUTY.md) · History: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — B-280 (BEAUTY) · F-223 (Prolog) concurrent
**HEAD:** `a4f44a3` B-280 (snobol4x)
**B-session:** M-BEAUTIFY-BOOTSTRAP ❌ — CSNOBOL4 ✅; ASM exit 0 ✅; output 10/784 lines; Parse Error at main02
**F-session:** M-PROLOG-CORPUS ❌ — rung05 encoding fix attempted, reverted clean
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — B-281 (M-BEAUTIFY-BOOTSTRAP continued):**

```bash
cd /home/claude/snobol4x
# Build beauty ASM binary (sno2c already fixed in B-280):
WORK=$(mktemp -d /tmp/beau_XXXXXX); RT=src/runtime; INC=demo/inc
for f in asm/snobol4_stmt_rt.c snobol4/snobol4.c mock/mock_includes.c \
          snobol4/snobol4_pattern.c engine/engine.c; do
  gcc -O0 -g -c "$RT/$f" -I"$RT/snobol4" -I"$RT" -Isrc/frontend/snobol4 -w -o "$WORK/$(basename $f .c).o"
done
gcc -O0 -g -c "$RT/asm/blk_alloc.c" -I"$RT/asm" -w -o "$WORK/blk_alloc.o"
gcc -O0 -g -c "$RT/asm/blk_reloc.c" -I"$RT/asm" -w -o "$WORK/blk_reloc.o"
./sno2c -asm -I"$INC" demo/beauty.sno > "$WORK/prog.s"
nasm -f elf64 -I"$RT/asm/" "$WORK/prog.s" -o "$WORK/prog.o"
gcc -no-pie "$WORK"/*.o -lgc -lm -o "$WORK/beauty_asm"
echo "build: $?"

# Run and diff:
snobol4 -f -P256k -Idemo/inc demo/beauty.sno < demo/beauty.sno > /tmp/oracle.sno
"$WORK/beauty_asm" < demo/beauty.sno > /tmp/asm_out.sno 2>/tmp/asm_err.txt
echo "ASM exit: $?  lines: $(wc -l < /tmp/asm_out.sno)"
diff /tmp/oracle.sno /tmp/asm_out.sno | head -30
```

**BUG B-280-PARSE-ERROR — beauty.sno emits "Parse Error" at main02/main05:**
```
ROOT CAUSE: Pattern `Src POS(0) *Parse *Space RPOS(0)` fails on Src at runtime.
  The Parse pattern variable is set by the parser. Pop() returns the parsed stmt.
  The failure means either:
    (a) Parse pattern is not matching/consuming the full Src buffer, OR
    (b) Pop() returns DIFFER-fail (stack empty), OR
    (c) The Src accumulation in main02 is wrong (Line concatenation bug)
  DIAGNOSE: Add xTrace output around main02 to see Src contents and Parse value.
  Compare Src/Parse values between CSNOBOL4 and ASM runs using the monitor.
  Run: bash test/beauty/run_beauty_subsystem.sh global (confirm still passing)
  Then: diagnose main02 Parse pattern match failure vs oracle.
```

**⚡ F-CRITICAL NEXT ACTION — F-224 (M-PROLOG-CORPUS):**

```
BUG: rung05 backtrack FAIL — prints a\nb instead of a\nb\nc.
ROOT CAUSE: prolog_emit.c emit_body last-goal user-call branch (~line 692).
  PG(γ) returns clause_idx. Caller increments. switch hits default → ω.
  Inner _cs lost. On retry _cs resets to 0, re-finds b not c.

RECOMMENDED FIX — inner_cs out-param:
  Change _r signature: int pl_F_r(args, Trail*, int _start, int *_ics_out)
  After _cr = pl_F_r(..., _lcs, &_ics): *_ics_out = _ics; goto γ;
  Caller retry: pass &_ics, on retry call with _start=ci, _ics pre-set.

After fix: run all 10 rungs → M-PROLOG-CORPUS fires.
```

---

## Last Two Sessions (3 lines each)

**B-280 (2026-03-24) — 3 emitter bugs fixed; beauty_asm now exit 0; Parse Error at main02:**
emit_byrd_asm.c: (1) cur_fn not set for DEFINE-body labels (body_label fallback); (2) cur_fn leaked past last fn body (body_end_idx + end_label pre-scan); (3) pre-scan must run before Pass 3. Result: 0 nasm errors, exit 0, 10/784 output lines, Parse Error. 106/106 corpus ALL PASS. HEAD `a4f44a3`.

**B-279 (2026-03-24) — ASM nasm errors fixed; binary assembles+links; runtime segfault next:**
3 bugs in emit_byrd_asm.c: (1) fn-body pattern BSS slots routed into per-box DATA via box_ctx; (2) ANY/SPAN/BREAK expr-tmp tlab/plab use flat_bss_register (direct label required by macros); (3) MAX_VARS 512→2048. Result: 0 nasm undefined symbols. Segfault on runtime run. HEAD `4bc319c`.
