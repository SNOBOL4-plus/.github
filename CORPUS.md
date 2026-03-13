# CORPUS.md — SNOBOL4-corpus

**Repo:** https://github.com/SNOBOL4-plus/SNOBOL4-corpus  
**What it is:** Single source of truth for all `.sno`, `.inc`, and `.spt` files. Shared by all three platforms. No per-platform duplicates.

---

## Current State

**HEAD:** `3673364`  
**Active priority:** Add Rebus oracle `.sno` files to `programs/rebus/` (Step R0 completion)  
**Status:** Stable. Changes only when new test programs or oracle outputs are added.

---

## What Lives Here

```
programs/
  snocone/          Snocone reference: snocone.sc, snocone.snobol4, report.md
  rebus/            Rebus test files: word_count.reb, binary_trees.reb, syntax_exercise.reb
                    ← ADD: oracle .sno outputs for round-trip test (Step R12)
  gimpel/           Gimpel library programs
  shafto/           Shafto AI corpus
  beauty/           beauty.sno + all 19 -INCLUDE helper libraries
  oracle/           Cross-engine oracle suite
```

## How Each Repo Uses This

- **SNOBOL4-jvm:** submodule at `corpus/lon`
- **SNOBOL4-dotnet:** submodule at `corpus`
- **SNOBOL4-tiny:** referenced via `$INC` path for `-INCLUDE` files

## Update Protocol

Never add per-platform files. If a `.sno` file is used by more than one platform, it goes here — not in the platform repo. When adding oracle outputs for Rebus (Step R12), add them here.
