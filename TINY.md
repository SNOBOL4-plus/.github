# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.**

→ Rules: [RULES.md](RULES.md) · Beauty plan: [BEAUTY.md](BEAUTY.md) · History: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — B-282 (BEAUTY)
**HEAD:** `c16c575` B-282 (snobol4x)
**B-session:** M-BEAUTIFY-BOOTSTRAP ❌ — 3 bugs fixed; M-BEAUTY-GLOBAL ✅ M-BEAUTY-IS ✅ M-BEAUTY-ASSIGN ✅; match driver has pre-existing TxInList bug
**Invariants:** 106/106 ASM corpus ALL PASS ✅ · claws5 now 0 nasm errors ✅

**⚡ CRITICAL NEXT ACTION — B-283:**

```bash
cd /home/claude/snobol4x

# 1. Fix match driver — TxInList undefined (in beauty.sno, not inc/).
#    Rewrite test/beauty/match/driver.sno to use a simple string+pattern test.
#    Regenerate ref: snobol4 -f -P256k -Idemo/inc test/beauty/match/driver.sno > test/beauty/match/driver.ref

# 2. Run monitor for remaining subsystems in dependency order:
for sub in match tree ShiftReduce TDump Gen Qize ReadWrite XDump semantic; do
  INC=demo/inc X64_DIR=/home/claude/x64 MONITOR_TIMEOUT=60 \
    bash test/beauty/run_beauty_subsystem.sh $sub
done

# 3. After all 19 PASS → run beauty bootstrap:
WORK=$(mktemp -d /tmp/beau_XXXXXX); RT=src/runtime; INC=demo/inc
for f in asm/snobol4_stmt_rt.c snobol4/snobol4.c mock/mock_includes.c \
          snobol4/snobol4_pattern.c engine/engine.c asm/blk_alloc.c asm/blk_reloc.c; do
  gcc -O0 -g -c "$RT/$f" -I"$RT/snobol4" -I"$RT" -I"$RT/asm" \
      -Isrc/frontend/snobol4 -w -o "$WORK/$(basename $f .c).o"
done
./sno2c -asm -I"$INC" demo/beauty.sno > "$WORK/prog.s"
nasm -f elf64 -Isrc/runtime/asm/ "$WORK/prog.s" -o "$WORK/prog.o"
gcc -no-pie "$WORK"/*.o -lgc -lm -o "$WORK/beauty_asm"
snobol4 -f -P256k -Idemo/inc demo/beauty.sno < demo/beauty.sno > /tmp/oracle.sno
"$WORK/beauty_asm" < demo/beauty.sno > /tmp/asm_out.sno 2>/tmp/asm_err.txt
diff /tmp/oracle.sno /tmp/asm_out.sno | head -30
```

**Subsystems PASSING after B-282:** global ✅ is ✅ fence ✅ io ✅ case ✅ assign ✅ counter ✅ stack ✅
**Subsystems NOT YET RUN:** match (driver bug) tree ShiftReduce TDump Gen Qize ReadWrite XDump semantic omega trace

---

## Last Two Sessions (3 lines each)

**B-282 (2026-03-24) — 3 bugs fixed; M-BEAUTY-GLOBAL/IS/ASSIGN PASS; 106/106:**
(1) stmt_match_descr: FAILDESCR coerced to empty string matched everywhere; fix IS_FAIL_fn guard.
(2) stmt_setup_subject: stale subject_len_val on early FAILDESCR return; fix zero before return.
(3) E_NAM: emitted DT_S(1) not DT_N(9); stmt_nreturn_deref updated. HEAD `c16c575`.

**B-281 (2026-03-24) — E_STAR split from E_INDR; beauty_asm exit 0; 10/784 lines:**
emit_byrd_asm.c: *VAR runtime DT_P dispatch fixed; binary exits 0; Parse Error at main02 remains.
HEAD `a732d3b` B-281.
