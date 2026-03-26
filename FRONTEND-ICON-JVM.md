# FRONTEND-ICON-JVM.md — Tiny-ICON → JVM Backend (L4)

Icon → JVM backend emitter. The Icon **frontend** (lex → parse → AST) is shared
and lives in `src/frontend/icon/`; this sprint is about `icon_emit_jvm.c` — the
**JVM backend emitter** that consumes the `IcnNode*` AST and emits Jasmin `.j` files,
assembled by `jasmin.jar` into `.class` files. Despite the file's location under
`src/frontend/icon/`, the work here is backend emission, not parsing.

**Session trigger phrase:** `"I'm working on Icon JVM"` — also triggered by `"playing with ICON frontend ... with JVM backend"` or any phrasing that combines Icon with JVM.
**Session prefix:** `IJ` (e.g. IJ-1, IJ-2, IJ-3)
**Driver flag:** `icon_driver -jvm foo.icn -o foo.j` → `java -jar jasmin.jar foo.j -d .` → `java FooClass`
**Oracle:** `icon_driver foo.icn -o foo.asm -run` (the x64 ASM backend)

*Session state → this file §NOW. Backend reference → BACKEND-JVM.md.*

---

## §NOW — Session State

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **Icon JVM** | `main` IJ-35 — M-IJ-TABLE 4/5; Bug3 key α re-snapshot | `6e41be2` IJ-35 | M-IJ-TABLE |

### CRITICAL NEXT ACTION (IJ-36)

**Baseline: 114/114 PASS (rungs 01–22). rung23: 4/5 (t01–t04 PASS, t05 FAIL).**

**THE BUG — `key(T)` α re-snapshot:** `every` drives the generator via α (not β) on each iteration → `ktr` re-snapshots keySet and resets kidx=0 → only the first key is yielded repeatedly.

