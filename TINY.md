# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `monitor-ipc` — wire 5-way FIFO IPC; SPITBOL + JVM + NET participants
**HEAD:** `6eebdc3` B-229 (asm-backend) · x64: `feb521b` B-231
**Milestone:** M-X64-S2 (next to fire)
**Invariants:** 97/106 ASM corpus (9 known failures: 022, 055, 064, cross, word1-4, wordcount)

**⚠ CRITICAL NEXT ACTION — Session B-232:**

```bash
# Repo: snobol4ever/x64
cd /home/claude/x64
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git pull --rebase origin main

# M-X64-S2: replace callef() in osint/syslinux.c with x64 direct implementation
# Root cause confirmed (B-231): MINSAVE()→pushregs()→save_regs corrupts reg_pc
# efb->efcod verified at offset 32 (raw dump: raw[4]=valid xnblk ptr)
# pfn at xnblk+24 (xndta[1])
#
# Step 1: Replace callef (lines ~104-277 of syslinux.c) with:
#   pnode = efb->efcod  (offset 32)
#   pfn = pnode->xnu.xndta[1]  (offset 24)
#   build struct descr cargs[nargs] from icblk.val (offset 8) on sp[]
#     sp layout: sp[nargs-1-i] = arg i (last-pushed = sp[0])
#   rc = pfn(&retval, nargs, cargs)
#   if rc==FALSE return 0 (FAIL)
#   pack retval.a.i into tscblk as icblk{typ=b_icl,val=retval.a.i}
#   return (union block*)&tscblk
#
# Step 2: make bootsbl CFLAGS="... -rdynamic"
# Step 3: ./bootsbl test_spl_add.sno → expect "PASS: spl_add(3,4) = 7"
# Step 4: M-X64-S2 fires → proceed to M-X64-S3 (UNLOAD lifecycle)
#
# Key verified facts from B-231 diagnostics:
#   struct icblk: { word typ(8); long val(8) } — val at offset 8
#   struct descr: { union{long i;double f} a; char f; uint v } — size 16
#   Integer type code v=2 (conint from equ.h)
#   tscblk is valid scratch block for return value
#   libspl.so compiled, spl_add/spl_strlen use lowercase names (SPITBOL folds)
#   -rdynamic required on bootsbl link for dlopen to resolve symbols
```

## Last Session Summary

**Session B-231 (2026-03-21) — M-X64-S1 ✅ + M-X64-S2 diagnostic:**
- M-X64-S1 fired (88ff40f): all compile errors fixed, make bootsbl EXIT 0
- M-X64-S2 diagnostic: LOAD()→zysld→loadDll→dlopen succeeds; callef entered
- Segfault root cause: MINSAVE()→pushregs()→save_regs corrupts reg_pc
- efb layout verified from raw dump: efcod at offset 32, valid xnblk ptr
- libspl.c/libspl.so written; sysld.c/sysex.c got missing stdio.h
- Pushed feb521b WIP

## Active Milestones

| ID | Trigger | Status |
|----|---------|--------|
| M-MONITOR-IPC-SO | monitor_ipc.so built; MON_OPEN/MON_SEND/MON_CLOSE; CSNOBOL4 LOAD() confirmed | ✅ `8bf1c0c` B-229 |
| M-MONITOR-IPC-CSN | inject_traces.py IPC preamble; CSNOBOL4 trace via FIFO; hello PASS | ✅ `6eebdc3` B-229 |
| **M-X64-S1** | syslinux.c compiles clean; `make bootsbl` succeeds | ❌ |
| **M-X64-S2** | LOAD end-to-end; spl_add(3,4)=7 | ❌ |
| **M-X64-S3** | UNLOAD lifecycle; reload; double-unload safe | ❌ |
| **M-X64-S4** | SNOLIB; errors 139/140/141; monitor_ipc.so in SPITBOL | ❌ |
| **M-X64-FULL** | S1–S4 done; SPITBOL = monitor participant | ❌ |
| M-MONITOR-IPC-5WAY | all 5 participants via FIFO; hello PASS all 5; no stderr/stdout blending | ❌ |
| M-MONITOR-IPC-TIMEOUT | monitor_collect.py watchdog: FIFO silence > T sec → kill + report | ❌ |
| M-MONITOR-4DEMO | roman+wordcount+treebank pass all 5 | ❌ |

Full milestone history → [PLAN.md](PLAN.md) · Beauty detail → [BEAUTY.md](BEAUTY.md)

---

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-229 | `asm-backend` | monitor-ipc — IPC-SO + IPC-CSN ✅ done |
| x64-fork | `snobol4ever/x64 main` | M-X64-S1: fix syslinux.c xndta fields; make bootsbl |
| J-next | `jvm-backend` | TBD |
| N-next | `net-backend` | TBD |
| F-next | `main` | TBD |
