# SCRIP_DEMOS.md — The Demo Ladder

Ten self-contained Scrip programs. Same algorithm in SNOBOL4, Icon, Prolog.
Same input, same output, three idioms. Each demo is a **product milestone** —
it fires when all three snobol4ever backends produce the correct output.

**Backends:** snobol4x x64 ASM · snobol4x JVM · snobol4x .NET

Reference interpreters (csnobol4, swipl, icont) were used to establish
`.expected` output only. They are not the product.

---

## The Ladder

| # | File | Algorithm | Key contrast |
|---|------|-----------|--------------| 
| DEMO1 | `hello.md` | Hello World | `OUTPUT =` vs `write()` vs `write/1` |
| DEMO2 | `wordcount.md` | Count words | SPAN patterns vs `!str` generator vs DCG |
| DEMO3 | `roman.md` | Integer → Roman numerals | Table-driven goto vs `suspend` vs arithmetic rules |
| DEMO4 | `palindrome.md` | Is string a palindrome? | `REVERSE` vs subscript walk vs `reverse/2` |
| DEMO5 | `fib.md` | Fibonacci first 10 | Labeled goto vs `suspend` generator vs `fib/2` rule |
| DEMO6 | `sieve.md` | Primes to 50 (Sieve) | ARRAY bitset vs list+every vs trial division |
| DEMO7 | `caesar.md` | ROT13 cipher | `REPLACE` parallel strings vs `map()` vs `maplist` |
| DEMO8 | `sort.md` | Sort 8 integers | Insertion sort vs `isort` vs `msort/2` |
| DEMO9 | `rpn.md` | RPN calculator | Pattern-driven stack vs list-as-stack vs DCG |
| DEMO10 | `anagram.md` | Detect anagrams | SORTCHARS+TABLE vs canonical+table vs `msort+assert` |

---

## Milestone Map

Each milestone fires when `run_demo.sh demoN` passes through all three
**snobol4ever backends** and output matches `.expected`.

```
Phase 1 — reference interpreters (COMPLETE):
  M-SD-DEMO1–10 ✅  csnobol4 + swipl + icont, 30/30

Phase 2 — snobol4ever backends (NEXT):
  M-SD-X64-1    hello      x64 ASM
  M-SD-X64-2    wordcount  x64 ASM
  M-SD-X64-3    roman      x64 ASM
  M-SD-X64-4    palindrome x64 ASM
  M-SD-X64-5    fib        x64 ASM
  M-SD-X64-6    sieve      x64 ASM
  M-SD-X64-7    caesar     x64 ASM
  M-SD-X64-8    sort       x64 ASM
  M-SD-X64-9    rpn        x64 ASM
  M-SD-X64-10   anagram    x64 ASM

  M-SD-JVM-1 … M-SD-JVM-10    JVM backend
  M-SD-NET-1 … M-SD-NET-10    .NET backend
```

**Product demo fires when:** all 30 milestones (10 × 3 backends) green.

---

## §NOW

| Milestone | Status |
|-----------|--------|
| M-SD-DEMO1–10 (reference) | ✅ COMPLETE — 30/30 csnobol4+swipl+icont |
| M-SD-X64-1   | ❌ **NEXT** — wire sno2c -asm into run_demo.sh; run hello through x64 backend |
| M-SD-X64-2–10 | ❌ |
| M-SD-JVM-1–10 | ❌ |
| M-SD-NET-1–10 | ❌ |

---

## Milestone firing condition

Each `M-SD-{BACKEND}-N` fires when:
1. `run_demo.sh demoN` invokes the snobol4ever backend (not reference interpreter)
2. Output matches `demo/scrip/demoN/NAME.expected`
3. Session note added to `SESSIONS_ARCHIVE.md`

## The Philosophy

Same algorithm, three idioms. Not "look, they interoperate" — that comes later.
This is: **the same thought expressed three ways, each beautiful in its own language.**

SNOBOL4: patterns consume input structurally.  
Icon: generators produce output lazily.  
Prolog: rules define truth declaratively.

One algorithm. Three windows into it. Through our compiler.

---

*SCRIP_DEMOS.md — the ladder. Each rung is a green test through snobol4ever.*
