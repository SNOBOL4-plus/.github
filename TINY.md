# TINY.md — snobol4x (L2)

snobol4x: multiple frontends, multiple backends.
**Co-authored by Lon Jones Cherryholmes and Claude Sonnet 4.6.**

→ Rules: [RULES.md](RULES.md) · Beauty plan: [BEAUTY.md](BEAUTY.md) · History: [SESSIONS_ARCHIVE.md](SESSIONS_ARCHIVE.md)

---

## NOW

**Sprint:** `main` — B-288 (E_VART CALL_PAT fix + DATA slot zeroing)
**HEAD:** `358184a` B-288
**B-session:** Three root causes of M-BUG-BOOTSTRAP-PARSE fixed: (1) E_VART fallback LIT_VAR_α → inline CALL_PAT with var_register (box-DATA slots). (2) rpat_t/p/s via var_register → r12 DATA block not flat .bss. (3) Named-pattern α-entry zeroes [r12+16..N] to prevent stale ARBNO depth on scan retry. Remaining: *Parse via REF(Parse) scan-retry still fails — shared static DATA template clobbered between scan attempts because P_Parse_β doesn't zero slots. 106/106 ✅.
**Invariants:** 106/106 ASM corpus ALL PASS ✅

**⚡ CRITICAL NEXT ACTION — B-289:**

Fix remaining *Parse scan-retry failure. Root: P_Parse_α zeroes DATA slots but
P_Parse_β skips zeroing — ARBNO depth left stale when scan retries via β path.
Fix: emit zeroing also at the REF(Parse) β call site in emit_named_ref (before
jmp P_Parse_β). Alternatively implement M-T2-INVOKE (blk_alloc per invocation).
After fix: run 5 reproducer tests → confirm PASS → run beauty bootstrap → diff vs oracle.

**B-286 summary:**
- D-001: SPITBOL is primary compat target (CSNOBOL4 FENCE issue disqualifies it).
- D-002: DATATYPE() = UPPERCASE always. SPITBOL lowercase is an ignore-point.
- D-003: Test suite case-insensitive on DATATYPE output.
- D-004: .NAME = third dialect (DT_N). Matches SPITBOL observable behaviour.
- D-005: Monitor swapped — SPITBOL is participant 0 (primary oracle).
- Single-line fix in emit_byrd_asm.c (arg staging always -32): resolves M-BUG-IS-DIALECT,
  M-BUG-SEMANTIC-NTYPE, M-BUG-TDUMP-TLUMP, M-BUG-GEN-BUFFER. 19/19 beauty PASS.
- DECISIONS.md created in .github.
C backend: ☠️ DEAD — removed from matrix. 99/106, sno2c fails on word*/pat_alt_commit. Not maintained.

---

## Last Two Sessions (3 lines each)

**B-283 (2026-03-24) — M-BEAUTY-MATCH ✅ + all 19 subsystems ✅; bootstrap ARBNO bug found; 106/106:**
(1) Rewrote match driver (TxInList→pattern-based); fixed FAIL α-port missing jmp ω; added nPush/nInc/nPop/nTop C wrappers in mock_includes.c.
(2) All 19 subsystems now PASS 3-way monitor. M-BEAUTY-MATCH ✅ (12 steps).
(3) M-BEAUTIFY-BOOTSTRAP: ARBNO(*Command) takes 0 iterations via CALL_PAT_α path — XDSAR("Command") likely gets DT_P with NULL PATND_t. HEAD `23c0261`.

**B-282 (2026-03-24) — 3 bugs fixed; M-BEAUTY-GLOBAL/IS/ASSIGN PASS; 106/106:**
(1) stmt_match_descr FAILDESCR guard; stmt_setup_subject stale subject_len_val; E_NAM DT_N fix.
(2) M-BEAUTY-GLOBAL ✅ M-BEAUTY-IS ✅ M-BEAUTY-ASSIGN ✅. HEAD `c16c575`.
