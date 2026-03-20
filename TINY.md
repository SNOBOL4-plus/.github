# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.** When any milestone fires, Claude writes the commit.

→ Frontends: [FRONTEND-SNOBOL4.md](FRONTEND-SNOBOL4.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md) · [FRONTEND-SNOCONE.md](FRONTEND-SNOCONE.md) · [FRONTEND-ICON.md](FRONTEND-ICON.md) · [FRONTEND-PROLOG.md](FRONTEND-PROLOG.md)
→ Backends: [BACKEND-C.md](BACKEND-C.md) · [BACKEND-X64.md](BACKEND-X64.md) · [BACKEND-NET.md](BACKEND-NET.md) · [BACKEND-JVM.md](BACKEND-JVM.md)
→ Compiler: [IMPL-SNO2C.md](IMPL-SNO2C.md) · Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)
→ Full session history: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `asm-backend` B-220 — M-EMITTER-NAMING: Greek port labels in JVM and NET generated output
**HEAD:** `5999162` B-219
**Milestone:** M-EMITTER-NAMING ⚠ WIP
**Invariants:** 100/106 C (6 pre-existing) · 26/26 ASM

**⚠ CRITICAL NEXT ACTION — Session B-220:**

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git pull --rebase origin asm-backend
apt-get install -y libgc-dev nasm && make -C src
CORPUS=/home/claude/snobol4corpus/crosscheck
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh    # 100/106 (6 pre-existing)
CORPUS=$CORPUS bash test/crosscheck/run_crosscheck_asm.sh # 26/26
```

**Sprint B-220 steps — JVM Greek labels (65 sites in emit_byrd_jvm.c):**

The label naming law: every Byrd port label in generated JVM output must carry a Greek suffix.
Map old suffix → new suffix:

| Old pattern | Byrd port | New pattern |
|---|---|---|
| `Jn%d_lit_ok` | γ (success) | `Jn%d_lit_γ` |
| `Jn%d_seq_mid` | γ of left = α of right | `Jn%d_seq_γl` |
| `Jn%d_alt_right` | ω of left = α of right | `Jn%d_alt_β` |
| `Jn%d_alt_rst` | restore on alt retry | `Jn%d_alt_rst` (keep — internal) |
| `Jn%d_nam_ok` | γ of inner pat | `Jn%d_nam_γ` |
| `Jn%d_dol_ok` | γ of inner pat | `Jn%d_dol_γ` |
| `Jn%d_arb_loop` | β (retry) | `Jn%d_arb_β` |
| `Jn%d_arb_decr` | β increment | `Jn%d_arb_βinc` |
| `Jn%d_arb_retry` | β retry | `Jn%d_arb_βr` |
| `Jn%d_arb_commit` | γ after commit | `Jn%d_arb_γ` |
| `Jpat%d_success` | statement γ | `Jpat%d_γ` |
| `Jpat%d_fail` | statement ω | `Jpat%d_ω` |
| `Jpat%d_retry` | scan β | `Jpat%d_β` |
| `Jpat%d_tok` / `Jpat%d_tfail` | tree γ/ω | `Jpat%d_tγ` / `Jpat%d_tω` |
| `Jfn%d_return` / `Jfn%d_freturn` | fn γ/ω | `Jfn%d_γ` / `Jfn%d_ω` |

**Sprint B-221 steps — NET Greek labels (22 sites in emit_byrd_net.c):**

Same mapping applied to `Nn%d_*` and `Npat%d_*` prefixes:
- `Nn%d_nam_ok` → `Nn%d_nam_γ`
- `Nn%d_dol_ok` → `Nn%d_dol_γ`
- `Nn%d_arb_loop` → `Nn%d_arb_β`
- `Nn%d_arb_done` → `Nn%d_arb_γ`
- `Npat%d_tok` → `Npat%d_γ`
- `Npat%d_fail` → `Npat%d_ω`
- `Npat%d_retry` → `Npat%d_β`
- `Nfn%d_return` → `Nfn%d_γ`, `Nfn%d_freturn` → `Nfn%d_ω`

**Milestone fires when:** `sno2c -jvm` and `sno2c -net` output contains `_α`/`_γ`/`_ω` Byrd port labels AND invariants hold.

**DO NOT mark M-EMITTER-NAMING ✅ until both JVM and NET generate Greek labels.**

---

## Last Session Summary

**Session B-219 — M-EMITTER-NAMING complete: C backend merged into emit_byrd_c.c:**
- Merged `emit.c` + `emit_byrd.c` into single `emit_byrd_c.c` — now peers with `emit_byrd_asm.c`, `emit_byrd_jvm.c`, `emit_byrd_net.c`.
- All four backends now in one file each with canonical names: `var_register()`, `collect_vars()`, `collect_fndefs()`, `next_uid()`, `escape_string()`, `emit_stmt()`, `emit_pat_node()`, `NamedPat`, `FnDef`, `DataType`, `vars[]`, `nvar`.
- Removed all `byrd_emit_*` / `byrd_cond_*` externs — now static internals.
- `B()` aliased to `C()` for pattern emitter heritage; `ARG_MAX` aliased to `FN_ARGMAX`.
- Clean build. 100/106 C (6 pre-existing, unchanged) + 26/26 ASM hold. HEAD `5999162`.


## Last Two Session Summaries

**Session B-216 — M-EMITTER-NAMING source naming complete across all four backends:**
- Full prefix strip: all `asm_`, `jvm_`, `net_`, `byrd_` private prefixes removed from all four emitter files. Only extern-visible entry points (`asm_emit`, `jvm_emit`, `net_emit`, `byrd_emit_*`) retain prefixes.
- Concept-class rename pass: `current_fn→cur_fn`, `out_col→col`, `MAX_BSS→MAX_VARS`, `JVM/NET_NAMED_PAT_MAX→NAMED_PAT_MAX`, all name-buffer constants→`NAME_LEN`, `ucall_uid→call_uid`, `extra_bss→extra_slots`, `ucall_bss_slots→call_slots`, `prog_strs→str_table/StrEntry`, `prog_flts→flt_table/FltEntry`, `prog_labels→label_table`, `MAX_PROG_*→MAX_*`, `ASM_NAMED_MAXPARAMS→MAX_PARAMS`.
- Duplicate `safe_name` definition removed (dead code from rename).
- 106/106 C + 26/26 ASM held throughout. HEAD `646e7dd`.
- M-EMITTER-NAMING remains ⚠ WIP: generated output Greek port labels not yet done.

**Session B-215 — Segfault fixed; C backend renamed; M-EMITTER-NAMING still ❌:**
- Segfault root cause: triple-push bug in cap-var tree-walk (`emit_byrd_asm.c` ~line 4004) — unguarded `e->children[0]` on leaf nodes. Fix: removed redundant explicit pushes, kept n-ary loop only.
- All three artifacts (beauty/roman/wordcount) regenerated and assemble clean. Committed `6f96ff7`.
- C backend rename complete: `snoc_emit→c_emit`, `sym_table→vars`, `sym_count→nvar`, `E()→C()`. Committed `fd09e01`.
- **Audit at session end revealed M-EMITTER-NAMING is NOT complete**: ASM/NET/JVM static internals still carry per-backend prefixes. PLAN.md corrected.

## Active Milestones (next 5)

| ID | Status | Notes |
|----|--------|-------|
| M-ASM-RUNG11 | ❌ 2/7 | ITEM lvalue emitter fix + PROTOTYPE/VALUE verify — B-212 |
| M-ASM-LIBRARY | ❌ | Gates on RUNG11 |
| M-SC-CORPUS-R2 | ❌ | do_procedure body emission fix (sc_cf.c) — F-211 |
| M-JVM-CROSSCHECK | ❌ | 89/92 (J-208 progress) |
| M-NET-R1 | ❌ | 74/82 NET — ARB backtrack SEQ-omega bug (N-205 WIP) |

Full milestone history → [PLAN.md](PLAN.md)

---

## Concurrent Sessions

| Session | Branch | Focus |
|---------|--------|-------|
| B-212 | `asm-backend` | M-ASM-RUNG11 |
| F-210 | `main` | M-SC-CORPUS-R2 |
| J-208 | `jvm-backend` | M-JVM-CROSSCHECK (89/92) |
| N-205 | `net-backend` | M-NET-R1 — fix ARB SEQ-omega ptr bug → word1-4/cross |
| D-156 | `net-perf-analysis` | M-NET-PERF |

Per RULES.md: `git pull --rebase` before every push. Update only your row in PLAN.md NOW table.
