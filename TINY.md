# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — M-MONITOR-SYNC in progress
**HEAD:** `e3d2bdb` B-254 (main)
**Milestone:** M-MONITOR-SYNC — sync-step barrier protocol; hello PASS all 5 sync = fire
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — Session B-255 (M-MONITOR-SYNC: cycle through divergences):**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git remote set-url origin https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x.git
git pull --rebase origin main

# Setup (if fresh container) — tarball at /mnt/user-data/uploads/snobol4-2_3_3_tar.gz:
bash setup.sh
ln -sf /home/claude/snobol4x/sno2c /home/claude/sno2c_net
cd /home/claude/x64 && make 2>&1 | tail -3
[ -e /home/claude/x64/bootsbl ] || ln -sf /home/claude/x64/sbl /home/claude/x64/bootsbl
gcc -shared -fPIC -O2 -Wall -o /home/claude/x64/monitor_ipc_spitbol.so \
    /home/claude/x64/monitor_ipc_spitbol.c
gcc -shared -fPIC -O2 -Wall \
    -o test/monitor/monitor_ipc_sync.so test/monitor/monitor_ipc_sync.c

# State at B-254 handoff:
#   5-way sync barrier working — all 5 connect and step together
#   Remaining divergence: ASM emits VALUE TAB='\t' at step 1 (pre-init constant)
#   Fix needed: gate comm_var() in snobol4.c to skip pre-init constants
#   (tab, ht, nl, lf, cr, ff, vt, bs, nul, epsilon, fSlash, bSlash, semicolon, UCASE, LCASE)

# Cycle protocol (repeat until hello PASS all 5):
# 1. Run monitor:
INC=/home/claude/snobol4corpus/programs/inc X64_DIR=/home/claude/x64 \
  MONITOR_TIMEOUT=15 bash test/monitor/run_monitor_sync.sh \
  /home/claude/snobol4corpus/crosscheck/hello/hello.sno
# 2. Read first DIVERGENCE line → identify which participant + variable
# 3. Fix → rebuild → repeat
```

## Last Session Summary

**Session B-252 (2026-03-22) — M-MONITOR-SYNC wiring:**
- JVM: added `sno_mon_ack_fd` static field; `sno_mon_init` opens both MONITOR_FIFO+MONITOR_ACK_FIFO; `sno_mon_var` blocks on ack after each write, exits on non-G
- NET: added `net_mon_sw`/`net_mon_ack` static fields; `net_mon_init()` new method (static-open both FIFOs, called from main); `net_mon_var` rewritten (no per-call StreamWriter, reads ack)
- `run_monitor_sync.sh`: fixed launch-order deadlock — participants start first, then controller opens FIFOs
- Remaining: LOAD error 142 on `monitor_ipc_sync.so` path — needs one more debug step
- 106/106 ALL PASS unchanged

## Active Milestones

| ID | Status |
|----|--------|
| M-MONITOR-SYNC     | ❌ one step away — LOAD path fix + hello PASS |
| M-MONITOR-4DEMO    | ❌ blocked on M-MONITOR-SYNC + 4 bug milestones |
| M-MON-BUG-NET-TIMEOUT | ❌ (resolved by static-open in B-252) |
| M-MON-BUG-SPL-EMPTY   | ❌ |
| M-MON-BUG-ASM-WPAT    | ❌ |
| M-MON-BUG-JVM-WPAT    | ❌ |

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-next | `main` | M-MONITOR-SYNC finish |
| F-212  | `main` | M-PROLOG-TERM — term.h + pl_unify.c + pl_atom.c (Sprint 1) |
