# snobol4ever — HQ

SNOBOL4/SPITBOL compilers targeting JVM, .NET, and native C.
**Team:** Lon Jones Cherryholmes (arch, MSIL), Jeffrey Cooper M.D. (DOTNET), Claude Sonnet 4.6 (TINY co-author, third developer).

---

## ⚡ NOW

Each concurrent session owns exactly one row. Update only your row on every push. `git pull --rebase` before every push — see RULES.md §CONCURRENT SESSIONS.

| Session | Sprint | HEAD | Next milestone |
|---------|--------|------|----------------|
| **⚠ GRAND MASTER REORG** | G-7 — FRONTEND-PROLOG-JVM.md trimmed 12KB→4.6KB | `eb9f2ec` G-7 | M-G0-FREEZE (Lon schedules) |
| **⭐ Scrip Demo** | SD-27: M-SD-3 🔄 roman — 5/6 PASS; ICON-JVM list subscript VerifyError | `dc4070c` SD-27 | M-SD-3 |
| **TINY backend** | B-292 — 106/106 | `acbc71e` B-292 | M-BEAUTIFY-BOOTSTRAP-ASM-MONITOR |
| **TINY NET** | N-248 — 110/110 | `425921a` N-248 | M-T2-FULL |
| **TINY JVM** | J-216 — STLIMIT/STCOUNT ✅ | `a74ccd8` J-216 | M-JVM-STLIMIT-STCOUNT |
| **TINY frontend** | F-223 — see TINY.md | `b4507dc` F-223 | M-PROLOG-CORPUS |
| **DOTNET** | D-164 — 1903/1903 | `e1e4d9e` D-164 | TBD |
| **README** | R-2 | `00846d3` R-2 | M-README-DEEP-SCAN |
| **ICON frontend** | I-11 — rung03 ✅ | `bab5664` I-11 | M-ICON-STRING |
| **Prolog JVM** | PJ-76 — SWI baseline partial | `d6c63ad` PJ-76 | M-PJ-SWI-BASELINE |
| **Icon JVM** | IJ-56 — rung36 0/51; 38 CE | `52e575c` IJ-56 | M-IJ-JCON-HARNESS |

**Invariants:** TINY `106/106` · DOTNET `1903/1903`

---

## Routing: Repo → Frontend → Backend → L4 doc

**Step 1 — Which repo?**

| Repo | What it is |
|------|-----------|
| `snobol4x` | Main compiler: all frontends × x64/JVM/.NET backends |
| `snobol4jvm` | Clojure JVM backend (snobol4 only) |
| `snobol4dotnet` | C# .NET backend (snobol4 only) |

**Step 2 — Which frontend × backend?** → your L4 doc

| Frontend | x64 ASM | JVM | .NET |
|----------|:-------:|:---:|:----:|
| SNOBOL4 | BACKEND-X64.md | BACKEND-JVM.md | BACKEND-NET.md |
| Icon | FRONTEND-ICON.md | FRONTEND-ICON-JVM.md | — |
| Prolog | FRONTEND-PROLOG.md | FRONTEND-PROLOG-JVM.md | — |
| Snocone | FRONTEND-SNOCONE.md | — | — |
| Rebus | FRONTEND-REBUS.md | — | — |

Special sessions: `SCRIP_DEMO.md` (SD), `BEAUTY.md` (beauty sprint), `GRAND_MASTER_REORG.md` (G).

---

*PLAN.md = L1. 3KB max. NOW table + routing only. No sprint content. No completed milestones. Ever.*
*Milestone fires → MILESTONE_ARCHIVE.md. Session details → L4 doc §NOW.*
