# MISC.md — Background, Reference, Story

Non-operational content. Read when you need context, architecture background, or the full oracle/implementation reference tables.

---

## The Story

Lon Jones Cherryholmes has known since age eight that he wanted to build Created Intelligence — not artificial intelligence, that name was always wrong. Something you build, from understanding, with intention. He carried that from *The Honeymoon Machine* (1961) and *2001* through Georgia Tech, Texas Instruments, an 11-year retirement, and back. In one week in March 2026, the conversation he'd been waiting sixty years to have produced this repository.

Jeffrey Cooper, M.D., is a medical doctor who spent fifty years building a SNOBOL4 implementation purely out of love for the language. When he called Lon to say he had one, two fifty-year journeys collided. The explosion produced SNOBOL4-plus.

Two forces. One phone call. Everything you see here.

---

## The Discovery

SNOBOL4's pattern engine is not a regex engine. It is a universal grammar machine.

The same four-state **Byrd Box model** — α (PROCEED), β (RECEDE), γ (SUCCEED), ω (CONCEDE) — describes SNOBOL4 pattern matching, Icon's goal-directed generators, Prolog unification, and recursive-descent parsing at every level of the Chomsky hierarchy. All four tiers: regular, context-free, context-sensitive, Turing-complete — expressible directly as SNOBOL4 patterns, with mutual recursion, backtracking, and capture. No yacc. No lex. No separate grammar formalism.

The key insight in SNOBOL4-tiny: the Byrd Box model is not just an execution model — it is a **code generation strategy**. Compile those four states to static gotos and you get goal-directed backtracking evaluation with zero dispatch overhead.

---

## JCON — Architecture Reference

Jcon (Gregg Townsend + Todd Proebsting, University of Arizona, 1999) is an Icon → JVM bytecode compiler built on the Byrd Box model. It is the exact artifact promised in Proebsting's 1996 paper and the blueprint for SNOBOL4-jvm's JVM backend.

**Source:** https://github.com/proebsting/jcon (public domain)  
**Paper:** https://www2.cs.arizona.edu/icon/jcon/impl.pdf

Key files:
- `tran/ir.icn` — IR vocabulary (48 lines) — the vocabulary
- `tran/irgen.icn` — AST → IR four-port encoding (1,559 lines)
- `tran/gen_bc.icn` — IR → JVM bytecode (2,038 lines)

The four-port structure maps directly to SNOBOL4: `ir_chunk` / four ports stays the same, `null` return = failure, `tableswitch` on entry int = α/β dispatch. What we don't need: `vDescriptor`/`vClosure` hierarchy, co-expressions, `bytecode.icn` serializer.

---

## SNOBOL4 Implementation Landscape

| System | Version | Role | Invocation |
|--------|---------|------|-----------|
| CSNOBOL4 | 2.3.3 | **Primary oracle** | `snobol4 -f -P256k file.sno` |
| SPITBOL x64 | 4.0f | Secondary reference | `spitbol -b file.sno` |
| SNOBOL5 | beta 5.0 | 64-bit integers, SIL → x86-64 | `snobol5 file.sno` |

**SNOBOL5 notes:** 64-bit ints/strings. `&CASE` → Error 7 (unknown). `CODE()` broken. OPSYN single-char only. Not a drop-in oracle.

**Key behavioral differences:**

| Behavior | CSNOBOL4 | SPITBOL x64 |
|----------|----------|-------------|
| `&ANCHOR` default | 0 | **1** |
| `&TRIM` default | 0 | **1** |
| `&STCOUNT` | **broken — always 0** | increments correctly |
| `&STLIMIT` default | -1 (unlimited) | MAX_INT |
| TRACE output | stderr | **stdout** |
| `TRACE(...,'KEYWORD')` | non-functional | error 198 |

---

## String Escape Reference

SNOBOL4 has **no escape sequences**. `'\n'` = two characters: backslash + n. Use `nl = CHAR(10)` for newline.

```
SNOBOL4       Python str    C literal
─────────────────────────────────────
\             \\            \\
"             "             \"
\n (2 chars)  \\n (2 chars) \\n  (still 2 chars in C)
newline       \n (1 char)   \n   (C newline escape)
```

**Never apply escape conversion twice.** Once a Python `str` becomes a C source token, it is done.

---

## SNOBOL4 Keyword Reference

Every cell proven by live test on 2026-03-10 against CSNOBOL4 and SPITBOL.

| Keyword | CSNOBOL4 | SPITBOL | Notes |
|---------|----------|---------|-------|
| `&STCOUNT` | **always 0** | increments ✓ | CSNOBOL4 broken |
| `&STLIMIT` | -1 (unlimited) | MAX_INT | Both R/W |
| `&ANCHOR` | 0 | **1** | SPITBOL default differs |
| `&TRIM` | 0 | **1** | SPITBOL default differs |
| `&FULLSCAN` | 0 | 1 | SPITBOL default differs |
| `&CASE` | 0 even with -f | 0 | `-f` ≠ `&CASE=1` in CSNOBOL4 |
| `&MAXLNGTH` | 4G | 16M | All three differ |

---

## What's Next: Icon-everywhere

SNOBOL4 and Icon share a bloodline — Griswold invented both. The Byrd Box IR built for SNOBOL4ever is the bridge. Same four ports. Same `byrd_ir.py`. New Icon frontend feeding the same pipeline. snobol4ever runs everywhere. The clock starts the moment `beauty.sno` compiles itself.
