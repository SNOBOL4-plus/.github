# JVM.md — snobol4jvm (L2)

JVM/Clojure backend: SNOBOL4 → JVM bytecode via multi-stage pipeline.

→ Backend reference: [BACKEND-JVM.md](BACKEND-JVM.md)
→ Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)

---

## NOW

**Sprint:** `jvm-backend` J2 — variable assign + arithmetic → M-JVM-ASSIGN
**HEAD:** `b430ceb` session194 (session195 commit pending push)
**Milestone:** M-JVM-LIT ✅ session195

**Next action:** Sprint J2 — assign to variables, read variables, full arith corpus rung (assign/ + arith/)

**Next action:** Implement inline EVAL! in `jvm_codegen.clj` — emit arithmetic/assign/cmp
directly into JVM bytecode instead of calling back into the interpreter.

---

## Session Start

```bash
cd /home/claude/snobol4x
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3   # verify HEAD = session195 commit
# Push if needed (token required):
git remote set-url origin https://TOKEN@github.com/snobol4ever/snobol4x
git push origin main
# Build + invariant:
apt-get install -y libgc-dev nasm && make -C src
ln -sfn /home/claude/snobol4corpus /home/snobol4corpus
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh   # 106/106
# JVM rung check:
bash test/crosscheck/run_crosscheck_jvm_rung.sh \
  /home/claude/snobol4corpus/crosscheck/hello \
  /home/claude/snobol4corpus/crosscheck/output  # 11/11
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