**Fix:** Add `icn_N_kinit I` static. α port: `getstatic kinit; ifne kchk` (skip re-snapshot if already init'd). `ktr` entry: `iconst_1; putstatic kinit` (mark init done on first entry).

```bash
grep -n "ktr\|kchk\|kidx\|karr\|ktbl\|kinit" src/frontend/icon/icon_emit_jvm.c
# Apply fix above → build → rung23 5/5 → total 119/119 → M-IJ-TABLE ✅
# Commit "IJ-36: M-IJ-TABLE ✅ — 119/119 PASS"
# Update PLAN.md NOW, §NOW above, SESSIONS_ARCHIVE.md
```

**Bootstrap IJ-36:**
```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x
gcc -Wall -Wextra -g -O0 -I. src/frontend/icon/icon_driver.c src/frontend/icon/icon_lex.c \
    src/frontend/icon/icon_parse.c src/frontend/icon/icon_ast.c \
    src/frontend/icon/icon_emit.c src/frontend/icon/icon_emit_jvm.c \
    src/frontend/icon/icon_runtime.c -o /tmp/icon_driver
bash test/frontend/icon/run_corpus_jvm.sh /tmp/icon_driver   # expect 114/114
bash test/frontend/icon/run_rung_jvm.sh /tmp/icon_driver 23  # expect 4/5
```

---

## §BUILD
```bash
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x && gcc -Wall -Wextra -g -O0 -I. src/frontend/icon/icon_driver.c \
    src/frontend/icon/icon_lex.c src/frontend/icon/icon_parse.c \
    src/frontend/icon/icon_ast.c src/frontend/icon/icon_emit.c \
    src/frontend/icon/icon_emit_jvm.c src/frontend/icon/icon_runtime.c \
    -o /tmp/icon_driver
```

## §TEST
```bash
# Full corpus:
bash test/frontend/icon/run_corpus_jvm.sh /tmp/icon_driver
# Single rung:
bash test/frontend/icon/run_rung_jvm.sh /tmp/icon_driver 23
```

---

## Key Files

| File | Role |
|------|------|
| `src/frontend/icon/icon_emit_jvm.c` | JVM backend emitter (main work file) |
| `src/frontend/icon/icon_emit.c` | x64 ASM emitter — Byrd-box logic oracle |
| `src/frontend/icon/icon_driver.c` | `-jvm` flag → `ij_emit_file()` branch |
| `src/backend/jvm/jasmin.jar` | Assembler — `java -jar jasmin.jar foo.j -d outdir/` |
| `test/frontend/icon/corpus/` | `.icn` tests; oracle = ASM backend output |

---

## Session Bootstrap (every IJ-session)

```bash
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/snobol4x
git clone https://TOKEN_SEE_LON@github.com/snobol4ever/.github
apt-get install -y default-jdk nasm libgc-dev
cd snobol4x
gcc -Wall -Wextra -g -O0 -I. src/frontend/icon/icon_driver.c src/frontend/icon/icon_lex.c \
    src/frontend/icon/icon_parse.c src/frontend/icon/icon_ast.c \
    src/frontend/icon/icon_emit.c src/frontend/icon/icon_emit_jvm.c \
    src/frontend/icon/icon_runtime.c -o /tmp/icon_driver
# tail -80 SESSIONS_ARCHIVE.md → find last IJ entry
# Read §NOW above → start at CRITICAL NEXT ACTION
```

---

## Milestone Table

| ID | Feature | Status |
|----|---------|--------|
| M-IJ-SCAFFOLD | `-jvm` flag wired, null.icn → null.j assembles | ✅ |
| M-IJ-HELLO | `write("hello")` → JVM output | ✅ |
| M-IJ-ARITH | Integer arithmetic, relops | ✅ |
| M-IJ-STRINGS | String ops, concatenation | ✅ |
| M-IJ-GENERATORS | `every`/`suspend`/`fail`/`return` | ✅ |
| M-IJ-CONTROL | `if`/`while`/`until`/`repeat`/`break`/`next` | ✅ |
| M-IJ-SCAN | `?` scanning, `find`/`match`/`tab`/`move`/`any`/`many`/`upto` | ✅ |
| M-IJ-CSETS | Cset literals, `&ucase`/`&lcase`/`&digits` | ✅ |
| M-IJ-ALTLIM | Alternation `|`, limitation `\` | ✅ |
| M-IJ-BANG | `!E` bang generator over strings | ✅ |
| M-IJ-REAL | Real arithmetic + relops | ✅ |
| M-IJ-SUBSCRIPT | `s[i]` string subscript, `*s` size | ✅ |
| M-IJ-CORPUS-R18 | 94/94 PASS rungs 01–18 | ✅ |
| M-IJ-LISTS | `list`, `push/put/get/pop/pull`, `[a,b,c]`, `!L` | ✅ |
| M-IJ-CORPUS-R22 | 114/114 PASS rungs 01–22 | ✅ |
| **M-IJ-TABLE** | `table`, `t[k]`, `key/insert/delete/member` | ❌ **NEXT** |
| M-IJ-RECORD | `record` decl, `r.field` access | ❌ |
| M-IJ-GLOBAL | `global` vars, `initial` clause | ❌ |
| M-IJ-BUILTINS-STR | `repl/reverse/left/right/center/trim/map/char/ord` | ❌ |
| M-IJ-BUILTINS-TYPE | `type/copy/image/numeric` | ❌ |
| M-IJ-SORT | `sort/sortf` (depends: LISTS+TABLE) | ❌ |
| M-IJ-POW | `^` exponentiation | ❌ |
| M-IJ-CASE | `case E of { ... }` | ❌ |
| M-IJ-READ | `read()`, `reads(n)` | ❌ |
| M-IJ-SCAN-AUGOP | `s ?:= expr` | ❌ |
| M-IJ-COEXPR | `create E`, `@C` co-expressions | 💭 |
| M-IJ-MATH | `atan/sin/cos/exp/log/sqrt` | 💭 |
| M-IJ-MULTIFILE | `link`, multi-file programs | 💭 |

**Sprint order:** TABLE → RECORD → GLOBAL → POW → READ → BUILTINS-STR → BUILTINS-TYPE → SORT → CASE → SCAN-AUGOP → COEXPR → MATH → MULTIFILE.

---

*FRONTEND-ICON-JVM.md = L4. §NOW = ONE bootstrap block only — current session's next action. Prior session findings → SESSIONS_ARCHIVE.md only. Completed milestones → MILESTONE_ARCHIVE.md. Size target: ≤8KB total.*
