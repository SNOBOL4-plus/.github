# HARNESS.md — SNOBOL4-harness

**Repo:** https://github.com/SNOBOL4-plus/SNOBOL4-harness  
**What it is:** Double-trace monitor, cross-engine oracle harness, benchmark pipeline.

---

## Current State

**Active priority:** Stable. Used as diagnostic tool when debugging SNOBOL4-tiny.

---

## The Double-Trace Monitor

Run the oracle interpreter and the compiled binary side by side, emit the same event stream from both, compare event by event. First divergence = root cause. Not a symptom — the actual bug, identified automatically.

**Oracle event stream** (from `beauty.sno` with TRACE hooks):
```snobol4
        TRACE('snoLine','VALUE')
        TRACE('snoSrc','VALUE')
```
CSNOBOL4 → stderr. SPITBOL → stdout. Monitor separates these.

**TRACE gotcha:** `TRACE(...,'KEYWORD')` is non-functional on both CSNOBOL4 and SPITBOL. Use VALUE trace on a probe variable. `&STCOUNT` broken in CSNOBOL4 (always 0) — use literal `&STLIMIT` values for binary search.

## Oracle Hierarchy

| Oracle | Role | Invocation |
|--------|------|-----------|
| CSNOBOL4 2.3.3 | **Primary** — `beauty.sno` reference | `snobol4 -f -P256k -I $INC file.sno` |
| SPITBOL x64 4.0f | Secondary reference | `spitbol -b file.sno` |

**SPITBOL disqualified for beauty.sno** — error 021 at END (indirect function call semantic difference).

## Install (if not present)

```bash
# CSNOBOL4
./configure && make && make install   # → /usr/local/bin/snobol4

# SPITBOL x64
apt-get install nasm
git clone https://github.com/spitbol/x64 spitbol-x64
cd spitbol-x64 && make && cp sbl /usr/local/bin/spitbol
```

## Benchmark Pipeline

Cross-engine grid: SPITBOL / CSNOBOL4 / Interpreter / Transpiler / Stack VM / JVM bytecode.  
Times include ~15ms process-spawn overhead for SPITBOL/CSNOBOL4 — subtract for fair comparison.

**Key results (2026-03-10):**

vs PCRE2 JIT — `(a|b)*abb`: SNOBOL4-tiny **2.3×** faster (33 ns vs 78 ns)  
vs PCRE2 JIT — `(a+)+b` pathological: SNOBOL4-tiny **7–33×** faster (0.7 ns vs 21 ns)  
vs Bison LALR(1) — `{a^n b^n}`: SNOBOL4-tiny **1.6×** faster (44 ns vs 72 ns)
