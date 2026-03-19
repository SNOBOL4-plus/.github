# JVM.md — snobol4jvm (L2)

JVM/Clojure backend: SNOBOL4 → JVM bytecode via multi-stage pipeline.

→ Backend reference: [BACKEND-JVM.md](BACKEND-JVM.md)
→ Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)

---

## NOW

**Sprint:** `jvm-backend` J-R2 — corpus ladder rungs 5–7 → M-JVM-R2
**HEAD:** `2b1d6a9` J-202
**Milestone:** M-JVM-LIT ✅ · M-JVM-ASSIGN ✅ · M-JVM-GOTO ✅ · M-JVM-PATTERN ✅ · M-JVM-CAPTURE ✅ J-201 · M-JVM-R1 ✅ J-202

**J-202 — M-JVM-R1: 22/22 rungs 1–4 PASS:**
- EQ/NE/LT/GT/LE/GE builtins: emit dcmpl + branch, return "" on success, null on failure
- Null-RHS failure propagation: OUTPUT/VAR assignments skip store/print when RHS is null
- INPUT-in-arithmetic pre-hoist: read INPUT into local 5 before expression, null-check at stack depth 0; eliminates VerifyError from partial-stack goto
- Rungs 5–7: 26/28 (pre-existing: expr_eval sno2c error, 053_pat_alt_commit)

**⚠ CRITICAL NEXT ACTION — Session J-203 (JVM):**

Sprint J-R2 — rungs 5–7 PASS → M-JVM-R2

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3   # verify HEAD = 2b1d6a9
apt-get install -y libgc-dev nasm default-jdk && make -C src
CORPUS=/home/claude/snobol4corpus/crosscheck
bash test/crosscheck/run_crosscheck_jvm_rung.sh \
  $CORPUS/control $CORPUS/patterns $CORPUS/capture 2>&1 | tail -5
# 26/28 expected — expr_eval and 053 are pre-existing, investigate if any new failures
# Then push to M-JVM-R2 and advance to rungs 8-9
```
---

## Session Start

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3   # verify HEAD = f24fb97
git remote set-url origin https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
apt-get install -y libgc-dev nasm && make -C src
# J3 smoke baseline:
TMPD=$(mktemp -d); JASMIN=src/backend/jvm/jasmin.jar
for t in size_test dupl_test remdr_test goto_s goto_f; do
  ./sno2c -jvm test/jvm_j3/${t}.sno > $TMPD/p.j
  java -jar $JASMIN $TMPD/p.j -d $TMPD/ 2>/dev/null
  cls=$(ls $TMPD/*.class | head -1 | xargs basename | sed 's/.class//')
  echo "$t: $(java -cp $TMPD $cls 2>/dev/null)"; rm -f $TMPD/*.class
done; rm -rf $TMPD
# Then implement J4: LIT/SEQ/ALT/ARBNO Byrd box pattern emitter
```
# Then Sprint J2: assign/ + arith/ rungs
```

---

## Milestones

| ID | Trigger | Status |
|----|---------|--------|
| **M-JVM-EVAL** | Inline EVAL! — arithmetic no longer calls interpreter | ❌ |
| M-JVM-SNOCONE | Snocone self-test: compile snocone.sc, diff oracle | ❌ |
| M-JVM-BOOTSTRAP | snobol4-jvm compiles itself | ❌ |

---

## Sprint Map

### Active → M-JVM-EVAL

| Sprint | What | Status |
|--------|------|--------|
| `jvm-edn-cache` | EDN cache — 22× per-program | ✅ `b30f383` |
| `jvm-transpiler` | SNOBOL4 IR → Clojure loop/case — 3.5–6× | ✅ `4ed6b7e` |
| `jvm-stack-vm` | Flat bytecode stack VM — 2–6× | ✅ `d9e4203` |
| `jvm-bytecode` | ASM .class gen — 7.6× | ✅ `c185893` |
| **`jvm-inline-eval`** | Emit arith/assign/cmp into JVM bytecode | ← active |
| `jvm-pattern-engine` | Compile patterns to Java methods | ❌ |

### → M-JVM-SNOCONE

| Sprint | Status |
|--------|--------|
| `jvm-snocone-corpus` | ✅ `ab5f629` |
| `jvm-snocone-lexer` | ✅ `d1dec27` |
| `jvm-snocone-expr` | ✅ `9cf0af3` |
| `jvm-snocone-control` | ❌ |
| `jvm-snocone-selftest` | ❌ |

---

## Pivot Log

| Date | What | Why |
|------|------|-----|
| 2026-03-12 | `jvm-inline-eval` declared active | Sprint 23D complete, 23E next |
| 2026-03-16 | Pivot from HARNESS `monitor-scaffold` to JVM `jvm-inline-eval` | Polish current M-JVM-EVAL milestone; harness M1 parked |
