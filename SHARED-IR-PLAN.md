# SHARED-IR-PLAN.md — Icon + Prolog + SNOBOL4 → 3 Backends via Shared IR

## The Problem

Three frontends. Three backends. Currently wired as:

```
SNOBOL4 → sno2c.h (EKind/EXPR_t/Program*) → emit_byrd_asm.c   (x64)
                                            → emit_byrd_jvm.c   (JVM)
                                            → emit_byrd_net.c   (.NET)

Prolog  → PlClause* → prolog_lower.c → Program* (sno2c EKind)
                                            → prolog_emit.c      (x64, DUPLICATE)
                                            → prolog_emit_jvm.c  (JVM, DUPLICATE)

Icon    → IcnNode* (own AST) → icon_emit.c       (x64, DUPLICATE)
                             → icon_emit_jvm.c    (JVM, DUPLICATE)
                             → (NET: missing entirely)
```

**Prolog is halfway there** — `prolog_lower.c` already produces `Program*` using
`EKind` IR (with Prolog-specific nodes `E_UNIFY`, `E_CLAUSE`, `E_CHOICE`, `E_CUT`,
`E_TRAIL_MARK`, `E_TRAIL_UNWIND` added to sno2c.h). But then it bypasses the shared
backends and feeds its own `prolog_emit.c` / `prolog_emit_jvm.c`.

**Icon is fully isolated** — `IcnNode*` is a parallel AST with no connection to `EKind`.
Both `icon_emit.c` and `icon_emit_jvm.c` are full independent implementations.

**The goal:** All three frontends produce `Program*` → all three backends consume it.
Each new frontend feature automatically works on all three backends. No duplication.

---

## The Target Architecture

```
SNOBOL4 ─────────────────────────────┐
                                      ├──→ Program* (EKind/EXPR_t/STMT_t)
Prolog  → PlClause* → icn_lower.c ───┤         │
                                      │         ├──→ emit_byrd_asm.c   (x64 NASM)
Icon    → IcnNode*  → icn_lower.c ───┘         ├──→ emit_byrd_jvm.c   (JVM Jasmin)
                                                └──→ emit_byrd_net.c   (.NET MSIL)
```

Three new EKind entries cover all Icon-specific constructs. The lowering layer
(`icn_lower.c`) converts `IcnNode*` → `EXPR_t*` using existing and new EKind values.
The three backends add handlers for the new nodes — but only once each.

---

## EKind Extensions Required

Icon constructs that need new EKind values (not already present):

```c
/* ---- Icon frontend (icn_lower.c) ---- */
E_ICN_TO,       /* E1 to E2          — range generator §4.4        */
E_ICN_TO_BY,    /* E1 to E2 by E3    — range generator with step    */
E_ICN_ALT,      /* E1 | E2           — value alternation (NOT E_OR) */
E_ICN_AND,      /* E1 & E2           — conjunction (ir_conjunction)  */
E_ICN_EVERY,    /* every E [do body] — drives generator              */
E_ICN_WHILE,    /* while E [do body]                                 */
E_ICN_UNTIL,    /* until E [do body]                                 */
E_ICN_REPEAT,   /* repeat body                                       */
E_ICN_IF,       /* if E then E2 [else E3] — indirect goto §4.5      */
E_ICN_SUSPEND,  /* suspend E [do body]   — co-routine yield          */
E_ICN_NOT,      /* not E                 — succeed if E fails        */
E_ICN_BANG,     /* !E                    — generate elements         */
E_ICN_LIMIT,    /* E \ N                 — limitation                */
E_ICN_SCAN,     /* E ? body              — string scanning           */
E_ICN_BREAK_LOOP, /* break [E]           — exit enclosing loop       */
E_ICN_NEXT,     /* next                  — restart enclosing loop    */
E_ICN_AUGOP,    /* E1 op:= E2            — augmented assign family   */
```

Existing EKind values that Icon can reuse directly (no new nodes needed):

| Icon construct | Reuses |
|---|---|
| Integer literal | `E_ILIT` |
| Real literal | `E_FLIT` |
| String literal | `E_QLIT` |
| Cset literal | `E_QLIT` (typed) |
| Variable | `E_VART` |
| `+` `-` `*` `/` `%` `^` | `E_ADD/SUB/MPY/DIV/EXPOP` |
| Unary `-` | `E_MNS` |
| `\|\|` concat | `E_CONC` |
| `<` `<=` `>` `>=` `=` `~=` | `E_ADD`-family (numeric relops) |
| `==` `~==` etc. | `E_FNC` (string relop builtins) |
| `f(args)` | `E_FNC` |
| `return E` / `fail` | `E_FNC` (special names) |
| `procedure` | `STMT_t` body |

---

## Incremental Rollout — 5 Phases

