# RULES.md — Mandatory Rules (no exceptions)

Violations are disruptive. Every rule here was created because something went wrong.

---

## ⛔ TOKEN — Never write the token into any file

The GitHub PAT was committed to SESSION.md on 2026-03-13. GitHub push protection
blocked the push. History had to be rewritten. **Never again.**

- Token lives in Lon's memory only. Provided at session start. Used in-memory only.
- Write `TOKEN=TOKEN_SEE_LON` as placeholder in any file that needs to reference it.
- If token appears in a commit: rotate immediately at https://github.com/settings/tokens

## ⛔ GIT IDENTITY — Every commit in every repo

```bash
git config user.name "LCherryholmes"
git config user.email "lcherryh@yahoo.com"
```
Run immediately after every clone, before any commit. No exceptions.

## ⛔ BYRD BOXES — mock_engine.c only, no interpreter

Every pattern in beauty_full_bin is a compiled Byrd box.
`mock_engine.c` is the only engine file linked. `engine.c` is superseded.
If a build uses engine.c, something is wrong — stop and diagnose.

## ⛔ ARTIFACTS — Snapshot generated C every session

At end of every session that touches sno2c or emit*.c or runtime/:
```bash
INC=/home/claude/SNOBOL4-corpus/programs/inc
BEAUTY=/home/claude/SNOBOL4-corpus/programs/beauty/beauty.sno
src/sno2c/sno2c -trampoline -I$INC $BEAUTY > /tmp/beauty_tramp_candidate.c
LAST=$(ls artifacts/beauty_tramp_session*.c 2>/dev/null | sort -V | tail -1)
# If md5 differs: cp /tmp/beauty_tramp_candidate.c artifacts/beauty_tramp_sessionN.c
# If same md5: update artifacts/README.md with "no change" note only
```
README.md must record: session N, date, md5, line count, compile status, active bug.

## ⛔ TEST INVARIANT — 106/106 rungs 1–11 before any work

```bash
STOP_ON_FAIL=0 bash test/crosscheck/run_crosscheck.sh
```
If not 106/106: fix before touching anything else. Regressions are bugs.

## ⛔ PLAN.md — 4096 bytes max, index only

PLAN.md is the top-level index. It points to downstream files.
When Lon says "update HQ" or "update the plan": update the downstream file
(TINY.md, TESTING.md, ARCH.md, etc.), not PLAN.md.
PLAN.md gets edited only when: org-level milestones change, active repo/sprint changes,
or the file index needs a new entry.

## Session Lifecycle

**Start:** Read SESSION.md. `git log --oneline -3`. Verify SESSION.md HEAD = git HEAD.
If stale: read SESSIONS_ARCHIVE.md recent entries before touching code.

**End:** Artifact check → update artifacts/README.md → update SESSION.md (all 4 fields)
→ update active repo MD (TINY.md etc.) → `git add -A && git commit && git push` all repos
→ push .github last.

**SNAPSHOT:** `git add -A && git commit -m "WIP: <what>" && git push` every touched repo.

**HANDOFF:** SNAPSHOT first → update SESSION.md → update repo MD → push .github.

**EMERGENCY:** `git add -A && git commit -m "EMERGENCY WIP: <state>"` every repo →
push all → one line in SESSION.md Pivot Log.
