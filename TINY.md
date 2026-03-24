# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.**

→ Rules: [RULES.md](RULES.md) · Beauty plan: [BEAUTY.md](BEAUTY.md) · History: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — B-277 (BEAUTY) · F-222 (Prolog) concurrent
**HEAD:** `bd9d6e3` B-277 (snobol4x) / `468c507` B-277 (.github)
**B-session:** M-BEAUTY-TRACE ❌ — next subsystem after omega ✅
**F-session:** M-PROLOG-CORPUS ❌ — rung05 bug root-caused; 16 puzzle stubs committed (M-PZ-03..20)
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — B-277 (M-BEAUTY-TRACE):**

```
NEXT: INC=demo/inc bash test/beauty/run_beauty_subsystem.sh trace

If test/beauty/trace/ does not exist:
  1. Write test/beauty/trace/driver.sno — exercises T8Trace/T8Pos/xTrace
  2. Generate oracle: INC=demo/inc snobol4 -f -P256k -Idemo/inc driver.sno > driver.ref
  3. Write tracepoints.conf with scan-visible DEFINE stubs for T8Trace/T8Pos
  4. Run 3-way monitor — fix any ASM divergences
  5. On PASS: corpus check (106/106), commit B-277: M-BEAUTY-TRACE ✅, push

PATTERN from B-276 (omega):
  - scan-visible DEFINE stubs needed for functions in -INCLUDE'd files
  - tracepoints.conf: INCLUDE ^FnName$ (anchored), EXCLUDE local vars
  - Binary E_ATP (pat @var) now fixed in emit_byrd_asm.c — no related bug expected

After trace: M-BEAUTIFY-BOOTSTRAP sprint begins.
```

---

## Last Two Sessions (3 lines each)

**B-276 (2026-03-24) — M-BEAUTY-OMEGA ✅:**
Found+fixed binary E_ATP (`pat @txOfs`) in emit_byrd_asm.c value-context: was OPSYN dispatch, now LHS passthrough + `stmt_at_capture` side-effect. Wrote 15-test omega driver with scan-visible DEFINE stubs for inject_traces.py. 3-way monitor PASS (13 steps), 106/106 corpus. Commit `151a99b` snobol4x, `.github` `468c507`.

**F-222 (2026-03-23) — puzzle stubs + milestones; no source fix:**
Split puzzles.pro into 16 stub files puzzle_03..20. Added M-PZ-03..20 milestones to PLAN.md ordered easy→hard. Updated FRONTEND-PROLOG.md with full sprint plan and source layout. HEAD `b4507dc`.

---

## Beauty Subsystem Status

See [BEAUTY.md](BEAUTY.md) for full sequence. Summary:
- ✅ 1–18: global/is/FENCE/io/case/assign/match/counter/stack/tree/SR/TDump/Gen/Qize/ReadWrite/XDump/semantic/omega
- ❌ 19: trace ← **now**