### Phase 0 — Foundations (1 session, no behaviour change)
**Goal:** Add Icon EKind entries and write `icn_lower.c` stub.

1. Add `E_ICN_*` values to `sno2c.h` EKind enum (append-only, no existing code breaks)
2. Create `src/frontend/icon/icn_lower.c`:
   - `Program *icn_lower(IcnNode **nodes, int count)` — converts IcnNode tree to Program*
   - Implement literal nodes first: `ICN_INT→E_ILIT`, `ICN_STR→E_QLIT`, `ICN_VAR→E_VART`
   - Stub all other nodes as `E_NULV` with a warning (keeps it compiling)
3. Add `-lower` flag to `icon_driver.c` that runs `icn_lower()` and dumps the Program*
4. **No existing tests change.** `icon_emit.c` / `icon_emit_jvm.c` still used by default.

Milestone: `M-ICN-LOWER-STUB` — driver with `-lower` flag produces Program* for literals.

---

### Phase 1 — x64 backend for Icon Rung 1 via shared IR (2–3 sessions)
**Goal:** `icon_driver -lower-asm` produces correct output for rung01–02 using
`emit_byrd_asm.c` instead of `icon_emit.c`.

1. Implement `icn_lower.c` for Rung 1 nodes:
   - `ICN_ADD/SUB/MUL/DIV/MOD` → `E_ADD/SUB/MPY/DIV` (exact reuse)
   - `ICN_LT/LE/GT/GE/EQ/NE` → `E_FNC("icn_lt"...)` or new `E_ICN_RELOP`
   - `ICN_TO` → `E_ICN_TO` (new node, inline counter pattern §4.4)
   - `ICN_EVERY` → `E_ICN_EVERY`
   - `ICN_CALL(write)` → `E_FNC`
   - `ICN_PROC` → `STMT_t` sequence

2. Add `E_ICN_TO` and `E_ICN_EVERY` handlers to `emit_byrd_asm.c`:
   - `E_ICN_TO`: emit α/β/check labels + counter BSS slot (paper §4.4, verbatim from `ij_emit_to`)
   - `E_ICN_EVERY`: drive generator to exhaustion (verbatim from `ij_emit_every`)

3. Run rung01–02 corpus through `-lower-asm`. Must match oracle exactly.

Milestone: `M-ICN-LOWER-R2-ASM` — rung01+02 pass via shared IR + x64 backend.

---

### Phase 2 — JVM backend for Icon Rung 1 via shared IR (1–2 sessions)
**Goal:** Same `Program*` from Phase 1 runs through `emit_byrd_jvm.c`.

1. Add `E_ICN_TO` and `E_ICN_EVERY` handlers to `emit_byrd_jvm.c` — copy the
   already-working logic from `icon_emit_jvm.c` (these are already debugged).
2. `icon_driver -lower-jvm` → Jasmin → JVM → same output as oracle.
3. Run rung01–02 corpus. Must pass.

Milestone: `M-ICN-LOWER-R2-JVM` — rung01+02 pass via shared IR + JVM backend.

At this point: **one lowerer, two backends, zero duplication for Rung 1.**

---

### Phase 3 — Extend shared IR to Rung 3–10 (4–6 sessions)
**Goal:** All currently-passing Icon rungs (01–10, 54 tests) pass via shared IR.

Each rung adds new `E_ICN_*` nodes to `icn_lower.c` and corresponding handlers
to both backends. Order follows the existing corpus ladder:

| Rung | New nodes | Both backends |
|------|-----------|---------------|
| 3 | `E_ICN_SUSPEND` | Tableswitch resume (already debugged in JVM) |
| 4 | `E_CONC` (reuse), `E_ICN_SCAN` | String ops + scan cursor |
| 5 | `E_ICN_NOT`, `E_ICN_TO_BY` | Not/neg/to-by |
| 6 | Cset via `E_QLIT` tag | any/many/upto builtins |
| 7 | `E_ICN_IF` | Indirect goto gate §4.5 |
| 8 | String builtins (`E_FNC`) | find/match/tab/move |
| 9 | `E_ICN_WHILE/UNTIL/REPEAT` | Loop family |
| 10 | `E_ICN_AUGOP`, `E_ICN_BREAK_LOOP`, `E_ICN_NEXT` | Augop + break/next |

Each rung milestone: `M-ICN-LOWER-RN-ASM` then `M-ICN-LOWER-RN-JVM`.

Strategy: implement in `icn_lower.c` once, then add backend handler twice (ASM + JVM).
The ASM backend handler is mostly a direct transliteration of the x64 code already in
`icon_emit.c`. The JVM handler is a transliteration of the code already in
`icon_emit_jvm.c`. **No new logic — pure mechanical port.**

