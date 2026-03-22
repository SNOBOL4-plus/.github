# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `t2-impl` — M-T2-INVOKE
**HEAD:** `1ceb92f` B-241 (asm-t2)
**Milestone:** M-MACRO-BOX ✅ → M-T2-INVOKE (next)
**Invariants:** 96/106 ASM corpus (9 known failures + 053 runtime)

**⚡ CRITICAL NEXT ACTION — Session B-242:**

```bash
cd /home/claude/snobol4x && git checkout asm-t2
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git pull --rebase origin asm-t2   # HEAD should be 1ceb92f B-241
export INC=/home/claude/snobol4corpus/programs/inc
export CORPUS=/home/claude/snobol4corpus/crosscheck

# M-T2-INVOKE: emit T2 call-sites at every named-box invocation
# For each user-defined function call (α entry of named box):
#   1. t2_alloc(box_X_data_size)  → rsi = new_data ptr
#   2. memcpy(new_data, box_X_data_template, box_X_data_size)
#   3. mov r12, new_data
#   4. jmp box_X_α  (TEXT still static — no relocation needed yet)
# γ/ω: emit t2_free(old_r12, box_X_data_size) + restore caller r12 before return jump
# Acceptance: bash test/crosscheck/run_crosscheck_asm_corpus.sh → 96/106 (invariant holds)
```

## Last Session Summary

**Session B-241 (2026-03-21) — M-MACRO-BOX ✅:**
- bref() fix: 18 emitter call sites patched — saved/cursor_save now resolve to [r12+N] in box context
- ARBNO macroized: ARBNO_ALPHA/BETA/CHILD_OK/CHILD_FAIL added to snobol4_asm.mac
- emit_arbno() replaced 35 lines of raw inline asm with 4 macro calls using bref()
- 96/106 corpus — invariant holds; commit 1ceb92f pushed

**Session B-240 (2026-03-21) — M-T2-EMIT-SPLIT ✅ ⚠:**
- Emitter splits named boxes into TEXT+DATA sections; r12=DATA-block pointer
- 3 regressions (bare .bss symbol refs) — fixed in B-241

## Active Milestones

| ID | Status |
|----|--------|
| M-MACRO-BOX     | ✅ `1ceb92f` B-241 |
| M-T2-INVOKE     | ❌ next |
| M-T2-RECUR      | ❌ |
| M-T2-CORPUS     | ❌ |
| M-T2-FULL       | ❌ |

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-next | `asm-t2` | M-T2-INVOKE |
| J-next | `jvm-t2` | TBD |
| N-next | `net-t2` | TBD |
| F-next | `main`   | TBD |
