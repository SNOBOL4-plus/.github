# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `monitor-ipc` — replace stderr/stdout blending with FIFO IPC via LOAD'd C module
**HEAD:** `19e26ca` B-227 (asm-backend) · `b67d0b1` J-212 (jvm-backend) · `2c417d7` N-209 (net-backend) · `6495074` F-210 (main)
**Milestone:** M-MONITOR-IPC-SO (next to fire)
**Invariants:** 100/106 C (6 pre-existing) · 26/26 ASM

**⚠ CRITICAL NEXT ACTION — Session B-228:**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git checkout asm-backend && git pull --rebase origin asm-backend

export CORPUS=/home/claude/snobol4corpus/crosscheck
STOP_ON_FAIL=0 CORPUS=$CORPUS bash test/crosscheck/run_crosscheck.sh   # 100/106
CORPUS=$CORPUS bash test/crosscheck/run_crosscheck_asm.sh               # 26/26

# IPC design — see MONITOR.md §IPC Architecture
# Step 1: build test/monitor/monitor_ipc.c → monitor_ipc.so
#   MON_OPEN(fifo_path)  — opens named FIFO, stores fd in .so global
#   MON_SEND(kind, body) — writes "KIND body\n" atomically to FIFO fd
#   MON_CLOSE()          — closes FIFO fd
#   Compatible ABI for both CSNOBOL4 and SPITBOL LOAD()
# Step 2: update inject_traces.py — emit LOAD()/MON_OPEN() preamble
#   MONCALL/MONRET/MONVAL now call MON_SEND() not TERMINAL=
# Step 3: update run_monitor.sh — mkfifo, set MONITOR_FIFO env var,
#   launch collector + all participants, wait, diff .norm files
# Step 4: update ASM runtime comm_var() — write to MONITOR_FIFO path not fd 2
# Pass: run_monitor.sh hello.sno exits 0; all streams via FIFO, none via stderr
```

**Full IPC plan → [MONITOR.md](MONITOR.md) §IPC Architecture**

## Last Session Summary

**Session B-228 (strategize, 2026-03-21) — IPC monitor architecture:**
- Studied CSNOBOL4 2.3.3 source (load.h, fork.c, ffi.c) and SPITBOL x64 source (syslinux.c, sysld.c)
- Both use identical dlopen/dlsym LOAD() ABI: `lret_t fn(LA_ALIST)`, RETSTR/RETINT/RETFAIL macros
- One `monitor_ipc.so` serves CSNOBOL4 + SPITBOL — binary compatible
- Key insight: replace TERMINAL= callbacks and comm_var stderr with named FIFO IPC
- No stdout/stderr blending ever; parallel participant execution; runtime panics stay clean
- New milestones: M-MONITOR-IPC-SO → M-MONITOR-IPC-CSN → M-MONITOR-IPC-5WAY replacing old 3WAY/5WAY approach
- Updated MONITOR.md §IPC Architecture; updated PLAN.md milestone dashboard

## Active Milestones

| ID | Trigger | Status |
|----|---------|--------|
| M-MONITOR-SCAFFOLD | test/monitor/ exists; CSNOBOL4 + ASM; one test passes | ✅ `19e26ca` B-227 |
| M-MONITOR-IPC-SO | monitor_ipc.so built; MON_OPEN/MON_SEND/MON_CLOSE work; CSNOBOL4 LOAD() confirmed | ❌ |
| M-MONITOR-IPC-CSN | inject_traces.py emits LOAD/MON_OPEN preamble; CSNOBOL4 trace via FIFO; hello PASS | ❌ |
| M-MONITOR-IPC-5WAY | all 5 participants via FIFO; hello PASS all 5; no stderr/stdout blending | ❌ |
| M-MONITOR-IPC-TIMEOUT | monitor_collect.py watchdog: FIFO silence > T sec → kill + report last trace event | ❌ |
| M-MONITOR-4DEMO | roman+wordcount+treebank pass all 5 | ❌ |
| M-BEAUTY-GLOBAL | global.sno driver passes | ❌ |
| M-BEAUTY-IS … M-BEAUTY-TRACE | 18 more subsystem drivers | ❌ |
| M-BEAUTIFY-BOOTSTRAP | beauty.sno fixed point on all 3 backends | ❌ |

Full milestone history → [PLAN.md](PLAN.md) · Beauty detail → [BEAUTY.md](BEAUTY.md)

---

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-228 | `asm-backend` | monitor-ipc — build monitor_ipc.so + wire IPC |
| J-next | `jvm-backend` | TBD |
| N-next | `net-backend` | TBD |
| F-next | `main` | TBD |

Per RULES.md: `git pull --rebase` before every push. Update only your row in PLAN.md NOW table.
