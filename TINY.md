# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `t2-impl` — Technique 2 dynamic allocation + relocation
**HEAD:** `c6a6544` B-237 (asm-backend) · x64: `4fcb0e1` B-233
**Milestone:** M-MONITOR-4DEMO → M-MONITOR-CORPUS9 → M-MERGE-3WAY → **M-T2-RUNTIME (pivot target)**
**Invariants:** 97/106 ASM corpus (9 known failures fixed by T2 — no manual patches)

**⚠ CRITICAL NEXT ACTION — Session B-238:**

```bash
cd /home/claude/snobol4x && git checkout asm-backend
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git pull --rebase origin asm-backend   # HEAD should be c6a6544 B-237

export PATH=$PATH:/usr/local/bin:/home/claude/x64
export INC=/home/claude/snobol4corpus/programs/inc
export PROG=/home/claude/snobol4corpus/programs

# PHASE 1: complete M-MONITOR-4DEMO (still needed before merge)
bash test/monitor/precheck.sh   # expect 28+/30 (ASM null precheck bug is cosmetic)
bash test/monitor/run_monitor.sh $PROG/roman/roman.sno
bash test/monitor/run_monitor.sh $PROG/wordcount/wordcount.sno
bash test/monitor/run_monitor.sh $PROG/treebank/treebank.sno
bash test/monitor/run_monitor.sh $PROG/claws5/claws5.sno || true  # document count only

# PHASE 2: M-MERGE-3WAY — after M-MONITOR-4DEMO fires
# See MONITOR.md and chat strategy for the staged merge protocol:
#   1. git checkout -b merge-staging origin/asm-backend
#   2. git merge --no-ff jvm-backend  (resolve shared files, run invariants)
#   3. git merge --no-ff net-backend  (resolve, run invariants)
#   4. all invariants hold → PR into main → fan out v-post-merge tag
#   5. new branches: asm-t2, jvm-t2, net-t2 from v-post-merge

# PHASE 3: M-T2-RUNTIME — first T2 milestone after merge
# Write src/runtime/asm/t2_alloc.c:
#   t2_alloc(size_t sz) → mmap(NULL,sz,PROT_READ|PROT_WRITE,MAP_PRIVATE|MAP_ANONYMOUS,-1,0)
#   t2_free(void*p,size_t sz) → munmap(p,sz)
#   t2_mprotect_rx(void*p,size_t sz) → mprotect(p,sz,PROT_READ|PROT_EXEC)
#   t2_mprotect_rw(void*p,size_t sz) → mprotect(p,sz,PROT_READ|PROT_WRITE)
# Write test/t2/test_t2_alloc.c — allocate, write, mprotect_rx, verify read, free
# Compile: gcc -o /tmp/test_t2 test/t2/test_t2_alloc.c src/runtime/asm/t2_alloc.c
# Run: /tmp/test_t2 → PASS
```

## Last Session Summary

**Session B-238 (2026-03-21) — PIVOT: Technique 2 planned; milestones + HQ updated:**
- Designed 3-way merge strategy (staged, asm-backend base, jvm then net)
- Identified M-BOOTSTRAP prerequisite as unnecessary — `emit_byrd_asm.c` can emit
  relocation tables at compile time without self-hosting
- Added M-MERGE-3WAY + M-T2-RUNTIME through M-T2-FULL milestone chain to PLAN.md
- Updated BACKEND-X64.md: removed M-BOOTSTRAP gate, added T2 milestone sprint table
- Updated ARCH.md: replaced "Why not now" blocker with 2026-03-21 PIVOT note
- Updated TINY.md NOW section: sprint = t2-impl
- No code changes this session — planning session only

## Active Milestones

| ID | Status |
|----|--------|
| M-MONITOR-4DEMO | ❌ next to fire |
| M-MONITOR-CORPUS9 | ❌ (may be superseded by T2) |
| M-MERGE-3WAY | ❌ |
| M-T2-RUNTIME | ❌ |
| M-T2-RELOC | ❌ |

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-next | `asm-backend` | M-MONITOR-4DEMO → M-MERGE-3WAY → M-T2-RUNTIME |
| J-next | `jvm-backend` | TBD |
| N-next | `net-backend` | TBD |
| F-next | `main` | TBD |
