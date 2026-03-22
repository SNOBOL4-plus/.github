# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md) · Monitor: [MONITOR.md](MONITOR.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — M-MONITOR-4DEMO in progress
**HEAD:** `f7c4143` B-256 (main)
**Milestone:** M-MONITOR-4DEMO — roman + wordcount + treebank PASS all 5; claws5 divergence count documented
**Invariants:** 106/106 ASM corpus ALL PASS ✅ · 110/110 NET corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — Session B-257 (M-MONITOR-4DEMO: diagnose treebank ASM/NET step-0 timeout):**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git remote set-url origin https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x.git
git pull --rebase origin main

# Setup (if fresh container):
bash setup.sh
apt-get install -y mono-complete
gcc -shared -fPIC -O2 -Wall -o test/monitor/monitor_ipc_sync.so test/monitor/monitor_ipc_sync.c
gcc -shared -fPIC -O2 -Wall -o /home/claude/x64/monitor_ipc_spitbol.so /home/claude/x64/monitor_ipc_spitbol.c

# Confirm hello still PASS all 5:
INC=/home/claude/snobol4corpus/programs/inc X64_DIR=/home/claude/x64 \
  MONITOR_TIMEOUT=15 bash test/monitor/run_monitor_sync.sh \
  /home/claude/snobol4corpus/crosscheck/hello/hello.sno

# Diagnose treebank ASM timeout at step 0:
# Manually compile+link+run ASM treebank with small input to check it produces output:
TMP=$(mktemp -d)
RT=src/runtime
./sno2c -asm -I/home/claude/snobol4corpus/programs/inc demo/treebank.sno > $TMP/tb.s
for src in $RT/asm/snobol4_stmt_rt.c $RT/snobol4/snobol4.c \
           $RT/mock/mock_includes.c $RT/snobol4/snobol4_pattern.c \
           $RT/engine/engine.c; do
  gcc -O0 -g -c $src -I$RT/snobol4 -I$RT -I src/frontend/snobol4 -w \
    -o $TMP/$(basename $src .c).o 2>/dev/null
done
nasm -f elf64 -I$RT/asm/ $TMP/tb.s -o $TMP/tb.o 2>/dev/null
gcc -no-pie $TMP/tb.o $TMP/snobol4_stmt_rt.o $TMP/snobol4.o \
  $TMP/mock_includes.o $TMP/snobol4_pattern.o $TMP/engine.o \
  -lgc -lm -o $TMP/tb_asm
echo "(S (NP (DT The) (NN cat)) (VP (VBZ sits)))" | \
  MONITOR_READY_PIPE="" MONITOR_GO_PIPE="" $TMP/tb_asm 2>&1 | head -10

# Then run wordcount monitor (known to work for CSN/SPL):
INC=/home/claude/snobol4corpus/programs/inc X64_DIR=/home/claude/x64 \
  MONITOR_TIMEOUT=30 bash test/monitor/run_monitor_sync.sh demo/wordcount.sno
```

## Last Session Summary

**Session B-256 (2026-03-22) — M-MON-BUG-NET-TIMEOUT ✅:**
- Root cause: mono's `ilasm` rejects `swap` opcode entirely — every real NET program failed to compile
- Fix: `emit_byrd_net.c` — replace `dup`+`stsfld`+`ldstr`+`swap` with `dup`+`stloc V_mon_val`+`stsfld`+`ldstr`+`ldloc V_mon_val`; add `string V_mon_val` to `.locals init` in `main()` and function bodies
- 110/110 NET corpus PASS after fix — no regressions
- `run_monitor_sync.sh` patched: `-I"$INC"` added to all three `sno2c` invocations (needed for -INCLUDE programs like treebank)
- `demo/treebank.input` created (1-line S-expression sample)
- wordcount monitor run: NET now participates; divergence at step 3 — M-MON-BUG-JVM-WPAT and M-MON-BUG-SPL-EMPTY confirmed live
- treebank monitor: ASM and NET still timeout at step 0 even after INC fix — needs diagnosis next session
- Commits: `1e9f361` (swap fix), `f7c4143` (INC + treebank.input)

**Session B-255 (2026-03-22) — M-MONITOR-SYNC ✅:**
- Added trace-registration hash set (64-slot open-addressed, `trace_set[]`) to snobol4.c
- `trace_register/trace_unregister/trace_registered` helpers using djb2 hash
- `comm_var()` now gates on `trace_registered(name)` — only sends events for variables explicitly registered via `TRACE(name,'VALUE')`; pre-init variables (tab/digits/etc.) silently skipped
- `_b_TRACE` builtin: `TRACE(varname,'VALUE')` registers name; other types accepted but no-op
- `_b_STOPTR` builtin: removes name from trace set
- Registered both with `register_fn` (TRACE 1-4 args, STOPTR 1-2 args)
- `monitor_ready` flag retained as secondary pre-init guard
- Result: hello **PASS all 5 sync** (csn/spl/asm/jvm/net agree at every step, 2 steps)
- SPL segfault is known sandbox artifact — harmless, SPITBOL still participates correctly
- mono installed via apt (needed for NET participant in fresh containers)
- 106/106 ALL PASS unchanged

## Active Milestones

| ID | Status |
|----|--------|
| M-MONITOR-SYNC     | ✅ `2652a51` B-255 |
| M-MON-BUG-NET-TIMEOUT | ✅ `1e9f361` B-256 |
| M-MONITOR-4DEMO    | ❌ **NEXT** — fix treebank ASM/NET step-0 timeout; then all 5 PASS on wordcount+treebank+claws5 |
| M-MON-BUG-SPL-EMPTY   | ❌ |
| M-MON-BUG-ASM-WPAT    | ❌ |
| M-MON-BUG-JVM-WPAT    | ❌ |

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-next | `main` | M-MONITOR-SYNC finish |
| F-212  | `main` | M-PROLOG-TERM — term.h + pl_unify.c + pl_atom.c (Sprint 1) |
