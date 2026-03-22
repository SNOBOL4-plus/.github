# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `t2-impl` — M-T2-RECUR
**HEAD:** `1cf8a0a` B-243 (asm-t2)
**Milestone:** M-T2-INVOKE ✅ → M-T2-RECUR (next)
**Invariants:** 96/106 ASM corpus (9 known failures + 053 runtime)

**⚡ CRITICAL NEXT ACTION — Session B-244:**

```bash
cd /home/claude/snobol4x && git checkout asm-t2
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git pull --rebase origin asm-t2   # HEAD should be B-243
export INC=/home/claude/snobol4corpus/programs/inc
export CORPUS=/home/claude/snobol4corpus/crosscheck

# Invariant check first:
bash test/crosscheck/run_crosscheck_asm_corpus.sh   # expect 96/106

# M-T2-RECUR: recursive SNOBOL4 functions correct under T2
# Two simultaneous live DATA blocks, one shared CODE.
# roman.sno must produce correct output for recursive inputs.
# Key: each ucall site already does t2_alloc+memcpy+push r12+t2_free.
# Verify with: INC=... ./snobol4-asm demo/roman.sno (pipe test inputs)
# Then run: bash test/crosscheck/run_crosscheck_asm_corpus.sh
```

## Last Session Summary

**Session B-243 (2026-03-22) — M-T2-INVOKE ✅ + ARBNO rename:**
- ARBNO_CHILD_OK → ARBNO_α1, ARBNO_CHILD_FAIL → ARBNO_β1 (shorter Greek names)
- T2 call-site protocol: push r12 (before ret pushes) → t2_alloc+memcpy → mov r12,rax → write ret slots → jmp α
- FN_α_INIT: no-op (alloc happens at call site)
- FN_γ/FN_ω: simple jmp [ret] (free happens inline at return labels via pop rdi+t2_free)
- SPAN_α/BREAK_α: r12 scratch → r13 (r12 reserved as T2 DATA ptr)
- t2_alloc.o + t2_reloc.o added to crosscheck script and snobol4-asm driver
- extern t2_alloc, t2_free, memcpy added to generated .s header
- 96/106 invariant holds; 5 artifacts regenerated; commit B-243 pushed

## Active Milestones

| ID | Status |
|----|--------|
| M-T2-INVOKE     | ✅ `1cf8a0a` B-243 |
| M-T2-RECUR      | ❌ next |
| M-T2-CORPUS     | ❌ |
| M-T2-FULL       | ❌ |

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-next | `asm-t2` | M-T2-INVOKE |
| J-next | `jvm-t2` | TBD |
| N-next | `net-t2` | TBD |
| F-next | `main`   | TBD |
