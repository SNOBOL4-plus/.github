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

**Sprint B-220 — JVM Greek labels (65 sites in emit_byrd_jvm.c):**

Every Byrd port label in generated JVM output must carry a Greek suffix.

| Old | Port | New |
|---|---|---|
| `Jn%d_lit_ok` | γ | `Jn%d_lit_γ` |
| `Jn%d_seq_mid` | γ→α | `Jn%d_seq_γ` |
| `Jn%d_alt_right` | β | `Jn%d_alt_β` |
| `Jn%d_nam_ok` | γ | `Jn%d_nam_γ` |
| `Jn%d_dol_ok` | γ | `Jn%d_dol_γ` |
| `Jn%d_arb_loop` | β | `Jn%d_arb_β` |
| `Jn%d_arb_decr` | β inc | `Jn%d_arb_βinc` |
| `Jn%d_arb_retry` | β retry | `Jn%d_arb_βr` |
| `Jn%d_arb_commit` | γ | `Jn%d_arb_γ` |
| `Jpat%d_success` | stmt γ | `Jpat%d_γ` |
| `Jpat%d_fail` | stmt ω | `Jpat%d_ω` |
| `Jpat%d_retry` | scan β | `Jpat%d_β` |
| `Jpat%d_tok` / `Jpat%d_tfail` | tree γ/ω | `Jpat%d_tγ` / `Jpat%d_tω` |
| `Jfn%d_return` / `Jfn%d_freturn` | fn γ/ω | `Jfn%d_γ` / `Jfn%d_ω` |

**Sprint B-221 — NET Greek labels (22 sites in emit_byrd_net.c):**

| Old | Port | New |
|---|---|---|
| `Nn%d_nam_ok` | γ | `Nn%d_nam_γ` |
| `Nn%d_dol_ok` | γ | `Nn%d_dol_γ` |
| `Nn%d_arb_loop` | β | `Nn%d_arb_β` |
| `Nn%d_arb_done` | γ | `Nn%d_arb_γ` |
| `Npat%d_tok` | stmt γ | `Npat%d_γ` |
| `Npat%d_fail` | stmt ω | `Npat%d_ω` |
| `Npat%d_retry` | scan β | `Npat%d_β` |
| `Nfn%d_return` / `Nfn%d_freturn` | fn γ/ω | `Nfn%d_γ` / `Nfn%d_ω` |

**Sprint B-222 — Local variable alignment across all four emit_pat_node functions:**

Goal: corresponding locals in corresponding functions have the same names across C/ASM/JVM/NET.

Key divergences found:

| Concept | C | ASM | JVM | NET | Canon |
|---|---|---|---|---|---|
| pattern node param | `pat` | `pat` | `pat` | `pat` | `pat` ✅ |
| subject param | `subj` (string name) | `subj` | `loc_subj` (int slot) | `loc_subj` | `subj` / `loc_subj` by target type ✅ |
| cursor param | `cursor` (string name) | `cursor` | `loc_cursor` | `loc_cursor` | `cursor` / `loc_cursor` by target type ✅ |
| subject len param | `subj_len` | `subj_len_sym` | `loc_len` | `loc_len` | `subj_len` / `loc_len` — ASM uses `subj_len_sym`, needs rename |
| capture slot allocator | (depth-based) | — | `p_cap_local` | `p_next_int` / `p_next_str` | `p_cap_local` — NET needs rename |
| uid in node | `u` (local) | `uid` | `uid` | `uid` | `uid` ✅ JVM/NET; C uses `u` — rename to `uid` |
| literal string | varies | `s` | `s` | `s` | `s` ✅ |
| literal length | varies | `slen` | `slen` | `slen` | `slen` ✅ |
| label buffers | `alpha_lbl`, `beta_lbl` | `alpha`, `beta` | `lbl_ok` etc | `lbl_ok` etc | use Greek: `lbl_α`, `lbl_β`, `lbl_γ`, `lbl_ω` |

parse_proto locals — C and JVM already match (`i`, `buf`, `j`, `k`). ✅

collect_fndefs locals:
- JVM uses `pbuf`, `sname`, `tbuf`, `ti`, `pi`, `fb` — NET uses `pbuf`, `proto`, `el`, `gl`
- Canon: `pbuf`, `proto`, `entry_lbl`, `end_lbl` across all

emit_stmt param:
- C: `(STMT_t *s, const char *fn)` — `fn` = current function name
- ASM: `(STMT_t *stmt)` — uses global `cur_fn`
- JVM: `(STMT_t *s, int stmt_idx)` — `stmt_idx` = uid for label generation
- NET: `(STMT_t *s, const char *next_lbl)` — `next_lbl` = fallthrough label
- These differ because targets differ. Canon param name: `s` for stmt ptr ✅ (ASM uses `stmt` — rename).

**DO NOT mark M-EMITTER-NAMING ✅ until B-220 + B-221 + B-222 all complete.**

---

## Last Session Summary

**Session B-219 — M-EMITTER-NAMING: C backend merged, Greek labels still needed:**
- Merged `emit.c` + `emit_byrd.c` into `emit_byrd_c.c` — all four backends now one file each.
- Canonical source names in place across all four: `var_register()`, `collect_vars()`, `collect_fndefs()`, `next_uid()`, `escape_string()`, `emit_stmt()`, `emit_pat_node()`, `NamedPat`, `FnDef`, `DataType`.
- M-EMITTER-NAMING remains ⚠ WIP: JVM/NET generated labels need α/β/γ/ω (B-220/B-221); local var alignment needed (B-222).
- 100/106 C (6 pre-existing) + 26/26 ASM. HEAD `5999162`.

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
