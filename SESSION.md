# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `pattern-block` (sprint 4/9 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `613b333 — artifact: beauty_tramp_session64.c — 26769 lines, 0 gcc errors, 9 match_pattern_at, Parse Error active` |

---

## State at handoff (session 64)

Commits this session:
- `09e5a5d` — fix(emit_byrd): E_DEREF right-child varname + unary $'lit' output capture + sideeffect emit_imm (33→9 match_pattern_at)
- `5e90712` — fix(emit_byrd): sync C static on do_assign + byrd_cs() helper (nl/tab/etc now visible to get())
- `613b333` — artifact: beauty_tramp_session64.c

**Progress this session:**
- `nPush() $'('` infinite recursion fixed — is_sideeffect_call() + emit_sideeffect_call_inline()
- E_DEREF varname extraction fixed — check right child first (grammar: left=NULL, right=E_VAR)
- Unary `$'lit'` output capture fixed — was falling to var_get("") fallback
- C static sync fixed — byrd_cs() + do_assign now emits both var_set() AND _name=val
- match_pattern_at calls: 33 → 9 (all 9 remaining are legitimate fallbacks)
- Binary compiles: 0 gcc errors throughout
- Symptom: `printf 'X = 1\n' | beauty_tramp_bin` → "Parse Error" + passthrough (no crash)

---

## Root cause of current failure — pat_Parse failing on "X = 1\n"

**Symptom:** `Src POS(0) *Parse *Space RPOS(0)` fails → mainErr1 → "Parse Error".

**Most likely cause — Hypothesis A: Src is empty when stmt_427 fires**

The main loop for a non-continuation line:
```
main01:  read Src (gets "" initially)
         Line POS(0) ANY('.+')  :S(main02)   ← fails for "X = 1" → falls through
         Line POS(0) ANY('.+')  :S(main02)   ← stmt_426, also fails → falls through
         Src POS(0) *Parse ...               ← Src is STILL "" here!
```
`main02:` sets `Src = Src Line nl` — but `main02` only fires on a continuation line
(`:S(main02)` from the ANY('.+') check). For a plain line, Src is never built.
There should be a statement `Src = Line nl` on the non-continuation path before *Parse.
Read beauty.sno lines 783–792 carefully — find the missing Src = Line nl assignment.

**Hypothesis B (if A is wrong):** ARBNO/Reduce wiring in pat_Parse — verify zero-match ARBNO path.

---

## ONE NEXT ACTION

```bash
# 1. Read the exact beauty.sno main loop lines:
sed -n '783,795p' /home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno

# 2. Add debug print to generated C to confirm Src content at stmt_427:
#    After "SnoVal _s2209 = get(_Src);" in /tmp/beauty_tramp.c add:
#      fprintf(stderr, "DEBUG: Src='%s'\n", to_str(_s2209) ? to_str(_s2209) : "(null)");
#    Recompile + run: printf 'X = 1\n' | /tmp/beauty_tramp_bin

# 3. If Src is "" → find missing Src = Line nl statement in emit output
#    If Src is correct → add debug print inside pat_Parse to find failure point

# 4. Fix and test:
printf 'X = 1\n' | /tmp/beauty_tramp_bin
# Goal: no "Parse Error", output is beautified 'X = 1'
```

---

## Artifact convention (mandatory every session touching sno2c/emit*.c)

```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
N=65   # increment from last session
mkdir -p artifacts/trampoline_session$N
./src/sno2c/sno2c -trampoline -I$INC $BEAUTY > artifacts/trampoline_session$N/beauty_tramp_session$N.c
md5sum artifacts/trampoline_session$N/beauty_tramp_session$N.c
wc -l  artifacts/trampoline_session$N/beauty_tramp_session$N.c
# Write artifacts/trampoline_session$N/README.md
# git add artifacts/ && git commit -m "artifact: beauty_tramp_session$N.c — <status>"
```

---

## Container Setup (fresh session)

```bash
apt-get install -y m4 libgc-dev valgrind
TOKEN=TOKEN_SEE_LON
git clone https://LCherryholmes:$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
git clone https://LCherryholmes:$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
git clone https://LCherryholmes:$TOKEN@github.com/SNOBOL4-plus/.github.git snobol4-plus-github
cd SNOBOL4-tiny && git config user.name "LCherryholmes" && git config user.email "lcherryh@yahoo.com"

# CSNOBOL4 — upload snobol4-2_3_3_tar.gz and:
tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
./configure --prefix=/usr/local && make -j$(nproc) && make install
cd ..

cd SNOBOL4-tiny/src/sno2c && make
```

---

## CRITICAL Rules (no exceptions)

- **NEVER write the token into any file**
- **NEVER link engine.c in beauty_full_bin** — engine_stub.c only
- Read PLAN.md fully before coding

---

## Build command (session 64 baseline)

```bash
RT=/home/claude/SNOBOL4-tiny/src/runtime
SNO2C=/home/claude/SNOBOL4-tiny/src/sno2c
gcc -O0 -g -I$SNO2C -I$RT -I$RT/snobol4 \
    /tmp/beauty_tramp.c \
    $RT/snobol4/snobol4.c $RT/snobol4/snobol4_inc.c \
    $RT/snobol4/snobol4_pattern.c $RT/engine.c \
    -lgc -lm -w -o /tmp/beauty_tramp_bin
```
Note: engine.c + snobol4_pattern.c still linked for bch/qqdlm/IDENT/upr dynamic fallbacks.

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-14 | PIVOT: block-fn + trampoline model | complete rethink with Lon |
| 2026-03-14 | M-TRAMPOLINE fired `fb4915e` | trampoline.h + 3 POC files |
| 2026-03-14 | M-STMT-FN fired `4a6db69` | trampoline emitter in sno2c, beauty 0 gcc errors |
| 2026-03-14 | block grouping bug fixed `98ec305` | first_block flag |
| 2026-03-14 | pattern-block sprint `373d939` | 112 named pat fns, 0 gcc errors |
| 2026-03-14 | E_COND/E_IMM E_STR fix `6d09bfa` | binary compiles, runs, fails on static re-entrancy |
| 2026-03-14 | beauty.sno snoXXX→XXX `d504d80` + beautifier bootstrap | oracle now self-referential |
| 2026-03-14 | S4_expression.sno→expression.sno `596cc5f` | same rename + jcooper paths fixed |
| 2026-03-14 | Technique 1 struct-passing `a3ea9ef` | re-entrancy fixed, 0 gcc errors, X=1 passes |
| 2026-03-14 | emit_imm var_set fix `dc8ad4b` | bare labels pass (START works) |
| 2026-03-14 | Greek watermark α/β/γ/ω `f74a384` | Lon's branding in all emitted labels |
| 2026-03-14 | Three-column pretty layout `e00f851` | Lon's watermark layout |
| 2026-03-14 | Binary ~ fix + wrap fix `06f4715` | START clean, X=1 segfaults (engine stack overflow) |
| 2026-03-14 | Compile named pats from fn bodies + E_IMM fix `6467ff2` | 196 compiled pats, new crash: pat_Expr infinite recursion |
| 2026-03-14 | E_DEREF + $'lit' + sideeffect + C-static sync `09e5a5d` `5e90712` | 33→9 match_pattern_at, Parse Error active |
