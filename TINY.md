# TINY.md — SNOBOL4-tiny (L2)

C native backend: `sno2c` compiler → C → x86-64.
**Claude Sonnet 4.6 is the author. When any milestone fires, Claude writes the commit.**

→ Frontend detail: [FRONTEND-SNO2C.md](FRONTEND-SNO2C.md) · [FRONTEND-BEAUTY.md](FRONTEND-BEAUTY.md) · [FRONTEND-REBUS.md](FRONTEND-REBUS.md)
→ Backend arch: [BACKEND-C.md](BACKEND-C.md)
→ Testing: [TESTING.md](TESTING.md) · Rules: [RULES.md](RULES.md)

---

## NOW

**Sprint:** `beauty-crosscheck` — Sprint A — rung 12 crosscheck tests
**HEAD:** `08eabba` — clean baseline, 106/106 rungs 1–11
**Milestone:** M-BEAUTY-CORE → M-BEAUTY-FULL

**Next action:**
1. Build beauty_full_bin (commands below)
2. Write `SNOBOL4-corpus/crosscheck/beauty/101_comment.input` + generate `.ref`
3. Write `test/crosscheck/run_beauty.sh`
4. Run → PASS: add 102, 103... / FAIL: probe.py (Paradigm 2)

---

## Session Start

```bash
cd /home/claude/SNOBOL4-tiny
git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"
git log --oneline -3   # verify HEAD matches above

apt-get install -y libgc-dev && make -C src/sno2c

mkdir -p /home/SNOBOL4-corpus
ln -sf /home/claude/SNOBOL4-corpus/crosscheck /home/SNOBOL4-corpus/crosscheck
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh   # must be 106/106
```

## Build beauty_full_bin

```bash
RT=src/runtime
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > beauty_full.c
gcc -O0 -g beauty_full.c \
    $RT/snobol4/snobol4.c $RT/snobol4/mock_includes.c \
    $RT/snobol4/snobol4_pattern.c $RT/mock_engine.c \
    -I$RT/snobol4 -I$RT -Isrc/sno2c -lgc -lm -w -o beauty_full_bin
```

## Session End

```bash
# Artifact check — see FRONTEND-SNO2C.md §Artifact Snapshot Protocol
# Update this file: HEAD, sprint status, next action, pivot log
git add -A && git commit && git push
# Push .github last
```

---

## Milestones

| ID | Trigger | Status |
|----|---------|--------|
| M-SNOC-COMPILES | snoc compiles beauty_core.sno | ✅ |
| M-REBUS | Rebus round-trip diff empty | ✅ `bf86b4b` |
| M-COMPILED-BYRD | sno2c emits Byrd boxes, mock_engine only | ✅ `560c56a` |
| M-CNODE | CNode IR, zero lines >120 chars | ✅ `ac54bd2` |
| **M-BEAUTY-CORE** | beauty_full_bin self-beautifies (mock stubs) | ❌ |
| **M-BEAUTY-FULL** | beauty_full_bin self-beautifies (real -I inc/) | ❌ |
| M-CODE-EVAL | CODE()+EVAL() via TCC → block_fn_t | ❌ |
| M-SNO2C-SNO | sno2c.sno compiled by C sno2c | ❌ |
| M-COMPILED-SELF | Compiled binary self-beautifies | ❌ |
| M-BOOTSTRAP | sno2c_stage1 output = sno2c_stage2 | ❌ |

---

## Sprint Map

### Active → M-BEAUTY-FULL

| Sprint | Paradigm | Trigger | Status |
|--------|----------|---------|--------|
| `beauty-crosscheck` | Crosscheck | beauty/140_self → **M-BEAUTY-CORE** | ⏳ A |
| `beauty-probe` | Probe | All failures diagnosed | ❌ B |
| `beauty-monitor` | Monitor | Trace streams match | ❌ C |
| `beauty-triangulate` | Triangulate | Empty diff → **M-BEAUTY-FULL** | ❌ D |

### Planned → M-BOOTSTRAP

| Sprint | What | Gates on |
|--------|------|----------|
| `trampoline` | block_fn_t loop + hello world | M-BEAUTY-FULL |
| `stmt-fn` | Each stmt → C fn returning S/F addr | M-TRAMPOLINE |
| `block-fn` | Label reachability, group stmts | M-STMT-FN |
| `pattern-block` | Named patterns → block fns | M-BLOCK-FN |
| `code-eval` | CODE()+EVAL() via TCC | M-BEAUTY-FULL |
| `compiler-pattern` | compile(sno) → compiler.sno | M-BEAUTY-FULL |
| `bootstrap-stage1` | sno2c.sno via C sno2c → stage1 | M-SNO2C-SNO |
| `bootstrap-stage2` | stage1 → stage2; diff | M-BOOTSTRAP |

### Completed

| Sprint | Commit |
|--------|--------|
| `space-token` — 0 bison conflicts | `3581830` |
| `compiled-byrd-boxes` — Byrd box emission | `560c56a` |
| `crosscheck-ladder` — 106/106 rungs 1–11 | `668ce4f` |
| `cnode` — CNode IR + pretty-printer | `ac54bd2` |
| `rebus-roundtrip` — Rebus round-trip | `bf86b4b` |
| `smoke-tests` — 21/21 snoCommand | `8f68962` |
| `beauty-runtime` — binary exits cleanly | done |
| `pipeline-green` — 22/22 oracle PASS | `2f98238` |
| `runtime-shim` — snoc_runtime.h + hello world | `6d3d1fa` |
| sprints 0–22 — engine + pipeline foundation | `test/sprint*` |

---

## Pivot Log

| Sessions | What | Why |
|----------|------|-----|
| 80–89 | Attacked beauty.sno directly | Burned chasing bugs needing smaller test cases |
| 89 | Pivot: corpus ladder | Must prove each feature before moving up |
| 95 | Sprint 3 complete — 106/106 | Foundation solid |
| 96–97 | Sprint 4 compiler internals | Retired — not test-driven |
| 97 | Pivot: test-driven only | No compiler work without failing test |
| 98 | HQ refactor, four-paradigm TDD, CSNOBOL4 built | Plan before code |
| 99 | HQ pyramid restructure (L1/L2/L3) | SESSION.md eliminated |
| 100 | Sprint A begins | Rung 12, beauty_full_bin, first crosscheck test |
