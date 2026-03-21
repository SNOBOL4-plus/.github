# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `asm-backend` B-225 — M-ASM-RUNG10 (4/9 PASS → WIP)
**HEAD:** `284d6cc` B-225
**Milestone:** M-ASM-RUNG10 ❌ (4/9: 1012+1014+1015+1018 PASS)
**Invariants:** 100/106 C (6 pre-existing) · 26/26 ASM

**⚠ CRITICAL NEXT ACTION — Session B-226:**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git pull --rebase origin asm-backend
apt-get install -y libgc-dev nasm && make -C src
cd src/runtime/asm && gcc -g -O0 -c snobol4_asm_harness.c -o snobol4_asm_harness.o && cd /home/claude/snobol4x
CORPUS=/home/claude/snobol4corpus/crosscheck
STOP_ON_FAIL=0 CORPUS=$CORPUS bash test/crosscheck/run_crosscheck.sh    # 100/106
CORPUS=$CORPUS bash test/crosscheck/run_crosscheck_asm.sh                # 26/26
bash test/crosscheck/run_crosscheck_asm_rung.sh $CORPUS/rung10           # 4/9 PASS
```

**Remaining 5 failures — known root causes:**

1. **1013_func_nreturn** — Route NRETURN → `fn_NAME_gamma` (not omega) in `resolve_special_goto` in `emit_byrd_asm.c` (~line 3357). One-line fix: change the NRETURN branch from `fn_NAME_omega` to `fn_NAME_gamma`.

2. **1017_arg_local** — `_b_ARG`/`_b_LOCAL` are now in snobol4.c and registered (B-225). Still need: in `emit_byrd_asm.c` after `PROG_INIT`, iterate all `named_pats[]` with `is_fn==1` and emit `lea rdi, [rel spec_str] ; xor esi, esi ; call DEFINE_fn` for each. Put spec strings in `.rodata` section.

3. **1016_eval** — In `snobol4_pattern.c` `EVAL_fn` (~line 1277): add `DT_P` branch before the `DT_S` check. Serialize pattern node to string then call `_ev_expr`, or call `_ev_expr` on a reconstructed source string.

4. **1010/1011** — APPLY_fn(fn==NULL) → NULVCL. Deferred to B-227 (trampoline complexity).

**Recommended attack order for B-226:**
1. Fix 1013 (one-liner in resolve_special_goto)
2. Fix 1017 (emit DEFINE_fn calls at PROG_INIT)
3. Fix 1016 (DT_P branch in EVAL_fn)
4. Defer 1010/1011 to B-227


## Last Session Summary

**Session B-225 — M-ASM-RUNG10 WIP (4/9, diagnosis + ARG/LOCAL foundation):**
- Diagnosed all 5 remaining rung10 failures with precise root causes.
- Added `_b_ARG` / `_b_LOCAL` to `snobol4.c` (after FNCBLK_t, forward decls + registrations). Builds clean. 100/106 C · 26/26 ASM hold.
- NRETURN: one-line fix in resolve_special_goto (→ gamma not omega).
- EVAL: DT_P branch needed in EVAL_fn.
- 1017: also needs DEFINE_fn emit at PROG_INIT in emitter.
- 1010/1011 deferred to B-227. HEAD `284d6cc`.


## Active Milestones (next 5)

| ID | Status | Notes |
|----|--------|-------|
| M-NET-INDR | ❌ | harness 210_indirect_ref FAIL — Dictionary/stsfld desync from N-209 fix — next NET sprint |
| M-NET-BEAUTY | ❌ | Gates on M-NET-INDR |
| M-ASM-RUNG10 | ❌ | 4/9 WIP — NRETURN/EVAL/ARG/LOCAL remaining — B-226 |
| M-ASM-RUNG11 | ❌ | Gates on RUNG10 |
| M-SC-CORPUS-R2 | ❌ | do_procedure body emission fix (sc_cf.c) — F-210 |

Full milestone history → [PLAN.md](PLAN.md)

---

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-226 | `asm-backend` | M-ASM-RUNG10 (4/9 → 9/9) |
| F-210 | `main` | M-SC-CORPUS-R2 |
| J-210 | `jvm-backend` | M-JVM-BEAUTY |
| N-210 | `net-backend` | M-NET-INDR — fix Dictionary/stsfld desync |
| D-162 | `net-spitbol-switches` | M-NET-SPITBOL-SWITCHES |

Per RULES.md: `git pull --rebase` before every push. Update only your row in PLAN.md NOW table.
