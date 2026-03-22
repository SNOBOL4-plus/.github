# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — MONITOR sprint; 4 bug milestones filed blocking M-MONITOR-4DEMO
**HEAD:** `e2c4fb5` B-249 (main)
**Milestone:** M-MON-BUG-NET-TIMEOUT next (blocks M-MONITOR-4DEMO)
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — Session B-251 (M-MON-BUG-NET-TIMEOUT):**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git remote set-url origin https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x.git
git pull --rebase origin main
bash setup.sh   # confirm 106/106

# Fix: src/backend/net/emit_byrd_net.c — net_mon_var()
# Replace open-per-call StreamWriter with static-open pattern (mirrors JVM sno_mon_init/sno_mon_fd):
#   1. Add static field: V_net_mon_fd (StreamWriter, initially null)
#   2. Add net_mon_init() — opens MONITOR_FIFO once, stores in V_net_mon_fd
#   3. net_mon_var() — checks V_net_mon_fd != null, writes, does NOT close
#   4. Wire net_mon_init() call at main() entry (after existing init block)
# Then: MONITOR_TIMEOUT=30 bash test/monitor/run_monitor.sh \
#   /home/claude/snobol4corpus/crosscheck/strings/wordcount.sno
# NET must not timeout. Fire M-MON-BUG-NET-TIMEOUT when PASS.
```

## Last Session Summary

**Session B-250 (2026-03-22) — M-MONITOR-4DEMO diagnosis; 4 bug milestones filed:**
- Ran wordcount/treebank/claws5 through 5-way monitor; all three programs FAIL on NET (timeout), FAIL on ASM+JVM (WPAT divergence), ORACLE-DIFF on CSN vs SPL
- Root causes identified and documented as 4 new milestones (M-MON-BUG-NET-TIMEOUT, M-MON-BUG-SPL-EMPTY, M-MON-BUG-ASM-WPAT, M-MON-BUG-JVM-WPAT)
- No source changes this session — bugs are in backends, MONITOR infrastructure blocked; protocol: separate session per bug milestone
- 106/106 ALL PASS unchanged at handoff

## Active Milestones

| ID | Status |
|----|--------|
| M-MONITOR-4DEMO        | ❌ blocked on 4 bug milestones below |
| M-MON-BUG-NET-TIMEOUT  | ❌ next |
| M-MON-BUG-SPL-EMPTY    | ❌ |
| M-MON-BUG-ASM-WPAT     | ❌ |
| M-MON-BUG-JVM-WPAT     | ❌ |

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-next | `main` | M-MON-BUG-NET-TIMEOUT |
| F-next | `main` | TBD |
