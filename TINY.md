# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `monitor-scaffold` — build five-way sync-step monitor infrastructure
**HEAD:** `7f44985` B-226 (asm-backend) · `b67d0b1` J-212 (jvm-backend) · `2c417d7` N-209 (net-backend) · `6495074` F-210 (main)
**Milestone:** M-MONITOR-SCAFFOLD (next to fire)
**Invariants:** 100/106 C (6 pre-existing) · 26/26 ASM

**⚠ CRITICAL NEXT ACTION — Session B-227:**

Sprint `monitor-scaffold`. All work in `snobol4x/test/monitor/` on `asm-backend` branch.

```bash
# Setup
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git checkout asm-backend && git pull --rebase origin asm-backend

# Verify invariants before any work
export CORPUS=/home/claude/snobol4corpus/crosscheck
STOP_ON_FAIL=0 CORPUS=$CORPUS bash test/crosscheck/run_crosscheck.sh      # 100/106
CORPUS=$CORPUS bash test/crosscheck/run_crosscheck_asm.sh                  # 26/26

# Build sno2c
apt-get install -y libgc-dev && make -C src/sno2c

# Sprint goal: create these four files
mkdir -p test/monitor
# 1. test/monitor/tracepoints.conf      — default include/exclude/ignore rules
# 2. test/monitor/inject_traces.py      — reads .sno + conf -> instrumented .sno
# 3. test/monitor/run_monitor.sh        — CSNOBOL4 only for M1; 5-way for M2
# 4. test/monitor/normalize_trace.py   — ignore-points + SPITBOL format normalization

# M-MONITOR-SCAFFOLD pass condition:
SNO=/home/claude/snobol4corpus/crosscheck/hello/001_output_string_literal.sno
bash test/monitor/run_monitor.sh $SNO
# Must exit 0, CSNOBOL4 trace stream non-empty
```

**Step-by-step:**
1. Write `tracepoints.conf` — INCLUDE `*` functions, INCLUDE OUTPUT, EXCLUDE &RANDOM/&TIME/&DATE; IGNORE &TERMINAL tty pattern, IGNORE DATATYPE case
2. Write `inject_traces.py` — scan DEFINE( for functions, scan LHS `=` for variables, prepend TRACE() calls
3. Write `run_monitor.sh` — inject → run CSNOBOL4 → capture stderr → report PASS/FAIL
4. Test on `001_output_string_literal.sno` → PASS
5. Commit → M-MONITOR-SCAFFOLD fires → update PLAN.md dashboard
6. Continue to Sprint M2: add SPITBOL + 3 backends + `normalize_trace.py`

**Full sprint plan → [MONITOR.md](MONITOR.md)**

## Last Session Summary

**Session (strategize-2, 2026-03-21) — Five-way monitor plan + milestones:**
- Defined five-way sync-step monitor: CSNOBOL4 + SPITBOL + ASM + JVM + NET
- Named trace-points (observe, never stop) and ignore-points (known-diff suppression)
- Defined M-BEAUTIFY-BOOTSTRAP: beauty.sno reads beauty.sno, all backends = oracle = input
- Rewrote MONITOR.md (L3): participants, trace/ignore-point config, 4-sprint plan M1-M4
- Added 4 milestones + 1 dream milestone (M-MONITOR-GUI) to PLAN.md dashboard
- Monitor infrastructure lives in snobol4x/test/monitor/ first, moves to harness later
- No code changes. No invariant runs needed (strategy session only).

## Active Milestones

| ID | Trigger | Status |
|----|---------|--------|
| M-MONITOR-SCAFFOLD | test/monitor/ exists; one test vs CSNOBOL4 passes | ❌ |
| M-MONITOR-5WAY | All 5 participants wired; one test passes all 5 | ❌ |
| M-MONITOR-4DEMO | roman+wordcount+treebank pass all 5 participants | ❌ |
| M-BEAUTIFY-BOOTSTRAP | beauty.sno fixed point on all 3 backends | ❌ |

Full milestone history → [PLAN.md](PLAN.md)

---

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-227 | `asm-backend` | monitor-scaffold — Sprint M1 |
| J-next | `jvm-backend` | TBD |
| N-next | `net-backend` | TBD |
| F-next | `main` | TBD |

Per RULES.md: `git pull --rebase` before every push. Update only your row in PLAN.md NOW table.