---

### Phase 4 — .NET backend for Icon (1 session)
**Goal:** All Icon rungs pass via `.NET` backend for free.

Because Phase 3 built all nodes into `Program*`, the .NET backend (`emit_byrd_net.c`)
only needs handlers for the `E_ICN_*` nodes — the lowering is already done.
This is the payoff: the third backend costs one session, not ten.

Milestone: `M-ICN-LOWER-NET` — rung01–10 pass on .NET backend.

---

### Phase 5 — Wire Prolog the same way (1–2 sessions)
**Goal:** `prolog_emit.c` and `prolog_emit_jvm.c` are deleted. Prolog uses the shared backends.

Prolog is already 90% there: `prolog_lower.c` produces `Program*`. The gap is that
`prolog_emit.c` / `prolog_emit_jvm.c` exist as parallel paths that bypass the shared
backends. The fix:

1. Audit which `E_CLAUSE / E_CHOICE / E_UNIFY / E_CUT / E_TRAIL_*` nodes are produced
   by `prolog_lower.c` but not yet handled in `emit_byrd_asm.c` / `emit_byrd_jvm.c`.
2. Add those handlers to the shared backends (already done in the parallel emitters —
   mechanical port, same as Phase 3).
3. Switch Prolog driver to use shared backends. Delete `prolog_emit.c` / `prolog_emit_jvm.c`.
4. Verify all Prolog corpus rungs still pass.

Milestone: `M-PROLOG-LOWER-SHARED` — Prolog uses shared backends, parallel emitters deleted.

---

## Transition Strategy — Old Emitters Kept Until Proven

The old `icon_emit.c` / `icon_emit_jvm.c` are **not deleted** until the new path
passes the full corpus. The driver flag controls which path runs:

```
icon_driver foo.icn -o foo.asm           # old path (icon_emit.c) — default during transition
icon_driver foo.icn -o foo.asm -lower    # new path (icn_lower → emit_byrd_asm)
```

Once `-lower` passes all rungs, flip the default and delete the old emitters.
This means zero regression risk at any phase boundary.

---

## File Changes Summary

| File | Action |
|------|--------|
| `src/frontend/snobol4/sno2c.h` | Add `E_ICN_*` enum values |
| `src/frontend/icon/icn_lower.c` | **CREATE** — IcnNode* → Program* lowering |
| `src/frontend/icon/icn_lower.h` | **CREATE** — `Program *icn_lower(IcnNode**, int)` |
| `src/frontend/icon/icon_driver.c` | Add `-lower`, `-lower-jvm`, `-lower-net` flags |
| `src/backend/x64/emit_byrd_asm.c` | Add `E_ICN_*` case handlers |
| `src/backend/jvm/emit_byrd_jvm.c` | Add `E_ICN_*` case handlers |
| `src/backend/net/emit_byrd_net.c` | Add `E_ICN_*` case handlers (Phase 4) |
| `src/frontend/icon/icon_emit.c` | Keep until Phase 3 complete, then delete |
| `src/frontend/icon/icon_emit_jvm.c` | Keep until Phase 3 complete, then delete |
| `src/frontend/prolog/prolog_emit.c` | Keep until Phase 5 complete, then delete |
| `src/frontend/prolog/prolog_emit_jvm.c` | Keep until Phase 5 complete, then delete |

---

## Effort Estimate

| Phase | Sessions | Deliverable |
|-------|----------|-------------|
| 0 | 1 | `icn_lower.c` stub, literals, driver flag |
| 1 | 2–3 | Rung 1–2 via shared IR on x64 |
| 2 | 1–2 | Rung 1–2 via shared IR on JVM |
| 3 | 4–6 | Rung 3–10 on both backends |
| 4 | 1 | .NET backend for Icon — free ride |
| 5 | 1–2 | Prolog unified, parallel emitters deleted |
| **Total** | **10–15 sessions** | Full 3×3 matrix, zero duplication |

The critical insight: **Phases 3–5 are mechanical ports, not new engineering.**
All the Byrd-box wiring logic for every Icon construct was already worked out
(and debugged) in `icon_emit.c` and `icon_emit_jvm.c`. `icn_lower.c` is a
tree-walk. The backend additions are copy-and-adapt from the existing parallel
emitters. The hard intellectual work is already done.

---

## Why This Wasn't Done From The Start

The Icon IJ-sessions were tactical: build the smallest thing that passes each rung.
`icon_emit_jvm.c` was scaffolded as a direct emitter (mirroring `icon_emit.c`) because
that was the fastest path to a passing test. The shared IR plan existed in
`FRONTEND-ICON.md §Shared IR` but was never executed — each session inherited
the direct-emit approach from the previous one and kept going.

The shared IR plan is now the correct next architectural step.
