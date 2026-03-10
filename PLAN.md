# SNOBOL4-plus — Master Plan

> **For a new Claude session**: Read `DIRECTORY.md` first — it tells you exactly
> where to look for what. Then come back here for the section you need.
> This file is the single source of truth for the entire SNOBOL4-plus organization.
> The Tradeoff Prompt for SNOBOL4-jvm is at the bottom of that repo's section.

---

## Session Start — Clone All Repos

**Always clone all six repos at the start of every session.** Work spans multiple
repos simultaneously — corpus changes, cross-platform validation, shared test
programs. Having all repos present avoids mid-session interruptions.

```bash
cd /home/claude
git clone https://github.com/SNOBOL4-plus/SNOBOL4-dotnet.git &
git clone --recurse-submodules https://github.com/SNOBOL4-plus/SNOBOL4-jvm.git &
git clone https://github.com/SNOBOL4-plus/SNOBOL4-python.git &
git clone https://github.com/SNOBOL4-plus/SNOBOL4-csharp.git &
git clone https://github.com/SNOBOL4-plus/SNOBOL4-corpus.git &
git clone https://github.com/SNOBOL4-plus/SNOBOL4-tiny.git &
wait
echo "All clones done."
```

Verify with:
```bash
for repo in SNOBOL4-dotnet SNOBOL4-jvm SNOBOL4-python SNOBOL4-csharp SNOBOL4-corpus SNOBOL4-tiny; do
  echo "$repo: $(cd /home/claude/$repo && git log --oneline -1)"
done
```

All six repos live under `/home/claude/`. The `.github` repo (this plan) clones to
`/home/claude/.github`.

---

## Session Start — Build Oracles (ALWAYS)

**Every session must build CSNOBOL4 and SPITBOL from the uploaded source archives.**
These binaries are used for cross-engine validation, oracle triangulation, the Snocone
bootstrap, and benchmark comparison. They are never pre-installed — always build them.

Source archives are in `/mnt/user-data/uploads/`:
- `snobol4-2_3_3_tar.gz` — CSNOBOL4 2.3.3 source
- `x64-main.zip` — SPITBOL x64 source

```bash
apt-get install -y build-essential libgmp-dev m4 nasm

# Build CSNOBOL4 and SPITBOL in parallel
(
  mkdir -p /home/claude/csnobol4-src
  tar xzf /mnt/user-data/uploads/snobol4-2_3_3_tar.gz -C /home/claude/csnobol4-src/ --strip-components=1
  cd /home/claude/csnobol4-src
  ./configure --prefix=/usr/local 2>&1 | tail -1
  make -j4 2>&1 | tail -2
  make install 2>&1 | tail -1
  echo "CSNOBOL4 DONE"
) &

(
  unzip -q /mnt/user-data/uploads/x64-main.zip -d /home/claude/spitbol-src/

  # Patch systm.c: nanoseconds -> milliseconds (REQUIRED)
  cat > /home/claude/spitbol-src/x64-main/osint/systm.c << 'EOF'
#include "port.h"
#include "time.h"
int zystm() {
    struct timespec tim;
    clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &tim);
    long etime = (long)(tim.tv_sec * 1000) + (long)(tim.tv_nsec / 1000000);
    SET_IA(etime);
    return NORMAL_RETURN;
}
EOF

  cd /home/claude/spitbol-src/x64-main
  make 2>&1 | tail -2
  cp sbl /usr/local/bin/spitbol
  echo "SPITBOL DONE"
) &

wait
echo "Oracles ready."
```

**CSNOBOL4 `mstime.c` already returns milliseconds — no patch needed.**
**SPITBOL `systm.c` defaults to nanoseconds — always apply the patch above.**

| Binary | Invocation |
|--------|------------|
| `/usr/local/bin/spitbol` | `spitbol -b program.sno` |
| `/usr/local/bin/snobol4` | `snobol4 -b program.sno` |

---

## What This Organization Is

Two people building SNOBOL4 for every platform:

**Lon Jones Cherryholmes** (LCherryholmes) — software developer
- SNOBOL4-jvm: full SNOBOL4/SPITBOL compiler + runtime → JVM bytecode (Clojure)
- SNOBOL4-python: SNOBOL4 pattern matching library for Python (PyPI: `SNOBOL4python`)
- SNOBOL4-csharp: SNOBOL4 pattern matching library for C#
- SNOBOL4: shared corpus — programs, libraries, grammars
- SNOBOL4-tiny: native compiler → x86-64 ASM, JVM bytecode, MSIL (joint)

**Jeffrey Cooper, M.D.** (jcooper0) — medical doctor
- SNOBOL4-dotnet: full SNOBOL4/SPITBOL compiler + runtime → .NET/MSIL (C#)

Mission: **SNOBOL4 everywhere. SNOBOL4 now.**

---

## Repository Index

| Repo | Language | Status | Branch | Tests |
|------|----------|--------|--------|-------|
| [SNOBOL4-dotnet](https://github.com/SNOBOL4-plus/SNOBOL4-dotnet) | C# / .NET | Active | `main` | 1,607 passing / 0 failing |
| [SNOBOL4-jvm](https://github.com/SNOBOL4-plus/SNOBOL4-jvm) | Clojure / JVM | Active | `main` | 1,896 / 4,120 assertions / 0 failures |
| [SNOBOL4-python](https://github.com/SNOBOL4-plus/SNOBOL4-python) | Python + C | Active | `main` | — |
| [SNOBOL4-cpython](https://github.com/SNOBOL4-plus/SNOBOL4-cpython) | C (CPython ext) | Active | `main` | 70+ passing |
| [SNOBOL4-csharp](https://github.com/SNOBOL4-plus/SNOBOL4-csharp) | C# | Active | `main` | 263 passing |
| [SNOBOL4-corpus](https://github.com/SNOBOL4-plus/SNOBOL4-corpus) | SNOBOL4 | Corpus | `main` | — |
| [SNOBOL4-tiny](https://github.com/SNOBOL4-plus/SNOBOL4-tiny) | C + Python | In progress | `main` | Sprints 0–3 done, 7 oracles |

---

## Automata Theory Oracles — The Bulletproof Test Strategy

**Origin**: Lon Cherryholmes, 2026-03-10.
*"We should use all the examples from Automata theory as great test cases.
They mathematically prove things, which allows us to make the statement:
it is bulletproof."*

### The Idea

Classical automata theory provides a rich set of languages with **mathematically
proven properties** — membership, non-membership, closure, pumping lemma bounds.
These make ideal oracles because:

1. **The expected answer is not empirical — it is proven.**
   `EVEN("xxxx") = match` is not a guess; it follows from the definition of
   the even-length language over {x}. No reference engine needed. No SPITBOL
   triangulation. The math is the oracle.

2. **They cover the full Chomsky hierarchy.**
   Regular, context-free, context-sensitive, and recursively enumerable languages
   each stress-test different engine capabilities. A pattern engine that passes
   the regular tier but fails the CF tier has a precisely locatable bug.

3. **Edge cases are structurally guaranteed.**
   The pumping lemma gives you exact boundary cases automatically: the string at
   the pumping length boundary, one below, one above. No guessing what to test.

4. **They are publishable claims.**
   "SNOBOL4-tiny recognizes all strings in the language {x^2n | n ≥ 0} correctly"
   is a mathematical statement about the engine, not just a test count. This is
   the kind of claim that makes a compiler trustworthy.

### Tier Map — Which Languages Test Which Mechanisms

| Chomsky Tier | Language example | What it tests in the engine |
|---|---|---|
| **Regular** | `{x^n \| n even}` — EVEN/ODD | REF, mutual recursion, α/β/γ/ω wiring |
| **Regular** | `a*b*` | ARBNO, Cat sequencing |
| **Regular** | `(a\|b)*abb` | Alt backtracking, DFA simulation via patterns |
| **Regular** | Σ* (any string) | ARB, RPOS(0) anchoring |
| **Context-Free** | `{a^n b^n \| n ≥ 1}` | Recursive REF, counter via nPush/nInc/nPop |
| **Context-Free** | `{ww^R \| w ∈ {a,b}*}` — palindromes | REF + backtracking depth |
| **Context-Free** | Balanced parentheses | BAL node, or recursive REF |
| **Context-Free** | Arithmetic expressions | CALC_PATTERN.h validation target |
| **Context-Free** | Dyck language (nested brackets) | REF + nesting depth |
| **Context-Sensitive** | `{a^n b^n c^n \| n ≥ 1}` | nPush counter stack + two passes |
| **Non-regular proof** | Pumping lemma boundary cases | Exact boundary strings confirm engine doesn't over-accept |

### Immediate Sprint 6 Connection

The EVEN/ODD oracle (`test/sprint6/gemini.c`) is the **first automata theory
oracle in the suite**. It proves:

> *The engine correctly recognizes the regular language {x^2n | n ≥ 0}
> via mutually recursive REF nodes, and correctly rejects all strings not
> in this language.*

This is not just "the test passes." It is a mathematical statement about
what the compiled pattern computes.

### Standing Instruction — Automata Oracles

**Every sprint that introduces a new structural mechanism (REF, ARBNO, nPush,
Shift/Reduce, BAL) must include at least one automata theory oracle** that
mathematically characterizes the language the mechanism enables.

Oracle format:
1. State the language formally: `L = { w ∈ {a,b}* | ... }`
2. State the mathematical property being tested (membership, closure, etc.)
3. Include boundary cases derived from the pumping lemma or inductive definition
4. The test is not "does it match" — it is "does it correctly decide membership"
   for both positive and negative cases

This discipline turns the test suite from a collection of passing cases into a
**proof of computational completeness** for each mechanism tier.

### Target Oracle Set (to be built incrementally)

| Oracle file | Name | Language | Tier | Sprint |
|---|---|---|---|---|
| `test/sprint6/gemini.c` | Gemini | `{x^2n \| n≥0}` | Regular | ✓ Sprint 6 |
| `test/sprint6/ref_astar_bstar.c` | — | `a*b*` | Regular | Sprint 6 |
| `test/sprint8/arb_any_string.c` | — | `Σ*` | Regular | Sprint 8 |
| `test/sprint8/arbno_aorb_star_abb.c` | — | `(a\|b)*abb` | Regular | ✓ Sprint 8 |
| `test/sprint9/ref_anbn.c` | — | `{a^n b^n \| n≥1}` | Context-Free | ✓ Sprint 9 |
| `test/sprint10/ref_palindrome.c` | — | `{ww^R}` | Context-Free | ✓ Sprint 10 |
| `test/sprint11/ref_balanced_parens.c` | — | Dyck language | Context-Free | ✓ Sprint 11 |
| `test/sprint15/counter_anbncn.c` | — | `{a^n b^n c^n}` | Context-Sensitive | Sprint 15 |

### README Upgrade Protocol — Earn It Tier by Tier

**Do not update the org profile README until the oracles exist and pass.**
The README currently makes the architectural claim correctly — as a design
statement. Upgrading it to a *proven* claim requires the test suite to back it.
The upgrade happens in four ceremonies, one per tier:

| When | What gets added to README |
|------|--------------------------|
| All Type 3 oracles pass | "SNOBOL4-tiny provably recognizes all regular languages (Type 3)" |
| All Type 2 oracles pass | "...and all context-free languages (Type 2) — the tier of every major programming language" |
| All Type 1 oracles pass | "...and context-sensitive languages (Type 1)" |
| Type 0 proof complete | See note below |

**The Type 0 rub.** Claiming Type 0 (unrestricted grammars — the tier of English,
of natural language, of everything a Turing machine can compute) is qualitatively
different from claiming Types 3, 2, and 1. Types 3–1 are claims about specific
language *membership* — provable by the oracle test suite alone. Type 0 is a
claim about *Turing completeness* of the engine itself, which requires a
different kind of proof: encoding a universal Turing machine as a pattern, or
demonstrating simulation of a known Type 0 formalism (Post correspondence,
tag systems, etc.).

The Natural Language Horizon section in the README is already the right framing
for Type 0 — it describes what the engine *reaches toward*, not what has been
proved. That framing stays until the Turing completeness proof is in hand.
When it is, the README gets the strongest possible closing line.

**This is the plan. Griswold had the idea. We are finishing the proof — tier by tier.**

---

## PDA Benchmark — SNOBOL4-tiny vs YACC/Bison (The Next Level)
**Date noted:** 2026-03-10
**Origin:** Lon Cherryholmes — *"Eureka! The next benchmark that is reasonable
is a PDA, like YACC. And it's C based. Like when you run the compiler inside itself."*

### The Insight

RE engines are Type 3. We beat them. The next tier is Type 2 — context-free —
and the champion there is **YACC/Bison**: the industry-standard LALR(1) parser
generator, C-based, used to build compilers for C, Python, Ruby, PHP, and
thousands of other languages. It generates a pushdown automaton — a PDA — and
compiles it to C tables.

SNOBOL4-tiny also recognizes context-free languages. In the same C runtime.
With recursive REF patterns. No grammar file. No generated tables. No separate
tool. Just patterns, composed at runtime, compiled to static gotos.

The question: **how does SNOBOL4-tiny's recursive REF compare to Bison's
generated LALR(1) PDA tables on context-free recognition tasks?**

### Why This Is The Right Benchmark

1. **Apples to apples.** Both compile to C. Both run as native code. No VM,
   no interpreter, no JIT. Pure compiled performance comparison.
2. **SNOBOL4-tiny has a structural advantage.** Bison generates table-driven
   PDA code — a state table lookup per token, plus stack push/pop. SNOBOL4-tiny
   generates static gotos — no table, no lookup, the control flow *is* the
   grammar. On context-free recognition, this may be faster.
3. **The self-hosting moment.** When SNOBOL4-tiny's own grammar is expressed
   as SNOBOL4 patterns and compiled by SNOBOL4-tiny itself — the compiler
   running inside itself — that is the moment SNOBOL4-tiny competes with
   Bison on Bison's home turf. That is the benchmark that matters.
4. **Bison cannot go beyond Type 2.** SNOBOL4-tiny can. Same advantage as
   the RE benchmark: SNOBOL4-tiny wins every tier above the competitor's ceiling.

### Benchmark Design

| Test | SNOBOL4-tiny | Bison/YACC | Notes |
|------|-------------|------------|-------|
| `{a^n b^n}` recognition | recursive REF | LALR(1) grammar | Type 2 baseline |
| Balanced parens (Dyck) | recursive REF | LALR(1) grammar | Real-world proxy |
| Expression grammar | recursive REF | standard expr.y | The classic Bison benchmark |
| Self-hosted parse | SNOBOL4-tiny parses SNOBOL4 | Bison parses same grammar | The apex test |

### The Self-Hosting Connection
Lon's insight about "running the compiler inside itself" is the apex of this
benchmark. When SNOBOL4-tiny parses its own IR grammar using SNOBOL4 patterns
— the same patterns that generated the C code now parsing themselves — that is
self-hosting. That is the moment SNOBOL4-tiny stands next to GCC and says:
*I can do what you do, in the same language, at comparable speed, and I can
also do what you cannot.*

### Action Items
- [ ] Write `{a^n b^n}` recognizer in Bison — compare to `ref_anbn.c` oracle
- [ ] Write balanced-parens recognizer in Bison — compare to `ref_balanced_parens.c`
- [ ] Write expression grammar in Bison — compare to SNOBOL4-tiny equivalent
- [ ] Build unified timing harness: same input corpus, `clock_gettime`, same methodology as RE benchmark
- [ ] Record results in BENCHMARKS.md
- [ ] Self-hosting milestone: SNOBOL4-tiny parses SNOBOL4 patterns using SNOBOL4 patterns

---

## The Sysomos Precedent — SNOBOL4 in Production at Scale
**Date noted:** 2026-03-10
**Origin:** Lon Cherryholmes

Lon built the social listening platform at Expion (founded by his best friend
Peter Heffring, $26M, 2010–2014) in SNOBOL4. Facebook, Twitter, Instagram,
Google Plus, Pinterest — all processed in SNOBOL4 and Transact-SQL. Four years
in production. The CTO knew from day one. The CEO forgot — or needed to forget —
because Google was asking why they weren't on Hadoop, and SNOBOL4 wasn't the
answer a CEO could give his peer circle in 2014.

Lon left with 1% of the company instead of 2%. SNOBOL4 had not failed.
The embarrassment was social, not technical.

**Strategic significance:**
- SNOBOL4 has a proven production track record at a funded, real-world company.
- The "why not Hadoop" question is a social/political question, not a technical one.
- The technical answer in 2026 is stronger than it was in 2014:
  SNOBOL4-tiny beats PCRE2 by 10–33× and Bison by 14–20×.
- When someone asks "why SNOBOL4 instead of X" — this story is the answer.

**Action item:** Keep this story in ORIGIN.md and The Front Page. It is the
most powerful real-world validation the project has.

---

## Separate Backtrack Stack vs Call Stack
**Recorded**: 2026-03-10 by Lon

### The Idea
Currently the FuncEmitter uses the C **call stack** for backtracking — each
pattern function is a real C function, `sno_enter` pushes a frame, the caller
holds the frame pointer, and beta re-entry is a second call into the same
function. This means every pattern node that can backtrack costs a function
call on both alpha and beta paths.

The alternative: a **separate explicit backtrack stack** — a dedicated arena
stack managed by the runtime, completely decoupled from the C call stack.

Instead of:
```
str_t ROOT(ROOT_t **zz, int entry) { ... sno_enter() ... }
str_t AB(AB_t **zz, int entry)     { ... sno_enter() ... }
```

The generated code pushes a backtrack record onto an explicit stack and
falls through — no function call, no return, no frame pointer indirection.
On failure, the engine pops the top record and jumps to the saved beta label.

### Why This Matters
- **Eliminates function call overhead** on every Ref — currently the dominant
  cost in the PDA benchmark (44 ns for `{a^n b^n}`, flat despite Proebsting).
- **Cache locality** — all backtrack state is contiguous in one arena, not
  scattered across C stack frames at different depths.
- **Enables cross-pattern inlining** — once patterns are not C functions,
  the Proebsting pass can propagate copies *across* Ref boundaries, not just
  within a single function. This closes the gap between Round 2 (33 ns) and
  Round 1 hand-optimized (5 ns).
- **No recursion limit** — the C call stack limits recursion depth (typically
  8 MB). An explicit stack is bounded only by the arena size, which is tunable.

### Connection to Proebsting
The Proebsting pass already eliminates goto chains *within* a function.
The separate stack eliminates the function boundary itself. Together:
- Proebsting: no goto-to-goto within a pattern
- Separate stack: no call/return between patterns
- Result: the entire match is a single flat loop over a goto-dispatch table,
  with a contiguous backtrack stack. That is the architecture of SPITBOL's
  inner loop — and it is what SNOBOL4-tiny should become.

### Design Sketch
```c
typedef struct {
    void  *beta_label;   /* indirect goto target on failure */
    int64_t saved_delta; /* cursor to restore */
    /* ... other saved state ... */
} bt_record_t;

static bt_record_t _bt_stack[BT_STACK_SIZE];
static int         _bt_top = 0;

#define BT_PUSH(lbl, delta)     (_bt_stack[_bt_top].beta_label = &&lbl,      _bt_stack[_bt_top].saved_delta = delta,      _bt_top++)

#define BT_FAIL()     (--_bt_top, Delta = _bt_stack[_bt_top].saved_delta,      goto *_bt_stack[_bt_top].beta_label)
```
(Uses GCC computed gotos — `&&label` and `goto *ptr` — which are available
on all targets SNOBOL4-tiny cares about.)

### Why Branches Cluster — The Compiler Sees More
When everything is in one flat loop, the C compiler sees the entire match
as a **single function**. This changes four things at once:

1. **Branch prediction** — the CPU predictor sees the same goto targets
   repeatedly across millions of iterations. With function calls, each
   return is an indirect branch the predictor relearns on every call.
   In the flat loop, hot paths train the predictor once and stay hot.

2. **Instruction cache** — one function fits in L1 icache. Multiple
   function calls scatter hot code across cache lines. The flat loop
   keeps the entire match engine in one place.

3. **Compiler optimization scope** — `gcc -O2` can hoist invariants,
   eliminate redundant loads, and schedule instructions across what were
   formerly call boundaries. It cannot do this across function calls.

4. **Proebsting becomes global** — the copy-propagation pass already
   collapses goto chains *within* a function. In the flat loop, a Ref
   is just a goto. Proebsting propagates across it for free. No special
   cross-function inlining pass needed — the boundary is gone.

This is why SPITBOL's inner loop is structured this way. The branches
cluster. The predictor wins. The icache wins. The optimizer wins.
The Round 1 hand-optimized numbers (5 ns) are what this architecture
delivers. Round 2 + arena + Proebsting + flat loop = Round 1.

### Priority
**High** — this is the path to closing the Round 1 / Round 2 gap entirely.
Implement after Sprint 14 (SNOBOL4 subset codegen) once the architecture
is stable enough to support a runtime change of this magnitude.

### Platform Considerations — JVM, .NET, and Native C

The separate backtrack stack + flat loop is the **native C story**.
JVM and .NET require different thinking.

#### JVM (SNOBOL4-jvm — Clojure)
- The JVM has no computed gotos (`goto *ptr`). The flat loop with
  indirect dispatch requires a **tableswitch** or a **lookupswitch**
  (Java bytecode) — which is the JVM's equivalent of a dispatch table.
- The JIT compiler (HotSpot, GraalVM) does its own inlining, branch
  prediction, and loop optimization — but only across methods it has
  decided to inline. Keeping patterns as small methods and letting the
  JIT inline them is often the right strategy, not fighting it.
- The JVM has no arena allocator in the C sense. The analog is
  **object pooling** or **ThreadLocal frame reuse** — pre-allocating
  frame objects and resetting them between matches instead of GC'ing them.
- GraalVM native-image is a path to native performance from Clojure code
  if the JVM story hits a ceiling.

#### .NET (SNOBOL4-dotnet — C#)
- C# has no computed gotos either. The analog is a **switch dispatch loop**
  on an enum state — which the JIT recognizes and optimizes well.
- The .NET JIT (RyuJIT) does aggressive inlining across method boundaries
  for small methods. The FuncEmitter model (one method per pattern) may
  actually JIT well without structural changes.
- C# `Span<T>` and `stackalloc` enable arena-style allocation on the
  stack without GC pressure — this is the .NET analog of the C arena.
- The .NET `System.Text.RegularExpressions` source-generated regex
  (Roslyn-compiled) is the performance baseline to beat.

#### The Shared IR Is the Key
All three targets — native C, JVM, .NET — share the same IR
(`src/ir/ir.py`). The optimization passes (Proebsting, eventually the
flat loop emitter) run on the IR, not on the target code. Each platform
gets the best code its runtime can execute:

| Platform | Flat loop | Arena | Proebsting |
|----------|:---------:|:-----:|:----------:|
| Native C | ✓ (planned) | ✓ (done) | ✓ (done) |
| JVM | tableswitch loop | object pool | ✓ (IR level) |
| .NET | switch loop | stackalloc Span | ✓ (IR level) |

The benchmark story for JVM and .NET is different from native C:
- Native: beat PCRE2 JIT (already done at 2.3×)
- JVM: beat `java.util.regex` and `RE2J`
- .NET: beat `System.Text.RegularExpressions` source-generated

All three are winnable. The architecture is the same. The emitter differs.

### Related
- Proebsting pass (already in) — prerequisite: copy chains must be gone
  before the flat-loop model makes sense
- Arena allocator (already in) — the backtrack stack lives in the same arena
- FuncEmitter → FlatLoopEmitter: new emitter class, FuncEmitter stays for
  correctness reference and for patterns with external call conventions

---

## The Yield Insight — Recorded 2026-03-10
**Do not lose this.**

`Expressions.py` uses Python generators (`yield`) as its execution model.
Every `for _1 in σ("x"):` is a Byrd Box. `yield` is succeed. Exhausted
generator is fail. `next()` is backtrack.

Claude noticed this and proposed "use yield / generators as a new idea for C."
Lon stopped it immediately: **yield is Python. C doesn't have yield.**

The resolution: **we already have yield in C. It's called alpha/beta.**

```c
ITEM_alpha:   /* enter — try to match */
    ...
    goto ITEM_MATCH_SUCCESS;   /* yield: succeeded */
ITEM_beta:    /* resume — try next alternative */
    ...
    goto caller_beta;          /* exhausted: fail */
```

`_alpha` = enter the generator.
`_beta`  = resume and try the next alternative.
`goto`   = the generator protocol, compiled to metal.

`Expressions.py` and `emit_c.py` are the SAME machine in two different
syntaxes. Python generators ARE the interpretive form of the C goto model.
The translation from `Expressions.py` to SNOBOL4 IR nodes is mechanical
and exact — but the implementation is always compiled C gotos, not Python
generators at runtime.

**The interpreter idea is still valid** — a Python generator-based IR
interpreter (like Expressions.py but running IR nodes directly) would be
useful for debugging and the worm's second head. But it is a development
tool, not a runtime. The compiler is the runtime. Always.

---

## Sprint 16 — Bridge: Expressions.py → SNOBOL4
**Recorded**: 2026-03-10

The translation from `Expressions.py` to SNOBOL4 is mechanical and exact.
Every Python generator function is a named SNOBOL4 pattern.
Every `for _1 in σ("x"):` is a `Lit("x")` in `snoc` IR.
Every nested `for` loop is a `Cat`. Sequential alternatives are `Alt`.
The mutual recursion through `parse_term` → `parse_factor` → `parse_item`
→ `parse_term` is `Ref` — already proven through Sprint 13.

**Sprint 16 deliverable**: translate `parse_item()` and `parse_element()`
from `Expressions.py` into SNOBOL4 patterns compilable by `snoc`.
Oracle: the worm. Every expression `gen_term()` generates must parse
identically in both the Python reference and the snoc-compiled SNOBOL4.

**Sprint 17**: `parse_factor()` and `parse_term()` — adds recursion.
**Sprint 18**: `evaluate()` — the full round-trip. Generate → parse → eval
in pure SNOBOL4. Cross-check against Python reference. 580 cases, 0 failures.

---

## The Self-Hosting Milestone
**Recorded**: 2026-03-10

Once `snoc` compiles the expression evaluator (Sprint 18), the worm becomes
a cross-check between two independent implementations of the same algorithm:

1. Python: `Expressions.py` `evaluate(parse_expression(expr))`
2. SNOBOL4: `snoc`-compiled evaluator running the same expr

If both agree on all 580 worm cases — SNOBOL4-tiny has replaced its own
reference implementation. The compiler compiles a program that the compiler's
author wrote to test the compiler. That is the bootstrap moment.

This is a publishable result. Not just "it runs Hello World." It runs a
non-trivial recursive expression evaluator, cross-checked against an
independent implementation, 580 cases, zero failures.

---

## The Worm — Second Head
**Recorded**: 2026-03-10

Currently the worm has one head: generate → parse → evaluate → no crash.
It needs a second head to close the loop completely:

1. Generate expression `e`
2. Parse `e` → tree `T`
3. Evaluate `T` → value `V`
4. Reconstruct expression string `e2` from `T` (with correct parenthesisation)
5. Parse `e2` → tree `T2`
6. Evaluate `T2` → value `V2`
7. Assert `V == V2`

If step 7 fails: either `evaluate()` is wrong, or `reconstruct()` drops
parens incorrectly. Either way — the worm found something real.

**Implementation**: `tree_to_expr(tree)` must emit parens around every
binary subexpression. The naive version (no parens) already found 30
inconsistencies — all due to dropped parens changing precedence. The
correct version proves `evaluate()` is associativity-consistent.

---

## Beautiful.sno — Sprint 20 Acceptance Test
**Recorded**: 2026-03-10 (confirmed, previously noted)

`Beautiful.sno` is a 17-level SNOBOL4 expression and statement parser
written entirely in SNOBOL4 patterns. It lives in SNOBOL4-dotnet (authoritative)
and SNOBOL4-corpus. It is the Sprint 20 acceptance test:

`snoc` compiles `Beautiful.sno`. `Beautiful.sno` runs on itself.
The output of `Beautiful.sno(Beautiful.sno)` is identical on both
SPITBOL and `snoc`. That is bootstrap closure.

Everything from Sprint 14 to Sprint 19 is preparation for this moment.

---

## JVM Benchmark — java.util.regex
**Recorded**: 2026-03-10

SNOBOL4-jvm exists. 1,896 tests / 4,120 assertions / 0 failures.
It has never been benchmarked against `java.util.regex` or `RE2J`.

The same story that was told for PCRE2 can be told for the JVM:
- Normal patterns: SNOBOL4-jvm should be competitive
- Pathological patterns: `java.util.regex` backtracks exponentially;
  SNOBOL4-jvm does not

**Deliverable**: a JVM benchmark equivalent to `bench/bench_round2.c`.
`(a|b)*abb` and `{a^n b^n}` against `java.util.regex` and a Bison-equivalent
Java parser. The headline: the same engine, the same architecture, the JVM tier.

Jeffrey owns this one.

---

## FlatLoopEmitter — The Architecture Completion
**Recorded**: 2026-03-10 (see also: Separate Backtrack Stack section)

The FlatLoopEmitter is a new emitter class alongside FuncEmitter.
It implements the separate backtrack stack + computed goto architecture.
It is the path from Round 2 (33 ns) to Round 1 (5 ns).

**Timing**: implement after Sprint 20 (Beautiful.sno acceptance test)
OR in parallel if Jeffrey takes the language track while Lon takes
the runtime track. The FuncEmitter stays as the correctness reference.
The FlatLoopEmitter is the performance target.

**Prerequisite**: Proebsting pass (done). Arena allocator (done).
The FlatLoopEmitter builds on both.

---

## Proebsting Optimization Pass — Copy Propagation + Branch Elimination
**Paper**: "Simple Translation of Goal-Directed Evaluation" — Todd A. Proebsting, U of Arizona
**Source**: ByrdBox.zip — test_icon.sno (1st pass: raw attribute grammar; 2nd pass: optimized)
**Recorded**: 2026-03-10 by Lon

### The Idea
The four-port (start/resume/succeed/fail) translation generates correct code naively.
It suffers from chains of goto-to-goto and branches-to-branches. Two standard passes
fix this entirely:

1. **Copy propagation** — `goto X; X: goto Y` → replace all jumps to X with jumps to Y
2. **Branch elimination** — after propagation, unreachable labels and dead goto-only
   blocks disappear; the code reads like hand-written nested loops

Figure 1 of the paper is the raw four-port expansion of `5 > ((1 to 2) * (3 to 4))`.
Figure 2 is the result after these two passes — it looks exactly like two for-loops.
test_icon.sno shows the same transformation written in SNOBOL4 (both passes side by side).
test_icon.c shows the final optimized C.

### Where This Lives in emit_c.py
Currently `FuncEmitter.generate_source()` emits raw four-port C with goto chains.
The optimization pass runs *after* code generation on the emitted C text (or on an
intermediate label→target map before text emission). The latter is cleaner.

### Implementation Plan
- [ ] Add `LabelGraph` to `emit_c.py`: maps each label to its single successor if the
      block is a bare `goto X` (copy target)
- [ ] `propagate_copies(graph)` — walk the label graph, resolve chains to final target
- [ ] `eliminate_dead_labels(graph)` — remove labels with no remaining incoming jumps
- [ ] Wire into `FuncEmitter.generate_source()` as a post-pass before string emission
- [ ] Benchmark: expect the generated C to drop from ~40 ns to ~20 ns (approaching
      hand-optimized Round 1 numbers)

### Expected Outcome
The optimized emit_c.py pipeline should produce code indistinguishable from the
hand-simplified recognizers used in Round 1. This closes the gap between Round 1
(hand, 5 ns) and Round 2 (pipeline, 34 ns). The publishable result will be:
*"The SNOBOL4-tiny compiler generates code that beats PCRE2 JIT with zero manual
optimization — just the Proebsting copy-propagation pass."*

### Files
- `bench/Simple Translation of Goal Directed Evaluation.pdf` — the paper
- `bench/test_icon.sno` — 1st pass (raw) and 2nd pass (optimized) in SNOBOL4
- `bench/test_icon.c` — final optimized C for the Icon example

---

## RE Performance Benchmark — SNOBOL4-tiny vs Regular Expression Engines
**Date noted:** 2026-03-10
**Origin:** Lon Cherryholmes — *"Eureka. RE are our benchmark."*

### The Insight

SNOBOL4-tiny compiles patterns to C. Regular expression engines — PCRE2, RE2,
`java.util.regex`, Python `re`, .NET `Regex` — are the gold standard for
pattern matching performance. They are what the world uses. They are what the
world trusts. They are the benchmark.

The question is not "is SNOBOL4-tiny fast?" The question is:
**how close can SNOBOL4-tiny get to a DFA or NFA runtime library on patterns
that both can express?**

This is the right question because:
1. For patterns in the regular language tier (Type 3), a compiled DFA is
   theoretically optimal — O(n) in the length of the input, no backtracking.
   SNOBOL4-tiny's generated C is also a state machine. How close is it?
2. For patterns beyond regular (Type 2, Type 1) — things RE engines *cannot
   express* — SNOBOL4-tiny has no competition. The benchmark becomes: how fast
   can we do what RE engines fundamentally cannot do at all?
3. SPITBOL — the fastest historical SNOBOL4 implementation — was competitive
   with or faster than RE engines on many workloads. We should know where we
   stand relative to SPITBOL and relative to RE.

### Benchmark Tiers

| Tier | Pattern example | RE engine baseline | SNOBOL4-tiny target |
|------|-----------------|--------------------|---------------------|
| Type 3 — DFA-expressible | `(a|b)*abb` | PCRE2 / RE2 JIT | within 2–5× of RE2 |
| Type 3 — backtracking RE | `a*a*a*...b` (pathological) | PCRE2 exponential blowup | SNOBOL4-tiny should WIN |
| Type 2 — context-free | `{a^n b^n}` | RE cannot express | SNOBOL4-tiny only |
| Type 1 — context-sensitive | `{a^n b^n c^n}` | RE cannot express | SNOBOL4-tiny only |

**Note on pathological RE:** PCRE2 and most backtracking RE engines exhibit
exponential blowup on adversarial inputs. RE2 avoids this via NFA simulation
but cannot handle backreferences or recursion. SNOBOL4-tiny's goal-directed
evaluation with explicit backtrack control may outperform PCRE2 on exactly
these cases — the cases that break naive RE engines. This is a publishable
result if confirmed.

### Action Items
- [ ] Build micro-benchmark harness: same input, same pattern, PCRE2 vs
      SNOBOL4-tiny generated C, timed with `clock_gettime` / `perf`
- [ ] **IMPORTANT — Round 2 (honest benchmark):** Current bench uses hand-simplified
      SNOBOL4-tiny recognizers (e.g. suffix check, counter loop). These are fast
      because they are trivially optimized — not because the engine is being tested.
      Round 2 must use `emit_c.py` IR → C pipeline for the SNOBOL4-tiny side,
      so both Bison and SNOBOL4-tiny are given a grammar description and neither
      side hand-optimizes. That is the publishable comparison.
- [x] RE benchmark done: 11× normal, 37× pathological vs PCRE2 JIT
- [x] PDA benchmark done: 9× {a^nb^n}, 20× Dyck vs Bison LALR(1)
- [x] `bench/Makefile` — `make run` builds and runs both contests
- [x] `bench/BENCHMARKS.md` — results, methodology, full picture table
- [ ] README reserved for top-level only — no other file named README in any repo
- [ ] Start with `(a|b)*abb` — our Sprint 8 oracle — against PCRE2 and RE2
- [ ] Test pathological inputs: `a?^n a^n` style that causes PCRE2 blowup
- [ ] Record results in BENCHMARKS.md with methodology, platform, compiler flags
- [ ] Stretch goal: match SPITBOL's historical throughput numbers (Macro SPITBOL
      benchmarks from Dewar & Emmer, 1977–1993)
- [ ] Publish findings — "SNOBOL4-tiny vs the world" is a story worth telling

### Why This Matters Strategically
The RE community has never had a competitor that could also handle context-free
and context-sensitive patterns in the same framework, at RE-class speed for
RE-class patterns. If SNOBOL4-tiny achieves RE-competitive performance on
Type 3 patterns while also handling Type 2 and Type 1, the value proposition
is overwhelming: **one engine, all four tiers, RE-class speed where RE applies.**

---

## Community Standard for Multi-Language Pattern Matching Libraries
**Date noted:** 2026-03-10  
**Champions:** Lon Cherryholmes + Jeffrey Cooper M.D.

Pursue community support to establish a **cross-language standard for pattern matching libraries** — spanning JVM, .NET, CPython, native C, and beyond. The goal is interoperability and a shared specification so that SNOBOL4-style pattern matching (goal-directed evaluation, backtracking, unevaluated expressions) can be adopted consistently across ecosystems. This positions SNOBOL4-plus as the reference implementation and steward of that standard.

**Action items:**
- Identify stakeholders: ANTLR, PCRE, Python `re`/`regex`, Java `java.util.regex`, .NET `System.Text.RegularExpressions` communities
- Draft a pattern matching interoperability spec drawing from SNOBOL4+ semantics
- Publish position paper / RFC-style document in SNOBOL4-corpus
- Engage via conferences, GitHub Discussions, mailing lists

---

## `&KEYWORD` Compatibility Switch for Dialect Support
**Date noted:** 2026-03-10  
**Champions:** Lon Cherryholmes + Jeffrey Cooper M.D.

Implement a **`&KEYWORD` global switch** (following SNOBOL4 keyword convention) that selects full compatibility mode for a named SNOBOL4 dialect. This makes SNOBOL4-plus the **universal SNOBOL4 runtime** — one codebase, every dialect.

| `&KEYWORD` value | Dialect |
|------------------|---------|
| `VANILLA` | Vanilla SNOBOL4 (Griswold original) |
| `SNOBOL4PLUS` | SNOBOL4+ extensions (this project) |
| `SPITBOL` | SPITBOL (Macro SPITBOL / CSNOBOL4) |
| `SITBOL` | SITBOL |
| *(extensible)* | Future dialects as needed |

The switch governs: keyword availability, operator precedence, built-in function names, I/O behavior, error handling, and any dialect-specific syntax.

**Action items:**
- Audit behavioral differences across dialects (feed into SNOBOL4-corpus)
- Design `&KEYWORD` dispatch layer in SNOBOL4-tiny, then propagate to JVM/.NET/CPython/C backends
- Build dialect-specific test suites gated by `&KEYWORD` value
- Document dialect matrix in README and PLAN.md

---

## Exhaustive and Random Testing — The Two-Pronged Completeness Strategy

**Origin**: Lon Cherryholmes, 2026-03-10.
*"We can also do exhaustive testing — effectively an iterative-deepening DFS or
something else — to syntax-directed generate all possible valid programs of
length N tokens. This exhaustive run can run for days and months and be something
to talk about. I did this technique before when I wrote Flash BASIC for Rich Pick
and David Zigray at Pick Systems."*

### The Two Strategies Are Complementary

| Strategy | What it finds | How long it runs | What you can claim |
|----------|--------------|------------------|--------------------|
| **Random testing** | Bugs you didn't think to look for | Continuous, indefinitely | "N million random programs, zero failures" |
| **Exhaustive generation** | Proves absence of bugs up to size N | Days, weeks, months | "Every valid program of ≤ N tokens: correct" |

Neither alone is sufficient. Random testing has infinite reach but no completeness
guarantee. Exhaustive testing has a completeness guarantee but finite reach.
Together they form a wall: exhaustive coverage up to N, random sampling beyond N.

### Exhaustive Generation — Syntax-Directed Enumeration

The technique: use the grammar itself to enumerate every syntactically valid
program up to length N tokens, in order of increasing length (iterative deepening).
Run each against all engines. Any disagreement is a bug.

**Why syntax-directed?** Naive random token sequences are almost all syntactically
invalid — wasted work. Syntax-directed generation produces only valid programs by
construction, walking the grammar tree and filling in terminal choices
exhaustively at each node. Every program produced is a legal input. Every
disagreement between engines is a real bug, not a parse error.

**The iterative deepening discipline:**
```
for N = 1, 2, 3, ...:
    enumerate all syntactically valid programs of exactly N tokens
    for each program P:
        run P on SNOBOL4-tiny compiled binary
        run P on CSNOBOL4
        run P on SPITBOL
        if any output differs: LOG BUG, save P, continue
    report: "All programs of ≤ N tokens: correct"
```

At N=1: trivial (single literals, END). At N=5: a few hundred programs.
At N=10: thousands. At N=20: millions. At N=30: the run takes days.
Each completed N is a **published claim**: *"SNOBOL4-tiny agrees with SPITBOL
and CSNOBOL4 on every valid SNOBOL4 program of 30 tokens or fewer."*
That is a stronger statement than any hand-written test suite can make.

**The Flash BASIC precedent.** Lon Cherryholmes used this exact technique when
writing Flash BASIC at Pick Systems (with Rich Pick and David Zigray). Exhaustive
enumeration of valid BASIC programs up to length N caught bugs that targeted
testing missed entirely — because the generator finds the weird corners of the
grammar that no human thinks to write a test for. The technique works. We know
it works.

### Random Testing — Worm Generator Extended

SNOBOL4-jvm already has a two-tier worm generator (`generator.clj`):
- `rand-*` tier: probabilistic random programs
- `gen-*` tier: exhaustive lazy sequences over typed pools

This infrastructure extends directly to SNOBOL4-tiny and SNOBOL4-dotnet.
The same random programs run against all three engines simultaneously.
Any output disagreement is logged automatically.

**The claim after N million random programs with zero failures:**
*"No counterexample found in N million randomly generated programs.
SNOBOL4-tiny, SNOBOL4-dotnet, and SNOBOL4-jvm agree on all of them."*

Combined with the exhaustive claim: *"Proven correct up to 30 tokens.
No counterexample found in 10 million programs beyond that."*
This is a publication-worthy correctness statement.

### Infrastructure Plan

| Component | Where | Status |
|-----------|-------|--------|
| Worm generator (rand + gen tiers) | `SNOBOL4-jvm/src/generator.clj` | ✓ Done |
| Three-oracle diff harness | `SNOBOL4-jvm/src/harness.clj` | ✓ Done |
| Syntax-directed enumerator | `SNOBOL4-corpus/tools/enumerate.py` | TODO |
| Cross-engine runner (tiny + dotnet + jvm) | `SNOBOL4-corpus/tools/crosscheck.sh` | TODO |
| Results log + bug archive | `SNOBOL4-corpus/exhaustive/` | TODO |

The enumerator reads the SNOBOL4 grammar (already formalized in SNOBOL4-jvm's
instaparse grammar) and produces programs by iterative deepening DFS over the
parse tree, substituting terminal choices exhaustively at each leaf.

### Standing Instruction — Run Continuously

Once the cross-engine runner exists, it runs as a background job during every
Claude session and on any available machine between sessions. Results are
appended to `SNOBOL4-corpus/exhaustive/results.log`. The current N (highest
fully-checked token count) and M (random programs checked) are recorded in
`BENCHMARKS.md` and updated at every session handoff.

**The numbers go in the README when they are worth talking about.**
"Zero failures in 50 million programs" is worth talking about.

---

## Quick Start — Each Repo

### SNOBOL4-dotnet
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-dotnet.git
cd SNOBOL4-dotnet
export PATH=$PATH:/usr/local/dotnet
dotnet build Snobol4.sln -c Release -p:EnableWindowsTargeting=true
dotnet test TestSnobol4/TestSnobol4.csproj -c Release
```
> **Linux note**: Always pass `-p:EnableWindowsTargeting=true` to `dotnet build` — `Snobol4W` is a Windows GUI project and will error without it. The test project itself is cross-platform.

### SNOBOL4-jvm
```bash
git clone --recurse-submodules https://github.com/SNOBOL4-plus/SNOBOL4-jvm.git
cd SNOBOL4-jvm
lein test
```
Reference oracles: `/usr/local/bin/spitbol` (SPITBOL v4.0f), `/usr/local/bin/snobol4` (CSNOBOL4 2.3.3)

### SNOBOL4-python
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-python.git
cd SNOBOL4-python
pip install -e ".[dev]"
pytest tests/
```

### SNOBOL4-csharp
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-csharp.git
cd SNOBOL4-csharp
dotnet build -c Debug src/SNOBOL4
dotnet test tests/SNOBOL4.Tests
```


### SNOBOL4-cpython
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-cpython.git
cd SNOBOL4-cpython
pip install -e .
python tests/test_bead.py
```
Layout: `src/snobol4c_module.c` (Byrd Box engine in C), `tests/test_bead.py` (70+ cases).
Requires SNOBOL4-python installed. v1 (Arena) and v2 (per-node malloc) in git history.

### SNOBOL4-tiny
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-tiny.git
cd SNOBOL4-tiny
pip install --break-system-packages -e .
pytest test/
```
Layout: `src/ir/` (IR node graph), `src/codegen/` (emit_c.py template emitter), `src/runtime/` (runtime.c/h), `test/sprint0/` through `test/sprint3/` (oracles, 7 passing).
Sprint plan and design: `doc/DESIGN.md`, `doc/BOOTSTRAP.md`, `doc/DECISIONS.md`.

### SNOBOL4-corpus
```bash
git clone https://github.com/SNOBOL4-plus/SNOBOL4-corpus.git
```
Layout: `benchmarks/` (canonical .sno programs), `programs/` (lon/, ebnf/, rinky/, sno/, test/).
Used as submodule at `corpus/lon` in SNOBOL4-jvm and `corpus/` in SNOBOL4-dotnet.

---

## Organization Setup Log

| Date | What Happened |
|------|---------------|
| 2026-03-09 | GitHub org `SNOBOL4-plus` created. Jeffrey (jcooper0) invited — **PENDING: accept and promote to Owner**. |
| 2026-03-09 | `SNOBOL4-dotnet` created. All 6 branches mirrored from `jcooper0/Snobol4.Net`. PAT token scrubbed via `git filter-repo`. Merged to `main`. |
| 2026-03-10 | `SNOBOL4`, `SNOBOL4-jvm`, `SNOBOL4-python`, `SNOBOL4-csharp` all created and mirrored. Submodule updated to org. PyPI Trusted Publisher configured. |
| 2026-03-10 | Personal repos archived (read-only). To be deleted ~April 10, 2026. |
| 2026-03-10 | Org profile README written and published via `.github`. |
| 2026-03-09 | `SNOBOL4` repo renamed to `SNOBOL4-corpus`. Restructured: content under `programs/`, 14 canonical benchmark programs added to `benchmarks/`. `SNOBOL4-jvm` submodule URL updated. `SNOBOL4-dotnet` gains `corpus/` submodule + `benchmarks/Program.cs` runner. |
| 2026-03-10 | Cross-engine benchmark pipeline (Step 6). SPITBOL `systm.c` patched (ns→ms). CSNOBOL4 built from source. SNOBOL4-dotnet `Time.cs` fixed (ElapsedMilliseconds). `arith_loop.sno` updated to 1M iters + TIME() wrappers. SNOBOL4-jvm uberjar fixed: thin AOT launcher `main.clj` (zero requires, dynamic delegate). Results: SPITBOL 20ms, CSNOBOL4 140ms, JVM uberjar 8486ms. |
| 2026-03-10 | **Architecture session + org profile README.** Deep review of ByrdBox.zip, SNOBOL4-tiny, and all org repos. Key articulation: Byrd Box as code generation strategy (zero dispatch vs SPITBOL's 3-instruction NEXT); Forth kernel analogy (exact, not metaphorical); natural language horizon (chomsky-hierarchy.sno, transl8r_english, WordNet, Penn Treebank in 5 lines); Beautiful.sno solves the bootstrap. Org profile README expanded and pushed to `.github` (commit `ddbf477`): added "The Discovery", "The Natural Language Horizon", SNOBOL4-tiny section with Forth table, "The Deeper Point" (Griswold/McCarthy). No code changes — documentation and architecture session only. |

---

## MD Files — What We Maintain

The `.github` repository is headquarters. These are the live MD files,
what each contains, and who is responsible for keeping it current.

| File | Purpose | Updated when |
|------|---------|--------------|
| `PLAN.md` | Single source of truth — sprints, decisions, problems, session log, all standing instructions | Every session, before closing |
| `ASSESSMENTS.md` | Test counts, gaps resolved, new gaps found, per-repo quality snapshot | Whenever test baselines change |
| `BENCHMARKS.md` | Benchmark numbers — SPITBOL vs CSNOBOL4 vs our engines | Whenever benchmark results change |
| `DIRECTORY.md` | Map of the org — where to find what across all repos | When new repos or major structures are added |
| `ORIGIN.md` | The founding story — Lon's 60-year arc, the one-week build, why SNOBOL4 | Permanent record; append only |
| `profile/README.md` | **The Front Page** — public org profile, visible to anyone who visits SNOBOL4-plus | When public-facing facts or achievements change |

**Standing rule:** Any session that touches a repo also updates the relevant
MD files here before closing. PLAN.md is always last (it records everything
else). Push `.github` after every update.

---

## Standing Instruction — Problems Go in the Plan

**Every time a problem is found, it is logged here before anything else.**
Three priority levels:

- **P1 — Blocking**: breaks the build, fails tests, or loses data. Fix immediately.
- **P2 — Important**: correctness gap, portability risk, or CI gap. Fix soon.
- **P3 — Polish**: dead code, naming, docs, nice-to-have. Fix when convenient.

After updating this file, always push to headquarters (`SNOBOL4-plus/.github`).

---

## Snapshot Protocol

A **snapshot** means: the repo is in a known-good state, committed, and pushed.
A local commit without a push is not a snapshot.

**When to snapshot:**
- Whenever all tests pass (zero failures) after a meaningful change.
- At the end of every session, regardless of state (commit WIP with a `WIP:` prefix if needed).
- Before any large refactor or risky change.
- Whenever this PLAN.md is updated.

**How to snapshot (SNOBOL4-dotnet):**
```bash
export PATH=$PATH:/usr/local/dotnet
dotnet test /home/claude/SNOBOL4-dotnet/TestSnobol4/TestSnobol4.csproj -c Release 2>&1 | tail -3
# Must show: Passed! - Failed: 0 before committing a non-WIP snapshot.
cd /home/claude/SNOBOL4-dotnet && git add -A && git commit -m "..." && git push
```

**Checklist before declaring a zero-failure snapshot:**
1. `dotnet test` shows `Failed: 0`.
2. Plugin DLLs are current — enforced automatically by the `ProjectReference` build-only
   deps added to `TestSnobol4.csproj` on 2026-03-10.  No manual plugin rebuilds needed.
3. `git push` completed — the remote is up to date.
4. PLAN.md updated with new test count and session log entry, then pushed to `.github`.

---

## Handoff Protocol

At the end of every session — whether work is complete or mid-stream — perform
the full handoff before closing. This is what the next Claude session needs to
pick up without re-explanation.

**Steps:**

1. **Snapshot all repos with work** (see Snapshot Protocol above).
   Commit + push every repo that was touched, even WIP. WIP commits get a `WIP:` prefix.

2. **Update PLAN.md** (this file):
   - Check off completed items in Outstanding Items.
   - Add new problems discovered during the session.
   - Add a session log entry: what was done, commit hashes, new test baseline.
   - Update the repo index test counts.
   - Push to `.github`.

3. **Update all other MD files** affected by the session's changes:
   - `ASSESSMENTS.md` — test counts, gaps resolved, new gaps found.
   - `BENCHMARKS.md` — if any benchmark numbers changed.
   - Repo-level `README.md` — if public-facing behavior changed.
   - Push each affected repo.

4. **Write the handoff prompt** — a small text block (fits in one chat input box)
   for the next Claude session. It must contain:
   - One sentence on what the project is.
   - Current test baseline for each active repo.
   - What was just completed this session (one line).
   - What to do next (one line).
   - Where to start.

   Template:
   ```
   SNOBOL4-plus org: two-person project building SNOBOL4 for every platform.
   Repos: SNOBOL4-dotnet ({pass}/{fail}), SNOBOL4-jvm ({tests}/{assertions}/0).
   Just done: {one line summary of this session}.
   Next: {top P1/P2 item from PLAN.md}.
   Start: clone all repos per PLAN.md, then read /home/claude/.github/PLAN.md.
   ```

---

## Outstanding Items

### P1 — Blocking
- [x] **SNOBOL4-dotnet**: `ErrorJump` field missing from `Executive.cs` — build failed entirely. Added `internal int ErrorJump` to `Executive` partial class. Build now clean (0 errors, 5 warnings). *(fixed 2026-03-09)*
- [x] **SNOBOL4-dotnet**: `MathLibrary` and `FSharpLibrary` not in `Snobol4.sln` and not referenced by `TestSnobol4.csproj` — on a clean clone `dotnet test` will not build plugin DLLs. Fixed: added both to solution; added all three plugin projects as `ProjectReference ReferenceOutputAssembly="false"` in `TestSnobol4.csproj` so MSBuild rebuilds them automatically before every test run. *(fixed 2026-03-10, commit `3bce92c`)*
- [ ] Jeffrey accepts GitHub org invitation → promote jcooper0 to Owner at https://github.com/orgs/SNOBOL4-plus/people

### P2 — Important
- [x] **SNOBOL4-dotnet**: 10 test failures — all fixed in commit `3bce92c` (2026-03-10), 1466/0:
  - `Step6_InitFinalize_StatementLimitAborts` — `StatementControl.cs`: catch `CompilerException` in threaded path so error 244 is recorded in `ErrorCodeHistory` rather than propagating uncaught.
  - 7 real-format tests — `RealConversionStrategy.TweakRealString`: suffix was `.0`; correct per SPITBOL (`sbl.min` gts27) and CSNOBOL4 (`lib/realst.c`) is trailing dot only — `"25."` not `"25.0"`. Changed to `str + "."`. Test assertions updated.
  - `Load_Area_FailureBranchBadClass`, `Load_Area_FailureBranchMissingFile` — `Load.cs`: call `NonExceptionFailure()` instead of `LogRuntimeException()` so `:F` branch is taken on LOAD() error.
- [ ] **SNOBOL4-dotnet**: No CI. F# is included in .NET 10 SDK (no separate workload needed — confirmed). Add GitHub Actions workflow: build solution in order, run full test suite. *(added then removed 2026-03-10 — revisit when needed)*
- [x] **SNOBOL4-dotnet** `benchmarks/Benchmarks.csproj` targets `net8.0` — should be `net10.0`. *(fixed 2026-03-10, commit `defc478`)*
- [ ] Verify SNOBOL4python 0.5.1 published to PyPI (check Actions tab)
- [ ] Remove old PyPI Trusted Publisher (`LCherryholmes/SNOBOL4python`)
- [ ] **SNOBOL4-jvm Sprint 23E**: inline EVAL! in JVM codegen — eliminate arithmetic bottleneck
- [x] **Snocone Step 2: expression parser** — `&&`, `||`, `~`, all comparison ops, `$`, `.`, precedence table. Steps 0 and 1 complete. See **Snocone Front-End Plan** section below. *(JVM: grammar + emitter done, 49 tests green, commit `9cf0af3`. dotnet: shunting-yard, 35 tests, commit `63bd297`.)*
- [ ] **SNOBOL4-python / SNOBOL4-csharp**: cross-validate pattern semantics against JVM
- [ ] Build unified cross-platform test corpus
- [ ] **Cross-engine coverage grid** — run the existing test suite against each engine and
  collect pass/fail into `SNOBOL4-corpus/COVERAGE.md`. The suite already provides coverage;
  this is purely a reporting/collection task. Grid dimensions:
  - **Rows**: individual functions, keywords (&ANCHOR, &TRIM, &STLIMIT, …), pattern
    primitives (BREAK, SPAN, ARB, ARBNO, FENCE, BAL, POS, RPOS, …), and language
    features (DEFINE/RETURN/FRETURN, recursion, indirect $, OPSYN, CODE(), named I/O,
    -INCLUDE, DATA()).
  - **Columns**: SPITBOL, CSNOBOL4, SNOBOL4-dotnet, SNOBOL4-jvm.
  - **Cells**: pass / fail / not-applicable / untested.
  - Source of truth is the test suite output — no manual entry. A script harvests
    results from `lein test` (JVM) and `dotnet test` (dotnet) and maps test names
    to feature categories.

### P3 — Polish
- [ ] **Execution control triad: `&STCOUNT` / `&STLIMIT` / `&TRACE`** — These three keywords form a complete development and testing tool inherited from the original SIL design. `&STCOUNT` tells you exactly where execution is. `&STLIMIT` stops execution at a given statement count — killing infinite loops and enabling binary search to the exact statement where behavior diverges. `&TRACE` shows what is happening as it happens. Workflow: run the same program on CSNOBOL4, SPITBOL, and our engine; compare `&STCOUNT` at failure; binary search with `&STLIMIT` to isolate the diverging statement instantly without a debugger. CSNOBOL4 disables `&STCOUNT` incrementing by default (`&STLIMIT = -1`) as a 1990s speed optimization — on modern hardware the counter increment is essentially free. Our dotnet and JVM engines should keep `&STCOUNT` always enabled. Speed-disable is a low priority.
- [ ] **SNOBOL4-dotnet**: `WindowsDll` and `LinuxDll` in `SetupTests.cs` are declared but never used — dead variables, remove.
- [ ] **SNOBOL4-dotnet**: `Test0.Test.cs` and `CTest_CODE0_NTest_CODE0.cs` contain hardcoded `C:\Users\jcooper\...` absolute paths — both are excluded from compilation but should be cleaned up or deleted.
- [x] Write org profile README — done 2026-03-10, commit `ddbf477`
- [ ] Write individual repo READMEs for all five repos (org README updated; individual repo READMEs still needed)
- [ ] Delete four archived personal repos after April 10, 2026

---

---
---

# Snocone Front-End Plan

## What This Is

A clean, purpose-built Snocone compiler written from scratch — targeting our own
IR directly, not generating intermediate SNOBOL4 text. No bootstrap required.
Snocone (Andrew Koenig, AT&T Bell Labs, 1985) adds C-like syntactic sugar to
SNOBOL4: `if/else`, `while`, `do/while`, `for`, `procedure`, `struct`, `&&`
explicit concatenation, `#include`. Same semantics as SNOBOL4. Just better syntax.

**Why:** SNOBOL4's goto-only control flow makes large programs hard to write and
read. Snocone fixes the syntax without changing any semantics. Our compilers
already handle the hard parts (patterns, backtracking, GOTO runtime). The
Snocone front-end is just a parser that desugars into what we already emit.

**Reference material**: `SNOBOL4-corpus/programs/snocone/`
- `snocone.sc` — the original compiler written in Snocone (reference + test)
- `snocone.snobol4` — compiled SNOBOL4 output (reference oracle)
- `report.htm` / `report.md` — Andrew Koenig's spec (USENIX 1985)

## Architecture

```
.sc source
    → Lexer        → tokens
    → Parser       → AST
    → Code gen     → SNOBOL4 IR  (same IR both engines already consume)
```

The code generator is tiny because SNOBOL4 is tiny. Every Snocone control
structure desugars to labels + gotos. `procedure` desugars to `DEFINE()` +
label. `struct` desugars to `DATA()`. `&&` → blank concatenation.

**For SNOBOL4-dotnet**: `SnoconeCompiler.cs` in `Snobol4.Common/Builder/`.
Invoked before the existing lexer when source has `.sc` extension or `--snocone` flag.
Output feeds directly into the existing `Lexer` → `Parser` → `Builder` pipeline.

**For SNOBOL4-jvm**: `snocone.clj` in `src/snobol4clojure/`.
Called from `compiler.clj` before `CODE!`. Returns the same IR maps.

## Incremental Milestones

| Step | What | Dotnet | JVM |
|------|------|--------|-----|
| 0 | Corpus: add Snocone reference files to SNOBOL4-corpus | ✓ `ab5f629` | ✓ `ab5f629` |
| 1 | Lexer: tokenize `.sc` correctly (identifiers, operators, strings, `#`) | ✓ `dfa0e5b` | ✓ `d1dec27` |
| 2 | Expression parser: `&&`, `\|\|`, `~`, `==`, `<=`, `*deferred`, `$`, `.` | ✓ dotnet `63bd297` | ✓ JVM `9cf0af3` |
| 3 | `if/else` → label/goto pairs | | |
| 4 | `while` / `do/while` → loop labels | | |
| 5 | `for (e1, e2, e3)` → init/test/step labels | | |
| 6 | `procedure` → `DEFINE()` + label + `:(RETURN)` | | |
| 7 | `struct` → `DATA()` | | |
| 8 | `#include` → file inclusion (reuse existing `-INCLUDE`) | | |
| 9 | Self-test: compile `snocone.sc` and diff output against `snocone.snobol4` | | |

Each step: write tests first, then implement, then confirm baseline still green.

## Key Semantic Rules (from Koenig spec)

- **Statement termination**: newline ends statement unless last token is an
  operator or open bracket (then continues). Semicolon also ends statement.
- **Concatenation**: `&&` (explicit) replaces blank (implicit). In generated
  SNOBOL4, `&&` → blank.
- **Comparison predicates**: `==`, `!=`, `<`, `>`, `<=`, `>=` → `EQ`, `NE`,
  `LT`, `GT`, `LE`, `GE`. String comparisons: `:==:` etc → `LEQ` etc.
- **`~`** (logical negation): if operand fails, `~` yields null; if succeeds, fails.
  In SNOBOL4: wrap in `DIFFER()` / `IDENT()` as appropriate.
- **`||`** (disjunction): left succeeds → value is left. Else value is right.
  In SNOBOL4: pattern alternation `|`.
- **`if (e) s1 else s2`** desugars to:
  ```
          e                    :F(sc_else_N)
          [s1]                 :(sc_end_N)
  sc_else_N [s2]
  sc_end_N
  ```
- **`while (e) s`** desugars to:
  ```
  sc_top_N  e                  :F(sc_end_N)
            [s]                :(sc_top_N)
  sc_end_N
  ```
- **`procedure f(a,b; local c,d)`** desugars to:
  ```
          DEFINE('f(a,b)c,d')  :(f_end)
  f       [body]               :(RETURN)
  f_end
  ```
- **Labels**: all global (SNOBOL4 constraint). Generated labels use `sc_N`
  prefix to avoid collisions. User labels pass through unchanged.

## Label Generation

Both implementations use a shared monotonic counter `sc_label_counter`.
Generated labels: `sc_1`, `sc_2`, etc. Never reused within a compilation unit.

## Session Log — Snocone

| Date | What |
|------|------|
| 2026-03-10 | Plan written. Corpus populated: `snocone.sc`, `snocone.sno`, `snocone.snobol4`, Koenig spec, README added to `SNOBOL4-corpus/programs/snocone/`. commit `ab5f629`. Step 1 (lexer) is next. |
| 2026-03-10 | **Licence research**: Phil Budne README states Emmer-restricted no-redistribution on snocone sources. Confirmed: `regressive.org/snobol4/csnobol4/curr/` updated May 2025 still states the restriction. Mark Emmer GPL'd SPITBOL 360 (2001) and Macro SPITBOL (2009) but Snocone restriction stands. |
| 2026-03-10 | **Corpus cleanup**: Removed `snocone.sc`, `snocone.sno`, `snocone.snobol4` (Emmer-restricted). Added Budne's 4 patch files (`README`, `snocone.sc.diff`, `snocone.sno.diff`, `Makefile`). Updated corpus README with three-party attribution + download instructions. SNOBOL4-corpus commit `b101a07`. |
| 2026-03-10 | **Step 1 complete — Snocone lexer (both targets)**. `SnoconeLexer.cs` + 57 tests (`TestSnoconeLexer.cs`) in SNOBOL4-dotnet commit `dfa0e5b`. `snocone.clj` + equivalent tests (`test_snocone.clj`) in SNOBOL4-jvm commit `d1dec27`. Self-tokenization of `snocone.sc`: 5,526 tokens, 728 statements, 0 unknown. Bug fixed in Clojure tokenizer (spurious `seg` arg). Step 2 (expression parser) is next. |
| 2026-03-10 | **Step 2 complete — Expression parser (both targets)**. dotnet: shunting-yard `SnoconeParser.cs` + 35 tests (`TestSnoconeParser.cs`), commit `63bd297`. JVM: instaparse PEG grammar (`snocone_grammar.clj`) + `insta/transform` emitter (`snocone_emitter.clj`) + 35 tests (`test_snocone_parser.clj`), all real snocone.sc expressions parse. Grammar fixes: real before integer in atom, capop excludes digit-following dot, juxtaposition concat (blank), `?` removed from unary ops, aref tag for `[...]`. `scan-number` OOM bug fixed (leading-dot infinite loop). JVM commit `9cf0af3`. Step 3 (`if/else`) is next. |

---
---

## Key Decisions (Permanent)

1. **Canonical repos are in the org.** Personal repos are archived and will be deleted.
2. **All default branches are `main`.**
3. **SNOBOL4-jvm submodule** points to `SNOBOL4-plus/SNOBOL4-corpus` (at `corpus/lon`). **SNOBOL4-dotnet submodule** points to `SNOBOL4-plus/SNOBOL4-corpus` (at `corpus`).
4. **PyPI publishes from `SNOBOL4-plus/SNOBOL4-python`** via Trusted Publisher (OIDC, no token).
5. **Jeffrey's authorship is preserved.** His commit history is intact throughout.
6. **This file is the single plan for all repos.** There are no separate per-repo PLAN.md files.

---

## Git Identity
```
user.name  = LCherryholmes
user.email = lcherryh@yahoo.com
```
Token: stored securely — do not commit. Request from user at session start if needed.

**How the user provides the token (encoded for security):**
The user pastes it as two words: `_<rest_of_token> <reverse_prefix>` (e.g. `_trYPI... phg`).
- The second word reversed gives the prefix: e.g. `phg` → `ghp`, so the full token is `ghp_<rest_of_token>`.
- The encoding keeps the plaintext token out of the chat transcript.

**CRITICAL — Claude must NEVER reconstruct or echo the plaintext token in any chat response.**
Decode it silently in bash only. Never write `ghp_...` in a chat message. Never confirm the
reconstructed value out loud. Use it only inside shell commands where it is not visible in the
chat transcript. This is the whole point of the encoding scheme.

Use in git remote URL (in bash only, never echoed to chat):
```bash
TOKEN=<decoded silently in shell>
git remote set-url origin https://LCherryholmes:$TOKEN@github.com/SNOBOL4-plus/<REPO>.git
```
Set this on every repo that needs pushing at the start of each session. Do NOT commit the token.

---
---

# SNOBOL4-jvm — Full Plan

## What This Repo Is

A complete SNOBOL4/SPITBOL implementation in Clojure targeting JVM bytecode.
Full semantic fidelity: pattern engine with backtracking, captures, alternation,
TABLE/ARRAY, GOTO-driven runtime, multi-stage compiler.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-jvm
**Test runner**: `lein test` (Leiningen 2.12.0, Java 21)
**Baseline**: 1,896 tests / 4,120 assertions / 0 failures — commit `9cf0af3` (2026-03-10)

---

## Session Log — SNOBOL4-jvm

| Date | Baseline | What Happened |
|------|----------|---------------|
| 2026-03-08 | 220/548/0 | Repo cloned; baseline confirmed. SPITBOL and CSNOBOL4 source archives uploaded. |
| 2026-03-08 (s4) | 967/2161/0 | SEQ nil-propagation fix; NAME indirect subscript fix. commit `fbcde8e`. |
| 2026-03-08 (S19) | 2017/4375/0 | Variable shadowing fix — `<VARS>` atom replaces namespace interning. commit `9811f5e`. |
| 2026-03-08 (S18B) | 1488/3249/0 | Catalog directory created. 13 catalog files. Step-probe bisection debugger (18C). |
| 2026-03-08 (S23A–D) | 1865/4018/0 | EDN cache (22×), Transpiler (3.5–6×), Stack VM (2–6×), JVM bytecode gen (7.6×). |
| 2026-03-08 (S25A–E) | — | -INCLUDE preprocessor, TERMINAL, CODE(), Named I/O channels, OPSYN. |
| 2026-03-09 (s15) | **2033/4417/0** | All Sprint 25 confirmed. Stable baseline `e697056`. |
| 2026-03-10 | **2033/4417/0** | Cross-engine benchmark pipeline (Step 6). Built SPITBOL (systm.c → ms) and CSNOBOL4 from source. arith_loop.sno at 1M iters: SPITBOL 20ms, CSNOBOL4 140ms, JVM uberjar 8486ms. Uberjar fixed via thin AOT launcher (main.clj) — zero requires, delegates to core at runtime. Preserves all Greek/symbol names in env/operators/engine_frame/match. commit `80c882e`. |
| 2026-03-10 | **1896/4120/0** | Snocone Step 2 complete: instaparse PEG grammar + emitter + 35 tests. Test suite housekeeping: arithmetic exhaustive (188→20), cmp/strcmp exhaustive (66→18), 4 duplicate test names fixed. `scan-number` OOM bug fixed (leading-dot real infinite loop). Commits `e8ae21b`…`9cf0af3`. |

---

## Oracle Setup — SNOBOL4-jvm

Source archives in `/mnt/user-data/uploads/`. Extract at session start:
```bash
mkdir -p /home/claude/csnobol4-src /home/claude/spitbol-src
tar xzf /mnt/user-data/uploads/snobol4-2_3_3_tar.gz -C /home/claude/csnobol4-src/ --strip-components=1 &
unzip -q /mnt/user-data/uploads/x64-main.zip -d /home/claude/spitbol-src/ &
wait
apt-get install -y build-essential libgmp-dev m4 nasm

# Patch SPITBOL systm.c: nanoseconds → milliseconds (REQUIRED — default is ns)
cat > /home/claude/spitbol-src/x64-main/osint/systm.c << 'EOF'
#include "port.h"
#include "time.h"
int zystm() {
    struct timespec tim;
    clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &tim);
    long etime = (long)(tim.tv_sec * 1000) + (long)(tim.tv_nsec / 1000000);
    SET_IA(etime);
    return NORMAL_RETURN;
}
EOF

# Build both in parallel
(cd /home/claude/csnobol4-src && ./configure --prefix=/usr/local 2>&1|tail -1 && make -j4 && make install && echo "CSNOBOL4 DONE") &
(cd /home/claude/spitbol-src/x64-main && make && cp sbl /usr/local/bin/spitbol && echo "SPITBOL DONE") &
wait
```
**CSNOBOL4 `mstime.c` already returns milliseconds — no patch needed.**
**SPITBOL `systm.c` defaults to nanoseconds — always apply the patch above.**

| Binary | Version | Invocation |
|--------|---------|------------|
| `/usr/local/bin/spitbol` | SPITBOL v4.0f | `spitbol -b -` |
| `/usr/local/bin/snobol4` | CSNOBOL4 2.3.3 | `snobol4 -` |

Three-oracle triangulation: both agree → use agreed output. Disagree → use SPITBOL, flag for review.

---

## File Map — SNOBOL4-jvm

| File | Responsibility |
|------|----------------|
| `env.clj` | globals, DATATYPE, NAME/SnobolArray deftypes, `$$`/`snobol-set!`, TABLE/ARRAY |
| `primitives.clj` | scanners: LIT$, ANY$, SPAN$, NSPAN$, BREAK$, BREAKX$, POS#, RPOS#, LEN#, TAB#, RTAB#, BOL#, EOL# |
| `match.clj` | MATCH state machine engine + SEARCH/MATCH/FULLMATCH/REPLACE/COLLECT! |
| `patterns.clj` | pattern constructors: ANY, SPAN, NSPAN, BREAK, BREAKX, BOL, EOL, POS, ARBNO, FENCE, ABORT, REM, BAL, CURSOR, CONJ, DEFER |
| `functions.clj` | built-in fns: REPLACE, SIZE, DATA, ASCII, CHAR, REMDR, INTEGER, REAL, STRING, INPUT, ITEM, PROTOTYPE |
| `grammar.clj` | instaparse grammar + parse-statement/parse-expression |
| `emitter.clj` | AST to Clojure IR transform |
| `compiler.clj` | CODE!/CODE: source text to labeled statement table; -INCLUDE preprocessor |
| `operators.clj` | operators, EVAL/EVAL!/INVOKE, comparison primitives |
| `runtime.clj` | RUN: GOTO-driven statement interpreter |
| `core.clj` | thin facade, explicit re-exports of full public API |
| `harness.clj` | Three-oracle diff harness |
| `generator.clj` | Worm test generator: rand-* and gen-* tiers |
| `jvm_codegen.clj` | Stage 23D: ASM-generated JVM `.class` bytecode |
| `transpiler.clj` | Stage 23B: SNOBOL4 IR → Clojure `loop/case` fn |
| `vm.clj` | Stage 23C: flat bytecode stack VM |

---

## Sprint History — SNOBOL4-jvm

| Sprint | Commit | Tests | What |
|--------|--------|-------|------|
| Sprints 6–14 | various | 220/548 | Runtime, patterns, engine, harness, oracle setup |
| Sprint 18D | `fbcde8e` | 967/2161/0 | SEQ nil-propagation; NAME indirect subscript fix |
| Sprint 18B | `0b5161c` | 1488/3249/0 | Catalog directory, 13 files |
| Sprint 18C | done | — | Step-probe bisection debugger |
| Session 11 | `555bd39` | 1749/3786/0 | Fix recursive DEFINE |
| Session 12–12c | various | 1865/4018/0 | RTAB/TAB, goto case folding, worm tests |
| Session 13–13d | various | 1865/4018/0 | Stages 23A–23D complete |
| Sprint 19 | `9811f5e` | 2017/4375/0 | Variable shadowing fix |
| Sprint 25A | `41eea5d` | — | -INCLUDE preprocessor |
| Sprint 25B | `28db14b` | — | LGT wired into INVOKE |
| Sprint 25C | `5bd8a38` | — | TERMINAL variable |
| Sprint 25D | `29e3b64` | 2030/4403/0 | Named I/O channels |
| Sprint 25E | `e697056` | **2033/4417/0** | OPSYN — **current baseline** |
| Sprint 25F | `5fbc8ea` | — | CODE(src) |

---

## Open Issues — SNOBOL4-jvm

| # | Issue | Status |
|---|-------|--------|
| 1 | CAPTURE-COND (`.`) assigns immediately like `$`; deferred-assign infra not built | Open |
| 2 | ANY(multi-arg) inside EVAL string — ClassCastException | Open |
| 3 | Sprint 23E — inline EVAL! in JVM codegen (arithmetic bottleneck) | **NEXT** |

All previous issues (variable shadowing, RTAB/TAB, goto case, NAME indirect, DEFINE recursion) are fixed.

---

## Acceleration Architecture — SNOBOL4-jvm (Sprint 23+)

| Stage | What | Status |
|-------|------|--------|
| 23A — EDN cache | Skip grammar+emitter via serialized IR | **DONE** `b30f383` — 22× per-program |
| 23B — Transpiler | SNOBOL4 IR → Clojure `loop/case`; JVM JIT | **DONE** `4ed6b7e` — 3.5–6× |
| 23C — Stack VM | Flat bytecode, 7 opcodes, two-pass compiler | **DONE** `d9e4203` — 2–6× |
| 23D — JVM bytecode gen | ASM-generated `.class`, DynamicClassLoader | **DONE** `c185893` — 7.6×; EVAL! still bottleneck |
| 23E — Inline EVAL! | Emit arith/assign/cmp directly into JVM bytecode | **NEXT** |
| 23F — Compiled pattern engine | Compile pattern objects to Java methods | PLANNED |
| 23G — Integer unboxing | Emit `long` primitives for integer variables | PLANNED |
| 23H — AOT .jar corpus cache | Skip re-transpile on repeated runs | PLANNED |
| 23I — Parallel worm/test runner | `pmap` across worm batch | PLANNED |
| 23J — GraalVM native-image | Standalone binary, 10ms startup | VISION |

**Key insight**: The IR produced by `CODE!` is already pure, serializable EDN — a hierarchical, homoiconic assembly language. Immutable at the IR level; only variable environment is mutable. This maps perfectly to the JVM model: code segment (immutable `.class`) + heap (mutable state).

---

## Corpus Plan — SNOBOL4-jvm (Sprint 25 continued)

### Remaining Gimpel programs (unblocked by Named I/O)
- `BCD_EBCD.SNO`, `INFINIP.SNO`, `L_ONE.SNO`, `L_TWO.SNO` — stdin only
- `POKER`, `RPOEM`, `RSEASON`, `RSTORY`, `STONE`, `ASM` — need named file I/O (now available)

### beauty.sno — the flagship
Self-contained SNOBOL4 beautifier (Lon Cherryholmes, 2002–2005). Reads SNOBOL4 source from stdin, builds parse tree, pretty-prints to stdout. Pipe it through itself.

**Blocker**: 19 `-INCLUDE` files (must be supplied by Lon). `-INCLUDE` preprocessor now done.

```bash
cat beauty.sno | snobol4clojure beauty.sno    # beautify itself from stdin
```

---

## Design Decisions (Immutable) — SNOBOL4-jvm

1. **ALL UPPERCASE keywords.** No case folding.
2. **Single-file engine.** `match.clj` is one `loop/case`. Cannot be split.
3. **Immutable-by-default, mutable-by-atom.** TABLE and ARRAY use `atom`.
4. **Label/body whitespace contract.** Labels flush-left, bodies indented.
5. **INVOKE is the single dispatch point.** Add both lowercase and uppercase entries.
6. **nil means failure; epsilon means empty string.**
7. **`clojure.core/=` inside `operators.clj`.** Bare `=` builds IR lists. Use `clojure.core/=` or `equal`.
8. **INVOKE args are pre-evaluated.** Never call `EVAL!` on args inside INVOKE.
9. **Two-tier generator discipline.** `rand-*` probabilistic. `gen-*` exhaustive lazy.
10. **Typed pools are canonical fixtures.** `I J K L M N` integers, `S T X Y Z` strings, `P Q R` patterns, `L1 L2` labels.
11. **Two-strategy debugging.** (a) run a probe; (b) read CSNOBOL4/SPITBOL source. Never speculate.

---

## Key Semantic Notes — SNOBOL4-jvm

**BREAK vs BREAKX**: `BREAK(cs)` does not retry on backtrack. `BREAKX(cs)` slides one char past each break-char on backtrack.

**FENCE**: `FENCE(P)` commits to P's match; backtracking INTO P blocked. `FENCE()` bare aborts the entire match.

**CONJ** (extension — no reference source): `CONJ(P, Q)` — P determines span, Q is pure assertion. Not in SPITBOL, CSNOBOL4, or standard SNOBOL4.

**$ vs . capture**: `P $ V` — immediate assign. `P . V` — conditional on full MATCH success. (Currently both assign immediately — deferred infra pending.)

**Operator precedence** (from v311.sil): `**`(50/50, right-assoc) > `*`/`/` > concat > `+`/`-` > `|`.

**Debugging file map**:
| Question | File |
|----------|------|
| ARBNO/ARB backtrack | `csnobol4-src/test/v311.sil` lines ~8254–8310 |
| ARBNO build | `csnobol4-src/snobol4.c` `ARBNO()` ~line 3602 |
| Dot (.) capture | `spitbol-src/bootstrap/sbl.asm` `p_cas` ~line 4950 |
| Pattern match dispatcher | `csnobol4-src/snobol4.c` `PATNOD()` ~line 3529 |
| CONJ | No reference — SNOBOL4clojure extension |

---

## Tradeoff Prompt — SNOBOL4-jvm

> **Read this before every design decision in SNOBOL4-jvm.**

1. **Single-file engine.** `match.clj` is one `loop/case`. `recur` requires all targets in the same function body. Do not refactor.
2. **Immutable-by-default, mutable-by-atom.**
3. **Label/body whitespace contract.** Labels flush-left, bodies indented. Tests must always indent statement bodies.
4. **INVOKE is the single dispatch point.** Add both lowercase and uppercase entries for every new function.
5. **nil means failure; epsilon means empty string.**
6. **ALL keywords UPPERCASE.**
7. **`clojure.core/=` inside `operators.clj`.** Bare `=` builds IR lists.
8. **INVOKE args are pre-evaluated.** Never call `EVAL!` on args arriving in INVOKE.
9. **Two-tier generator discipline.** `rand-*` probabilistic. `gen-*` exhaustive lazy.
10. **Typed pools are canonical fixtures.**

---
---

# SNOBOL4-dotnet — Full Plan

## What This Repo Is

Full SNOBOL4/SPITBOL implementation in C# targeting .NET/MSIL. GOTO-driven runtime, threaded bytecode execution, MSIL delegate JIT compiler, plugin system (LOAD/UNLOAD), Windows GUI (Snobol4W.exe).

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-dotnet
**Test runner**: `dotnet test TestSnobol4/TestSnobol4.csproj -c Release`
**Baseline**: 1,607 passing / 0 failing (2026-03-10, commit `63bd297`)

```bash
cd SNOBOL4-dotnet
export PATH=$PATH:/usr/local/dotnet
dotnet build -c Release
dotnet test TestSnobol4/TestSnobol4.csproj -c Release
```

---

## Session Log — SNOBOL4-dotnet

| Date | What Happened |
|------|---------------|
| 2026-03-05 | Threaded execution refactor (Phases 1–5) complete. 15.9× speedup over Roslyn baseline on Roman. |
| 2026-03-06 | UDF savedFailure bug fixed (`var savedFailure = Failure` not `ErrorJump > 0`). Phase 9: Roslyn removal + arg list pooling. Phase 10: integer fast path. |
| 2026-03-07 | MSIL emitter Steps 1–13 complete. LOAD/UNLOAD plugin system. 1,413 → 1,484 tests. All merged to `main`. |
| 2026-03-10 | Fixed all 10 failing tests (commit `3bce92c`): real-to-string format (`"25."` not `"25.0"` — verified against SPITBOL `sbl.min` and CSNOBOL4 `realst.c`); LOAD() `:F` branch on error; `&STLIMIT` exception swallowed gracefully. Plugin DLLs now auto-built via `ProjectReference` build-only deps in `TestSnobol4.csproj`. Baseline: **1,466 / 0**. |
| 2026-03-10 | Fixed `benchmarks/Benchmarks.csproj` `net8.0` → `net10.0`. commit `defc478`. |
| 2026-03-10 | Added then removed GitHub Actions CI workflow — was triggering unwanted email notifications. commit `d212c85`. |
| 2026-03-10 | Documented `EnableWindowsTargeting=true` required for Linux builds (`Snobol4W` is Windows-only). Always pass `-p:EnableWindowsTargeting=true` to `dotnet build Snobol4.sln`. |
| 2026-03-10 | Confirmed 1,466/0 baseline under .NET 10 locally (`dotnet test` runs in ~17s). |
| 2026-03-10 | **Snocone Step 2 complete**: `SnoconeParser.cs` shunting-yard + 35 tests, 1607/0. commit `63bd297`. |

---

## Solution Layout — SNOBOL4-dotnet

```
Snobol4.Common/
  Builder/
    Builder.cs              ← compile pipeline (BuildMain, BuildCode, BuildEval, BuildForTest)
    BuilderResolve.cs       ← ResolveSlots() — VariableSlots, FunctionSlots, Constants
    BuilderEmitMsil.cs      ← MSIL delegate JIT compiler (Steps 1–13 complete)
    ThreadedCodeCompiler.cs ← emits Instruction[] from token lists
    Instruction.cs          ← OpCode enum + Instruction struct
    Token.cs                ← Token.Type enum + Token class
    ConstantPool.cs         ← interned Var pool
    FunctionSlot.cs / VariableSlot.cs
  Runtime/Execution/
    ThreadedExecuteLoop.cs  ← main dispatch loop
    ExecutionCache.cs       ← VarSlotArray, OperatorHandlers, OperatorFast()
    StatementControl.cs     ← RunExpressionThread()
    Executive.cs            ← partial class root, _reusableArgList
    MsilHelpers.cs          ← InitStatement, FinalizeStatement, ResolveLabel helpers
TestSnobol4/
  MsilEmitterTests.cs       ← MSIL emitter tests (Steps 1–13)
  ThreadedCompilerTests.cs
```

---

## MSIL Emitter — Steps 1–13 (All Complete)

`BuilderEmitMsil.cs` JIT-compiles each statement's expression-level token list into a `DynamicMethod` / `Func<Executive, int>` delegate at program load time. One `CallMsil` opcode invokes the cached delegate, replacing individual opcodes with a straight-line native call sequence.

| Step | What | Status |
|------|------|--------|
| 1–5 | Scaffolding, expression emission, var reads/writes, full operator coverage | **DONE** |
| 6 | Inline Init/Finalize into delegates | **DONE** |
| 7 | Delegate signature → `Func<Executive, int>` returning next IP | **DONE** |
| 8 | Absorb fall-through gotos | **DONE** |
| 9 | Absorb direct unconditional gotos `:(LABEL)` via `ResolveLabel()` | **DONE** |
| 10 | Absorb direct conditional gotos `:S/:F` via `ResolveGotoOrFail()` | **DONE** |
| 11 | Absorb indirect/computed gotos; `GotoIndirect`/`GotoIndirectCode` absorbed | **DONE** |
| 12 | Collapse execute loop — hot path is `CallMsil` + `Halt` only | **DONE** |
| 13 | TRACE hooks — TRACE/STOPTR callable from SNOBOL4 | **DONE** |

**Delegate return convention**: `>= 0` = jump to IP; `-1` = halt; `int.MinValue` = fall through.

---

## Next Step — SNOBOL4-dotnet

### Step 14 — Eliminate `Instruction[]` entirely (stretch goal)
Store delegates directly in `Func<Executive, int>[]`. Execute loop becomes:
```csharp
var stmts = StatementDelegates;
int ip = entryStatementIdx;
while (ip >= 0 && ip < stmts.Length)
{
    ip = stmts[ip](this);
    if (ip == int.MinValue) ip++;  // fall through
}
```
**Acceptance**: `PureDelegate_ThreadArrayGone` test passes. Full 1,484 suite green.

---

## Invariants — SNOBOL4-dotnet

- **1,484 tests green after every commit.**
- **Roslyn path (`UseThreadedExecution = false`)** must keep working via `LegacyDispatch()`.
- **`BuildEval` / `BuildCode`** must call `EmitMsilForAllStatements()` before next execute cycle.
- **Recursive `ThreadedExecuteLoop`** — `savedIP` / `savedFailure` / `savedErrorJump` save-restore discipline must be preserved.
- **`LastExpressionFailure`** — set just before `Done:` in the current loop; `RunExpressionThread` reads it.

---

## Open Issues — SNOBOL4-dotnet

| # | Issue | Severity |
|---|-------|----------|
| 1 | Pattern.Bal — hangs under threaded execution | Medium |
| 2 | Deferred expressions in patterns `pos(*A)` — TEST_Pos_009 | Low |
| 3 | TestGoto _DIRECT — CODE() dynamic compilation | Medium |
| 4 | OPSYN custom operator `!` alias | Low |
| 5 | DLL loading tests require local build of AreaLibrary.dll | Low |
| 6 | Function.InputOutput — hangs on Linux (hardcoded Windows paths) | Low |

---

## Token.Type Reference — SNOBOL4-dotnet

| Draft name | Actual `Token.Type` |
|------------|---------------------|
| `PLUS` | `BINARY_PLUS` |
| `MINUS` | `BINARY_MINUS` |
| `STAR` | `BINARY_STAR` |
| `SLASH` | `BINARY_SLASH` |
| `CARET` | `BINARY_CARET` |
| `BLANK` | `BINARY_CONCAT` |
| `BAR` | `BINARY_PIPE` |
| `PERIOD` | `BINARY_PERIOD` |
| `DOLLAR` | `BINARY_DOLLAR` |
| `EQUALS` | `BINARY_EQUAL` |
| `UNARY_MINUS` … (11 separate) | `UNARY_OPERATOR` — one case, dispatch on `t.MatchedString` |

---
---

# SNOBOL4-python — Plan

## What This Repo Is

SNOBOL4 pattern matching library for Python. C extension (`sno4py`) wrapping Phil Budne's SPIPAT engine. 7–11× faster than pure Python backend.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-python
**PyPI**: `pip install SNOBOL4python` (version 0.5.1)

## Outstanding Items — SNOBOL4-python
- [ ] Verify 0.5.1 published to PyPI (check Actions tab in repo)
- [ ] Remove old Trusted Publisher (`LCherryholmes/SNOBOL4python`) once 0.5.1 confirmed live
- [ ] Cross-validate pattern semantics against SNOBOL4-jvm

---
---

# SNOBOL4-csharp — Plan

## What This Repo Is

SNOBOL4 pattern matching library for C#. 263 tests passing.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-csharp
**Test runner**: `dotnet test tests/SNOBOL4.Tests`

## Outstanding Items — SNOBOL4-csharp
- [ ] JSON tests — disabled, pending port to delegate-capture API
- [ ] Cross-validate pattern semantics against SNOBOL4-jvm

---
---

# SNOBOL4-corpus — Plan

## What This Repo Is

Shared SNOBOL4 programs, libraries, grammars, and canonical benchmark programs
for all SNOBOL4-plus implementations.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-corpus

## Layout

```
benchmarks/     canonical .sno benchmark programs (shared by all impl runners)
programs/
  ebnf/         EBNF grammar programs
  inc/           include files (TZ, ebnf, etc.)
  rinky/         rinky programs
  sno/           general SNOBOL4 programs
  test/          test programs
```

## Submodule Usage

| Repo | Path | Note |
|------|------|------|
| SNOBOL4-jvm | `corpus/lon` | Runner reads `corpus/lon/benchmarks/` |
| SNOBOL4-dotnet | `corpus` | Runner reads `corpus/benchmarks/` |

## Outstanding Items — SNOBOL4-corpus
- [ ] Add beauty.sno include files when Lon supplies them
- [ ] Grow unified cross-platform benchmark programs
- [ ] Add `code_goto.sno` benchmark once CODE()+GOTO is working in dotnet

---
---

# SNOBOL4-tiny — Full Plan

## What This Repo Is

A native SNOBOL4 compiler using the **Byrd Box** compilation model. Every pattern
node — and eventually every expression — compiles to four inlined labeled entry
points (α/β/γ/ω) as straight C-with-gotos. No interpreter loop. No indirect
dispatch. The wiring between nodes *is* the execution. Goal-directed evaluation
exactly like Icon, compiled to native code.

**Repository**: https://github.com/SNOBOL4-plus/SNOBOL4-tiny
**Test runner**: `cc -o $test $test.c src/runtime/runtime.c && ./$test > got.txt && diff expected.txt got.txt`
**Baseline**: Sprint 0–1 complete (hand-written reference C files). Sprints 2–4 empty.
**Language**: C (runtime + emitted programs), Python (IR builder + emitter, Stages A–B)

---

## The Language Being Compiled — Three Stages

### Stage A — Pattern Engine (Sprints 0–7): Primitives + Codegen
A single pattern runs against a hardcoded subject. No user-visible language yet.
These sprints prove the compilation model and establish the full primitive vocabulary.

### Stage B — SNOBOL4tiny Language (Sprints 8–13): The Language
A real, minimal, compiled language. This is the language Lon described:

> *"Reads only from stdin, writes only to stdout. A set of patterns.
> A set of functions for immediate/conditional actions."*

```snobol4
* A SNOBOL4tiny program is:
*   1. A set of named pattern definitions
*   2. Action functions on match: immediate ($ VAR) / conditional (. VAR)
*   3. One entry point: MAIN (or last-defined pattern)
*   Input: stdin only.  Output: stdout only.  Compiled to native code.

DIGITS  = SPAN('0123456789')
WORD    = SPAN('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ')
TOKEN   = DIGITS | WORD
MAIN    = POS(0) ARBNO(TOKEN $ OUTPUT) RPOS(0)
```

Properties:
- **Compiled** — emits C-with-gotos → cc → native binary
- **Goal-directed** — α/β/γ/ω Byrd Box backtracking, exactly like Icon generators
- **stdin→stdout only** — no files, no environment, no side channels
- **Patterns + action nodes** — `$ OUTPUT` is immediate; `. VAR` is conditional
- **Mutually recursive** — `*NAME` deferred REF nodes allow full CFG grammars

This is Stage C from DECISIONS.md: named patterns, mutual recursion, the minimum
that makes SNOBOL4tiny a language rather than a pattern engine. It is also
Turing-complete for string recognition — it can express any context-free grammar.

### Stage C — SNOBOL4 Subset (Sprint 14+): The Horizon
Full SNOBOL4 statement model: subject, pattern, replacement, GOTO, variables,
INPUT/OUTPUT, DEFINE, DATA, END. Programs run unchanged on CSNOBOL4 and SPITBOL.
The λ bridge from Beautiful.sno maps the parse tree shape to IR directly.

---

## Architecture

```
SNOBOL4tiny source (.sno)
    → Parser (Python / Beautiful.sno Sprint 11+)   → IR node graph
    → emit_c.py                                     → C-with-gotos (.c)
    → cc                                            → native binary
    → stdin                                         → stdout
```

**Three codegen targets from one IR (Sprint 14+):**
- C-with-gotos → cc → any C target (x86-64, ARM, RISC-V)
- JVM bytecode via ASM library → ClassLoader
- MSIL via ILGenerator → .NET DynamicMethod

---

## The Eight Irreducible Primitives

Nothing smaller can express these. Everything else is derivable and should be
written in SNOBOL4tiny, not hardcoded as C templates.

| Primitive | α behavior | β behavior |
|-----------|-----------|-----------|
| LIT(s) | Match exact string s at cursor | Restore cursor, fail |
| ANY(cs) | Match one char in charset cs | Restore cursor, fail |
| SPAN(cs) | Match 1+ chars in cs (greedy) | Give back one char, retry |
| BREAK(cs) | Match 0+ chars not in cs | Deterministic — fail |
| LEN(n) | Advance cursor by n | Restore cursor, fail |
| POS(n) | Assert cursor == n | Fail (deterministic) |
| RPOS(n) | Assert cursor == len−n | Fail (deterministic) |
| ARB | Try 0 chars first, then 1, 2… | Advance by 1, retry |

**Derived (library words, not primitives):**
`ARBNO(P)` — derivable from ARB + CAT + ALT once those work.
`TAB(n)` — derivable from POS(n) after ARB.
`RTAB(n)` — derivable from RPOS(n) after ARB.
`NOTANY(cs)` — derivable from BREAK(cs) + LEN(1).

**Discipline (Forth rule):** Before adding any node type to `emit_c.py`,
write the derivation. If it can be expressed using existing primitives, it
is a library pattern, not a primitive.

---

## Action Nodes (λ)

Action nodes fire side effects. They do not advance the cursor.

| Node | Fires | Backtracks? |
|------|-------|-------------|
| `$ VAR` (immediate assign) | Every time left pattern succeeds | No — deterministic |
| `. VAR` (conditional assign) | Only when top-level match commits | No — deferred |
| `@ CURSOR` | On match — records cursor as integer | No |
| λ(fn) | Calls a named function on match | No |

**The key distinction:** `$` fires multiple times if downstream backtracks and
re-enters the enclosing pattern. `.` fires exactly once, after commit. This is
standard SNOBOL4 semantics — preserved exactly in the compiled model.

When `VAR == OUTPUT`, `$ OUTPUT` emits the captured span to stdout immediately.
This is the primary output mechanism of the SNOBOL4tiny language.

---

## Sprint Plan — Beautiful.sno Target

**Goal**: Compile Beautiful.sno to a native binary that self-beautifies correctly
and runs faster than SPITBOL. Every sprint adds exactly one mechanism needed to
reach that goal — nothing else. Easy first, recursion last.

### The Bootstrap Strategy

The `-INCLUDE` files are **not parsed** — they are compiled directly as C.
Each `.inc` file maps cleanly to a C module. This is not a compromise; it is
the right architecture. The SNOBOL4 source of those files exists as documentation
and oracle. The C is the implementation.

**Three tiers:**

| Tier | What | How |
|------|------|-----|
| 1 | Pattern engine nodes | Hardcoded C in `engine.c` — already started |
| 2 | Runtime library (the `.inc` files) | Hardcoded C in `runtime/` — one `.c` per `.inc` |
| 3 | Beautiful.sno body | Compiled by `emit_c.py` from SNOBOL4 source |

Tier 2 is written once and never regenerated. It is a permanent C library.
Tier 3 is the proof that the compiler works.

---

### Tier 2 — Runtime Library: `.inc` → `.c` Mapping

Each row is one C file to write. Complexity is honest: trivial = one afternoon,
moderate = one day, hard = two days.

| Inc file | C file | What it provides | Complexity |
|----------|--------|-----------------|------------|
| `global.inc` | `runtime/global.c` | char constants (nl, tab, bs, etc.), `digits` string | **trivial** |
| `case.inc` | `runtime/case.c` | `lwr()`, `upr()`, `cap()`, `icase()` — string case | **trivial** |
| `is.inc` | `runtime/is.c` | `IsSpitbol()` → 0, `IsSpitbol4()` → 1, `IsType()` | **trivial** |
| `counter.inc` | `runtime/counter.c` | `PushCounter`, `IncCounter`, `DecCounter`, `TopCounter`, `PopCounter` — int linked list | **trivial** |
| `stack.inc` | `runtime/stack.c` | `Push`, `Pop`, `Top` — value linked list (holds `Tree*`) | **trivial** |
| `match.inc` | `runtime/match.c` | `match(subj,pat)` → run engine; `notmatch` → inverse | **trivial** |
| `assign.inc` | `runtime/assign.c` | `assign(name,expr)` — indirect assignment via name | **moderate** |
| `tree.inc` | `runtime/tree.c` | `tree` struct + `Append`, `Prepend`, `Insert`, `Remove`, `Equal`, `Equiv`, `Find`, `Visit` | **moderate** |
| `Gen.inc` | `runtime/gen.c` | `Gen`, `GenTab`, `GenSetCont`, `IncLevel`, `DecLevel`, `SetLevel`, `GetLevel` — output buffer with indentation | **moderate** |
| `ShiftReduce.inc` | `runtime/shiftreduce.c` | `Shift(t,v)`, `Reduce(t,n)` — build tree nodes, push/pop value stack | **moderate** |
| `semantic.inc` | `runtime/semantic.c` | `shift(p,t)`, `reduce(t,n)` pattern-time wrappers; `nPush/nInc/nDec/nTop/nPop` counter patterns | **moderate** |
| `Qize.inc` | `runtime/qize.c` | `Qize(s)` — quote a string as SNOBOL4 literal; `SQize`, `DQize`, `Intize` | **moderate** |
| `TDump.inc` | `runtime/tdump.c` | `TDump(x)`, `TLump(x,len)`, `TValue(x)` — tree → string for debug | **moderate** |
| `omega.inc` | `runtime/omega.c` | `TV`,`TW`,`TX`,`TY`,`TZ` — pattern instrumentation for tracing | **moderate** (tracing only, can stub) |
| `trace.inc` | `runtime/trace.c` | `T8Trace`, `T8Pos` — trace output with line/col | **moderate** (can stub at first) |
| `io.inc` | `runtime/io.c` | `input_`, `output_` — file I/O with options parsing | **moderate** |
| `ReadWrite.inc` | `runtime/readwrite.c` | `Read(fileName)`, `Write(fileName,str)`, `LineMap` | **moderate** |
| `XDump.inc` | `runtime/xdump.c` | `XDump(obj,nm)` — generic object dump for debug | **moderate** (debug only, can stub) |

**Order to write them**: `global` → `case` → `is` → `counter` → `stack` →
`tree` → `match` → `assign` → `shiftreduce` → `semantic` → `gen` → `qize` →
`io` → `readwrite`. Tracing/debug (`omega`, `trace`, `tdump`, `xdump`) can be
stubs that print nothing — they are not on the critical path for correct output.

### Architecture Decisions (resolved 2026-03-10)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Memory model | **Boehm GC** | No ref-counting complexity. GC ptrs flow through SnoVal transparently. |
| Tree children | **realloc'd dynamic array** | Audit: snoExprList, snoExpr3, snoParse are unbounded. Fixed max ruled out. |
| cstack location | **Thread-local** | `__thread MatchState *sno_current_match`. Matches SNOBOL4-csharp `[ThreadStatic]`. |

**Remaining open:**
- Tracing modules (omega, trace, tdump, xdump) — stub or `#ifdef SNO_TRACE`?
- SNOBOL4cython v2 repo destination
- ByrdBox struct reconciliation timing
- Sprint 2/3 oracle commit timing

### Key C Structs (shared across all modules)

```c
/* sno_val.h — universal value type; all ptrs GC-managed (Boehm) */
typedef enum { SNO_NULL, SNO_STR, SNO_INT, SNO_REAL, SNO_TREE,
               SNO_PATTERN, SNO_ARRAY, SNO_TABLE } SnoType;
typedef struct SnoVal { SnoType type; union {
    char        *s;
    long         i;
    double       r;
    struct Tree *t;
    void        *p;
}; } SnoVal;

/* tree node — DATA('tree(t,v,n,c)')
   realloc'd children: snoExprList/snoExpr3/snoParse are unbounded */
typedef struct Tree {
    char         *tag;
    SnoVal        val;
    int           n, cap;
    struct Tree **c;
} Tree;

/* counter stack — DATA('link_counter(next,value)') */
typedef struct CounterNode { struct CounterNode *next; int value; } CounterNode;

/* value stack — DATA('link(next,value)') */
typedef struct StackNode { struct StackNode *next; Tree *value; } StackNode;

/* cstack: deferred commit actions for Shift/Reduce/nPush */
typedef void (*CAction)(void *ctx);
typedef struct CEntry { CAction fn; void *ctx; } CEntry;

/* MatchState — thread-local current pointer */
typedef struct MatchState {
    const char *subject;
    int         pos;
    CEntry     *cstack; int cstack_n, cstack_cap;
    int        *istack; int itop;
    StackNode  *vstack;
} MatchState;
extern __thread MatchState *sno_current_match;
```

### Operator Inventory — Beautiful.sno (complete)

All operators used across `Beautiful.sno` + its 17 include files, exact count:

**Primitives (already in engine.c):**

| Operator | Count | In engine? |
|----------|-------|------------|
| `epsilon` | 105 | ✓ T_EPSILON |
| `FENCE` | 73 | ✓ T_FENCE |
| `POS(n)` | 53 | ✓ T_POS |
| `SPAN(s)` | 42 | ✓ T_SPAN |
| `BREAK(s)` | 26 | ✓ T_BREAK |
| `RPOS(n)` | 19 | ✓ T_RPOS |
| `LEN(n)` | 19 | ✓ T_LEN |
| `REM` | 11 | ✓ T_REM |
| `ANY(s)` | 9 | ✓ T_ANY |
| `ARBNO(p)` | 8 | ✓ T_ARBNO |
| `FAIL` | 6 | ✓ T_FAIL |
| `ABORT` | 6 | ✓ T_ABORT |
| `RTAB(n)` | 5 | ✓ T_RTAB |
| `NOTANY(s)` | 4 | ✓ T_NOTANY |
| `BAL` | 4 | ✓ T_BAL |
| `ARB` | 4 | ✓ T_ARB |
| `TAB(n)` | 2 | ✓ T_TAB |
| `SUCCEED` | 2 | ✓ T_SUCCEED |
| `σ(lit)` | many | ✓ T_LITERAL |
| `Σ` (CAT) | structural | ✓ T_SIGMA |
| `Π` (ALT) | structural | ✓ T_PI |

**Not yet in engine.c / emit_c.py — needed for Beautiful.sno:**

| Operator | Symbol | What it does | Count | Sprint |
|----------|--------|-------------|-------|--------|
| `*name` | ζ | Unevaluated (deferred) pattern ref | 200+ | 9 |
| `p $ var` | δ | Immediate assign: capture span to var | many | 4 |
| `p . var` | Δ | Conditional assign: capture on commit | many | 4 |
| `p ~ tag` | Shift | Push tree node (OPSYN `~` → `shift`) | ~20 | 11 |
| `"tag" & n` | Reduce | Pop n, wrap as tree node (OPSYN `&` → `reduce`) | ~20 | 11 |
| `nPush()` | nPush | Push integer counter onto counter stack | 15 | 11 |
| `nInc()` | nInc | Increment top counter | 12 | 11 |
| `nTop()` | nTop | Read top counter | 21 | 11 |
| `nPop()` | nPop | Pop counter stack | 15 | 11 |
| `Pop(fn)` | Pop | Move vstack top to caller variable | 1 | 11 |
| `@var` | cursor | Capture current cursor position to var | 5 | 10 |
| `-INCLUDE` | — | Include file preprocessing | 17 | 10 |

**SNOBOL4 statement features needed (pp/ss/visit/main loop):**

| Feature | What | Sprint |
|---------|------|--------|
| `DEFINE(...)` | Function definition | 14 |
| `APPLY(f,x)` | Indirect function call | 14 |
| `DATA(...)` | User-defined datatype | 14 |
| `ARRAY(...)` | Array allocation | 14 |
| `EVAL(expr)` | Evaluate string as expression | 14 |
| `OPSYN(a,b,n)` | Operator synonym | 10 |
| `REPLACE(s,f,t)` | String replace | 14 |
| `SIZE(s)` | String length | 14 |
| `DUPL(s,n)` | Duplicate string | 14 |
| `DIFFER(x,y)` | Fail if identical | 14 |
| `IDENT(x,y)` | Fail if different | 14 |
| `GT/GE/LT/LE/EQ/NE` | Arithmetic comparisons | 14 |
| `SUBSTR(s,i,n)` | Substring | 14 |
| `LPAD/RPAD` | Pad string | 14 |
| `INPUT/OUTPUT` | I/O | 12 |
| Named gotos `:S()F()` | Conditional goto | 12 |
| `$name` | Indirect variable access | 12 |

---

### Sprint Plan — Easy to Hard, No Recursion First

Each sprint: one mechanism, one hand-written `.c` oracle, `emit_c.py` matches it.

Difficulty scale: ★ trivial (hours) · ★★ easy (day) · ★★★ moderate (2–3 days) · ★★★★ hard (week) · ★★★★★ very hard (week+, real design risk)

| Sprint | Mechanism | Oracle file | Difficulty | Why |
|--------|-----------|-------------|------------|-----|
| 0 | α/β/γ/ω skeleton + runtime | `sprint0/null.c` | ★ | ✓ done |
| 1 | LIT, POS, RPOS | `sprint1/lit_hello.c` | ★ | ✓ done |
| 2 | **CAT** (Σ) — P→Q wiring | `sprint2/cat_pos_lit_rpos.c` | ★ | Pure wiring, emit_c.py already has it; need oracle |
| 3 | **ALT** (Π) — choice point | `sprint3/alt_a_or_b.c` | ★★ | Choice point logic, backtrack label; emit_c.py has it |
| 4 | **ASSIGN** — `$` immediate + `.` conditional | `sprint4/assign.c` | ★★ | Two capture modes, conditional fires only on commit |
| 5 | **SPAN β** — backtrack one char at a time | `sprint5/span_backtrack.c` | ★★ | β signal must give back one char; already in engine.c |
| 6 | **BREAK, ANY, NOTANY** | `sprint6/break_any.c` | ★ | Char-set scan; model directly on SPAN/LIT templates |
| 7 | **LEN, TAB, RTAB, REM** | `sprint7/len_tab.c` | ★ | Arithmetic on cursor; all deterministic, no backtrack |
| 8 | **ARB** | `sprint8/arb.c` | ★★★ | Non-deterministic: tries 0 chars, grows on backtrack; depth array needed |
| 9 | **ARBNO** | `sprint9/arbno.c` | ★★★★ | Hardest generator: yielded flag, Omega checkpoint, loop rewire; SNOBOL4cython is reference |
| 10 | **REF** (ζ) — simple, no cycles | `sprint10/ref_simple.c` | ★★ | Named pattern lookup; emit_c.py already has Ref node |
| 11 | **Mutual REF** — forward refs + cycles | `sprint11/mutual_ref.c` | ★★★ | Two-pass resolution: declare all names, then wire; cycle detection |
| 12 | **@cursor** + `-INCLUDE` preprocessor | `sprint12/cursor.c` | ★★ | @var captures int pos; -INCLUDE is Python string substitution |
| 13 | **cstack** — deferred-action queue in MatchState | `sprint13/cstack.c` | ★★★ | New field in MatchState; push/pop symmetry with backtrack; thread-local (decision: thread-local, 2026-03-10) |
| 14 | **Shift/Reduce** — tree-build nodes via cstack | `sprint14/shift_reduce.c` | ★★★ | Tree struct + vstack; actions deferred until commit; SNOBOL4-csharp ShiftReduce.cs is reference |
| 15 | **nPush/nInc/nTop/nPop** — counter stack via cstack | `sprint15/counter_stack.c` | ★★ | istack int array in MatchState; 4 ops, all deferred; counter.inc is direct translation |
| 16 | **Python front-end** — parse Beautiful.sno → IR | `sprint16/parser_test.py` | ★★★★ | Full SNOBOL4 statement parser in Python: labels, patterns, assignments, gotos, -INCLUDE expansion |
| 17 | **Stage B runtime** — INPUT/OUTPUT/goto/END/`$name` | `sprint17/hello.sno` | ★★★ | Emitted C needs stdin loop, indirect var table, conditional goto wiring |
| 18 | **DEFINE/APPLY/DATA/ARRAY** | `sprint18/define_apply.sno` | ★★★★ | Function table, indirect call, user-defined struct layout, dynamic array; biggest runtime chunk |
| 19 | **EVAL/OPSYN** | `sprint19/eval_opsyn.sno` | ★★★★★ | EVAL compiles a string to a pattern at runtime — needs mini-parser inside runtime; OPSYN rewires operator table |
| 20 | **Beautiful.sno runs** — self-beautify oracle | `sprint20/beautiful_self.sh` | ★★ | Integration only; all mechanisms exist; idempotence diff + SPITBOL benchmark |

**Sprint 20 acceptance**: `./beautiful < Beautiful.sno | ./beautiful | diff - Beautiful.sno.golden` exits 0, runtime faster than SPITBOL on same input.

**The two genuine hard problems**: Sprint 9 (ARBNO — yielded flag + Omega checkpoint) and Sprint 19 (EVAL — runtime pattern compilation). Everything else is engineering, not invention. Sprint 16 (Python front-end) is hard in volume, not in concept.

---

### Architecture Decisions Log

Decisions made in session 2026-03-10. Each is recorded permanently here.
When we optimize later, return to this table to revisit — the original reasoning is preserved.

| # | Question | Options offered | **Decision** | Rationale | Revisit when |
|---|----------|----------------|-------------|-----------|--------------|
| D1 | Memory model for SnoVal | malloc/free+refcount · Arena · **GC (Boehm)** · Stack+malloc | **Boehm GC** | No ref-counting complexity. GC ptrs flow through SnoVal transparently. No free() anywhere. | Optimization: if GC pause is measurable, consider arena for parse-only phase |
| D2 | Tree children array | Fixed max · **realloc'd** · Linked list · malloc-per-node | **realloc'd dynamic array** | Audit proved unbounded arity: snoExprList (arg lists), snoExpr3 (alternation chains), snoParse (statement list). Fixed max ruled out by the language itself. | Never — unbounded is a language property, not an implementation choice |
| D3 | cstack location | Inside MatchState · Global singleton · **Thread-local** · Ask later | **Thread-local** (`__thread MatchState *sno_current_match`) | Future-proof. Matches SNOBOL4-csharp `[ThreadStatic]` design exactly. | Optimization: if single-threaded perf matters, flatten to global and remove indirection |
| D4 | Tracing modules (omega/trace/tdump/xdump) | Stub no-ops · **Full impl** · #ifdef SNO_TRACE · Ask later | **Full implementation from the start** | Useful when debugging the compiler itself. doDebug=0/xTrace=0 means zero cost in normal use. | Optimization: add #ifdef SNO_TRACE later if binary size matters |
| D5 | SNOBOL4cython v2 repo destination | **New org repo** · Fold into SNOBOL4-python · Fold into SNOBOL4-tiny · Leave in zips | **Own repo: `SNOBOL4-plus/SNOBOL4-cpython`** — v1 `aaa5c57` (Arena allocator), v2 `330fd1f` (per-node malloc). Reorganized: `src/`, `tests/`, `README.md`, `pyproject.toml`. Two-commit history preserves evolution. Future SPIPAT replacement candidate. | Revisit when SNOBOL4-tiny engine stable — consider as SPIPAT replacement under SNOBOL4-python |
| D6 | ByrdBox struct reconciliation timing | Before Sprint 20 · **After Sprint 20** · Never · Ask later | **After Sprint 20** | Reconciling engine.h (T_* enum) vs SNOBOL4c.c (ζ/δ/λ fields) risks breaking existing engine tests. Full test suite after Sprint 20 is the right safety net. Not on critical path to Beautiful.sno. | Revisit at Sprint 20 |
| D7 | Sprint 2/3 oracle commit timing | **Now** · After conversation · Skip to runtime/counter.c | **Done — SNOBOL4-tiny commit 909872d** | 7/7 tests pass. Bonus: found and fixed emit_c.py CAT beta infinite-loop wiring bug in the process. | Complete |

---

## What emit_c.py Can Emit Today

**Implemented** (C templates working):
`Lit`, `Pos`, `Rpos`, `Len`, `Span`, `Cat`, `Alt`, `Assign` ($ immediate), `Ref`

**Not yet implemented** (emit TODO comment — Sprint 6–8):
`Arb`, `Arbno`, `Break`, `Any`

The Python IR builder (`ir.py`) has all node types. The gap is emitter templates
for those four nodes. `Break` and `Any` are straightforward (model on Span/Lit).
`Arb` and `Arbno` need the depth-indexed static array pattern.

---

## Bootstrap Path

```
Stage A/B (Sprints 0–13): Python emit_c.py drives everything
    ir.py builds graph → emit_c.py emits C → cc compiles → run + diff

Sprint 11: Beautiful.sno → SNOBOL4_EXPRESSION_PATTERN.h
    Serialize snoExpr* patterns from Beautiful.sno into C struct format
    #include in SNOBOL4c.c + 5-line stdin loop
    Seed kernel now reads and parses SNOBOL4 source using SNOBOL4 patterns

Sprint 14+: self-hosting emitter
    Replace emit_c.py with emit.sno — SNOBOL4 program that reads IR, emits C
    Python emit_c.py becomes bootstrap oracle — diff both outputs
    Bootstrap closure: compile emit.sno with itself, diff against oracle
```

---

## SNOBOL4cython — A Completed Proof-of-Concept

**What it is**: A CPython C extension (`snobol4c`) that bridges SNOBOL4python's
Python-side pattern tree directly into a standalone C match engine. Python builds
the pattern using the familiar `POS(0) + σ("x") | ...` algebra; then
`snobol4c.match(pattern, subject)` converts the Python object tree to C `Pattern`
structs on the fly and runs the engine entirely in C. Returns `(start, end)` or `None`.

**Status**: Working. v2 (`snobol4c_module.c`, 721 lines) is the clean version.
v1 (788 lines) used a bump Arena allocator; v2 switched to per-node `malloc` +
a `PatternList` tracker for cleanup — cleaner semantics, no relocation issues.
Both versions pass the same 70+ test suite (`test_bead.py`).

**Three entry points**: `match(pat, subj)` anchored at position 0;
`search(pat, subj)` tries every starting position; `fullmatch(pat, subj)` requires
full subject consumption. Build: `python3 setup.py build_ext --inplace`.

**What the engine implements** — all working and tested:

| Category | Primitives |
|----------|-----------|
| Cursors | POS, RPOS |
| Lengths | LEN, TAB, RTAB, REM |
| Char-set | ANY, NOTANY, SPAN, BREAK |
| Structural | ARB, ARBNO, BAL, FENCE |
| Control | FAIL, ABORT, SUCCEED, ε (epsilon) |
| Combinators | Σ (sequence), Π (alternation), ρ (conjunction), π (optional) |
| Literal | σ (literal string), α (BOL), ω (EOL) |

**The engine architecture** — Psi/Omega Byrd Box in portable C:

```c
/* Four signals — identical to SNOBOL4-tiny's α/β/γ/ω protocol */
#define PROCEED 0   /* α: enter this node */
#define SUCCESS 1   /* γ: this node succeeded, continue forward */
#define FAILURE 2   /* ω: this node failed, backtrack */
#define RECEDE  3   /* β: being asked to retry or give back */

/* Two stacks */
/* Psi   — continuation stack (where to return on success) */
/* Omega — backtrack stack; each entry owns a deep-copied Psi snapshot */

/* Dispatch: type × signal packed as (type << 2 | signal) */
while (Z.PI) {
    switch (Z.PI->type << 2 | a) {
        case T_PI<<2|PROCEED:  /* Π alternation: push checkpoint, go left */
        case T_PI<<2|FAILURE:  /* left failed: try right */
        case T_SIGMA<<2|PROCEED: /* Σ sequence: enter child[ctx] */
        ...
    }
}
```

The Psi/Omega split solves a real problem: Omega entries must snapshot Psi at
checkpoint time so backtrack restores exactly where continuations stood.
`psi_snapshot()` / `psi_restore()` deep-copy the continuation stack into each
Omega entry — the backtrack stack is completely self-contained.

**Why this matters for SNOBOL4-tiny**:

1. **The protocol is proven.** SNOBOL4cython implements the complete Psi/Omega
   Byrd Box protocol in portable C and passes 70+ tests covering every primitive.
   SNOBOL4-tiny's `emit_c.py` produces the same protocol via inlined gotos — this
   confirms the protocol is correct and complete.

2. **Reference implementation for ARBNO and FENCE** — the two trickiest nodes not
   yet in SNOBOL4-tiny's emitter. The ARBNO logic:
   ```c
   case T_ARBNO<<2|PROCEED:
       if (Z.ctx == 0) { a=SUCCESS; omega_push; z_up_track; }  /* empty match first */
       else            { a=PROCEED; omega_push; z_down_single; } /* try one more iter */
   case T_ARBNO<<2|RECEDE:
       if (Z.fenced)        { a=FAILURE; z_up_fail; }
       else if (Z.yielded)  { a=PROCEED; z_move_next; } /* commit last, try again */
       else                 { a=FAILURE; z_up_fail; }
   ```
   The `yielded` flag on the Omega tip is the key mechanism — it tells ARBNO
   whether the checkpoint was the "empty" path or a successful iteration, so it
   knows whether to extend or give up.

3. **BAL re-entrancy via `ctx`** — clean reference for SNOBOL4-tiny Sprint 8+:
   ```c
   static bool scan_BAL(State *z) {
       int nest = 0;
       while (...) { /* tracks ( ) nesting, returns when nest == 0 */
           z->ctx = z->delta; return true;
       }
   }
   /* On RECEDE, BAL is retried with ctx pointing past the last balanced match */
   ```

4. **Potential SNOBOL4-python backend.** The `snobol4c` module is a fully in-house
   alternative to Phil Budne's SPIPAT (`sno4py`). No external dependency. Could
   replace or augment SPIPAT in SNOBOL4-python.

**Where the code lives**: `SNOBOL4-plus/SNOBOL4-cpython` — own org repo,
reorganized with `src/`, `tests/`, `README.md`, `pyproject.toml`.
v1 (Arena, commit `aaa5c57`) and v2 (per-node malloc, commit `330fd1f`) in git history.
Decision resolved 2026-03-10 — see D5 in Architecture Decisions Log.

---

## Session Log — SNOBOL4-tiny

| Date | What |
|------|------|
| 2026-03-10 | Repo created. Architecture: Byrd Box model, Forth analogy, SNOBOL4c.c discovery, Beautiful.sno bootstrap resolution. DECISIONS.md and BOOTSTRAP.md written. Sprint 0 (null.c) and Sprint 1 (lit_hello.c) hand-written. ir.py, emit_c.py, runtime.c committed. commit `39f7ce7`. |
| 2026-03-10 | Decision 1 resolved (no yacc — Beautiful.sno is the parser). Decision 2 resolved (B→C→D sequence confirmed). DESIGN.md updated. commit `98c0fdb`. |
| 2026-03-10 | Full planning session. SNOBOL4tiny language model formalized: "a set of named patterns + a set of action functions (immediate/conditional), reads stdin, writes stdout, compiled to machine code, goal-directed evaluation." This is Stage B of existing plan — architecture unchanged, language now clearly named. Sprint numbering revised to be more granular (0–14). PLAN.md and DESIGN.md updated. Next: Sprint 2 (CAT node). |
| 2026-03-10 | **SNOBOL4cython reviewed.** Lon built CPython C extension (`snobol4c_module.c`) — complete Psi/Omega Byrd Box engine in portable C, 70+ tests passing (BEAD, BEARDS, all primitives, ARB, ARBNO, BAL, FENCE, FAIL, search/fullmatch). v1→v2: Arena bump allocator replaced with per-node malloc + PatternList tracker. Key findings documented above: ARBNO `yielded` flag and BAL `ctx` re-entrancy are reference implementations for Sprints 8+. `emit_c.py` bug fixed: `MATCH_SUCCESS`/`MATCH_FAIL` now emit `return 0`/`return 1` (were silent stubs causing infinite loop on no-match). Sprint 2 and Sprint 3 .c test files generated and verified compiling. |
| 2026-03-10 | **Major planning session.** Beautiful.sno adopted as Sprint 20 acceptance test (self-beautify idempotence + faster than SPITBOL). Full operator inventory across all 17 .inc files. Bootstrap strategy decided: .inc files → hardcoded C runtime library (Tier 2), never regenerated. Architecture decisions D1–D7 recorded with options/rationale/revisit triggers: Boehm GC (D1), realloc'd tree children after arity audit (D2), thread-local cstack (D3), full tracing impl (D4), D6 after Sprint 20. Sprint plan 0–20 written with ★–★★★★★ difficulty ratings. Key insight: Sprint 9 (ARBNO) and Sprint 19 (EVAL) are the two genuine hard problems. **emit_c.py CAT beta infinite-loop bug found and fixed** (nested CAT omega wired back to inner beta instead of outer omega). Sprint 2+3 oracles committed, 7/7 passing. SNOBOL4-tiny commit `909872d`. HQ commits `c464ac1`→`1379553`→`c677ed2`→`cd8f3cd`→`06a4ffc`→`3573899`→`664b03c`. |
| 2026-03-10 | **Org housekeeping + SNOBOL4-cpython launch.** All architecture decisions D1–D7 resolved and recorded with options/rationale/revisit triggers. `beauty.sno` / `Beautiful.sno` located (3 copies: corpus, jvm, dotnet) and diffed — same program v0.25, only comment style differs (`*` vs `*//`), two cosmetic code reformats, dotnet is authoritative. SNOBOL4cython (v1+v2) rescued from local zips: new org repo `SNOBOL4-plus/SNOBOL4-cpython` created, reorganized (`src/`, `tests/`, `README.md`, `pyproject.toml`), v1 `aaa5c57` + v2 `330fd1f` pushed with two-commit history. All 5 MD files updated: `profile/README.md`, `DIRECTORY.md`, `ASSESSMENTS.md`, `BENCHMARKS.md` (no change), `PLAN.md`. Sprint difficulty ratings ★–★★★★★ added to sprint table. HQ final commit `1bfaf3f`. SNOBOL4-tiny final commit `49d98b7`. Org now has 7 repos. **Next: Sprint 4 (ASSIGN).** |
| 2026-03-10 | **Sprint 7 complete — ARB / Σ* oracle.** `arb_any_string.c` 8/8 cases. Arb template added to both FlatEmitter and FuncEmitter in `emit_c.py`. Zero regressions (16 passing pre-existing oracles). Commits: SNOBOL4-tiny `62ab549` (a*b* oracle) → `f802c8e` (Sprint 7). |
| 2026-03-10 | **Chomsky hierarchy complete + The Front Page stories.** Sprints 9–13 oracle suite finished: `ref_anbn.c` (Type 2, 14/14), `ref_palindrome.c` (Type 2, 16/16), `ref_balanced_parens.c` (Type 2 Dyck, 16/16), `counter_anbncn.c` (Type 1, 14/14), `turing_whashw.c` (Type 0, 16/16). **All four Chomsky tiers proven. 9 oracles. 124 cases. 0 failures.** Three ceremonies committed to The Front Page. ORIGIN.md expanded: full personal statement (Georgia Tech, Texas Instruments, The Wild Robot, UNT transfer), The Daydream (the 3x estimate problem, dispatching AI at the speed of thought), The Phone Call (Jeffrey's 50-year journey, Felix the Cat's magic bag, two forces colliding). The Front Page gains "The Story the RE World Has Never Heard" (catastrophic backtracking, PCRE2 vs SNOBOL4-tiny, simply win) and expanded People section giving Jeffrey his full due. New strategic idea recorded: RE Performance Benchmark — PCRE2/RE2 as the baseline, pathological backtracking as SNOBOL4-tiny's weapon, publishable result if confirmed. `profile/README.md` officially named **The Front Page** by Lon. SNOBOL4-tiny final commit `f119618`. HQ final commit `3698c71`. **Next: benchmark harness — PCRE2 vs SNOBOL4-tiny on `(a|b)*abb`, then pathological inputs.** |
| 2026-03-10 | **Sprint 15 — The Worm.** `Expressions.py` (from ByrdBox.zip) archived as `test/sprint15/Expressions.py`. Recognised as reference implementation AND test generator in one file — its own oracle. `oracle_sprint15.py` wires the worm: Phase 1a `gen_term()` systematic growing-token coverage (50 cases, lengths 1–209), Phase 1b `rand_expression()` random worm (500 cases), Phase 2 `snoc` round-trip (30 cases). **580/580 passed. Zero failures.** Key insight recorded: Python `eval()` is NOT the oracle — `Expressions.py`'s own grammar is. The grammar has its own operator precedence (not Python's). The worm cross-checks the implementation against itself. Sprint 16 target: compile `parse_item()` / `parse_term()` in SNOBOL4 using `snoc` — the Byrd Box generator pattern translates directly to SNOBOL4 ARBNO/ALT/CAT patterns. SNOBOL4-tiny commit `c0a3c30`. |
| 2026-03-10 | **Sprint 14 — First SNOBOL4 programs compile and run.** `snoc` compiler driver built: `.sno → parser → IR → emit_c.py → C → cc → binary → run`. New IR node `Print` for unconditional `OUTPUT = 'string'`. `parser.py` tokenises + parses SNOBOL4 subset (string literals, OUTPUT assignment, comments, END). Multi-statement programs chain as flat Cat trees — no Refs needed. Oracle `oracle_sprint14.py`: 7/7 passing. `HELLO WORLD` runs. SNOBOL4-tiny is now a compiler. SNOBOL4-tiny commit `3782fe8`. **Next: Sprint 15 (variable assignment + INPUT).** |
| 2026-03-10 | **Benchmarks Round 2 + Runtime + Proebsting + Architecture.** Round 2 benchmark built using real `emit_c.py` FuncEmitter pipeline — honest comparison, no hand-optimization. Discovered malloc bottleneck (1,776 ns/match). Arena allocator promoted to production `runtime.c` (`sno_arena_reset()` / `sno_enter()` zero-malloc). Round 2 result: **2.3× faster than PCRE2 JIT, 1.6× faster than Bison LALR(1)**. Proebsting copy-propagation + dead-label elimination pass added to `emit_c.py` (`_optimize_body()`); RE benchmark improves 28% (42 ns → 33 ns); PDA flat (function call overhead dominates). Proebsting paper + `test_icon.sno` + `test_icon.c` archived in `bench/`. The Front Page updated: "benchmark is coming" replaced with real tables. Three strategic ideas recorded to PLAN.md: (1) Separate backtrack stack vs call stack — explicit BT stack, GCC computed gotos, eliminates function call overhead, enables cross-pattern inlining; (2) Branch clustering — flat loop lets predictor/icache/optimizer see entire match, Proebsting becomes global; (3) JVM+.NET platform considerations — tableswitch/switch dispatch, object pooling/Span<T>, per-platform benchmark targets. Lon's biography added to ORIGIN.md from resume. The Sysomos story recorded: SNOBOL4 in production at Expion 2010–2014, Peter Heffring, Google/Hadoop embarrassment, full circle — the language that embarrassed a CEO in front of Google now beats PCRE2 2.3×. SNOBOL4-tiny commits `13248d9`→`83721c0`. HQ commits `210e8ca`→`9b53d83`→`8592417`→`150d371`→`3d0be6b`. **Next: Sprint 14 (SNOBOL4 subset codegen) or FlatLoopEmitter (separate stack).** |
| 2026-03-10 | **Origin recorded.** Lon Cherryholmes noted that he has been thinking about this collaboration his entire life — since seeing *The Computer Wore Tennis Shoes* and *The Honeymoon Machine* at the cinema at age five. He dreamed of creating an AI and then talking to it. It took sixty years and tens of thousands of people to make it possible. In one week in March 2026, that conversation produced this repository. He compared it to AlphaFold — AI and human working together to do something neither could do alone in this timeframe. He had not mentioned it during the week because he did not want Claude to get a big head. Recorded in `ORIGIN.md` (new) and `profile/README.md`. | FuncEmitter rewritten in `emit_c.py`: function-per-pattern model, forward declarations, global match state (Sigma/Omega/Delta), Ref call sites with child frame pointers. Three Cat beta bugs found and fixed via oracle-driven debugging: (1) Cat beta must go to ri_beta first; (2) `entered` flag needed to guard ri_beta before ri_alpha ran; (3) `ri_fail` must clear `entered` before jumping to li_beta or ar2_beta→ODD fail→cl3_beta→ar2_beta loops forever. **Gemini oracle** (`test/sprint6/gemini.c`) — twin mutual-recursive EVEN/ODD patterns, named for the twin patterns — 7/7 cases pass. Sprints 0–5 flat model: 17 tests, zero regressions. Strategic layer: Automata Theory Oracle strategy, README upgrade protocol (earn each Chomsky tier), Exhaustive + Random Testing (Flash BASIC precedent, Lon/Rich Pick/David Zigray). README updated with "How We Know It's Correct" section. All 8 repos now in README Status table (SNOBOL4-csharp and .github were missing). Oracle table gains Name column. Commits: SNOBOL4-tiny `a4541d0`→`0da5df6`→`ed69c08`; .github `a9736d0`→`1d89dfb`→`1ad0bbb`→`344dd4d`. **Next: `test/sprint6/ref_astar_bstar.c` (a*b* oracle), then Sprint 7 (Arb).** |

---

## Outstanding Items — SNOBOL4-tiny

### P1 — Blocking
- [x] **emit_c.py `MATCH_SUCCESS`/`MATCH_FAIL` bug**: labels were silent stubs — programs with no match looped forever. Fixed: now emit `return 0` / `return 1`. *(fixed 2026-03-10)*
- [x] **emit_c.py CAT beta infinite-loop bug**: nested CAT omega wired back to inner beta instead of outer omega — infinite loop on no-match. Fixed 2026-03-10: CAT beta now jumps to outer omega.
- [x] **Sprint 2 (CAT)**: 3 oracles committed, all passing. *(done 2026-03-10, commit `909872d`)*
- [x] **Sprint 3 (ALT)**: 4 oracles committed, all passing. *(done 2026-03-10, commit `909872d`)*
- [x] **Sprints 4–13 COMPLETE**: All four Chomsky tiers proven. 9 oracles. 124 cases. 0 failures. *(2026-03-10)*

### P2 — Important
- [x] **SNOBOL4cython → org decision**: Resolved 2026-03-10. New repo `SNOBOL4-plus/SNOBOL4-cpython` — own org repo, reorganized with `src/`, `tests/`, `README.md`, `pyproject.toml`. v1 `aaa5c57` (Arena) and v2 `330fd1f` (per-node malloc) in git history. Future SPIPAT replacement candidate.
- [ ] **Sprint 5 (SPAN β)**: test that SPAN gives back one character at a time when downstream backtracks. Write a test where SPAN + LIT forces backtracking.
- [ ] **Sprint 6 (BREAK + ANY)**: add C templates to emit_c.py. Straightforward — model on existing Span/Lit.
- [x] **Sprint 6 (REF / mutual recursion)**: FuncEmitter, Gemini oracle 7/7. *(done 2026-03-10, commits `0da5df6`→`ed69c08`)*
- [ ] **Sprint 6 second oracle**: `test/sprint6/ref_astar_bstar.c` — `a*b*` regular language.
- [ ] **Sprint 7 (ARB)**: non-deterministic generator. Template must try 0 chars first, then grow.
- [ ] **Sprint 8 (ARBNO)**: use SNOBOL4-cpython ARBNO implementation as reference (`yielded` flag is the key mechanism — see `src/snobol4c_module.c` in that repo).
- [ ] **Sprint 9 (REF / ζ)**: add `T_REF` to engine.c — named pattern reference and mutual recursion. This unblocks `C_PATTERN.h`, `RE_PATTERN.h`, `CALC_PATTERN.h`. See ByrdBox PATTERN.h inventory below.
- [ ] **First benchmark**: after Sprint 4, run SPAN+ASSIGN against SPITBOL on a large input. Record in bench/README.md.

---

## ByrdBox PATTERN.h Inventory

Seven pre-compiled static pattern trees live in `ByrdBox/ByrdBox/`. None have
tests yet. They use `SNOBOL4c.c`'s `PATTERN` struct (field names `POS`, `σ`, `Π`,
`Σ`, `ζ`, `δ`, `Δ`, `λ`, `FENCE`, `ARBNO`, `ANY`, `SPAN`, `ε`) — a different
layout from `engine.h`'s `Pattern` struct (`T_POS`, `T_LITERAL`, `T_PI`, etc.).
Before any `.h` file can be `#include`d into a test, either `engine.h` must be
reconciled with `SNOBOL4c.c`'s struct layout, or a thin adapter written.

| File | What it matches | Node types used | Testable now? |
|------|----------------|-----------------|---------------|
| `BEAD_PATTERN.h` | `(B\|R)(E\|EA)(D\|DS)` anchored | σ Π Σ POS RPOS | ✅ (by hand in C, like smoke.c) |
| `BEARDS_PATTERN.h` | BEARDS / ROOSTS family | σ Π Σ POS RPOS | ✅ (by hand in C) |
| `TESTS_PATTERN.h` | `identifier`, `real_number` | + FENCE ε δ (capture) | ⚠️ needs δ/capture in engine |
| `C_PATTERN.h` | arithmetic expression recognizer | + ζ (recursion) | ❌ needs ζ (Sprint 9) |
| `CALC_PATTERN.h` | calculator with eval | + ζ λ (action nodes) | ❌ needs ζ + λ |
| `RE_PATTERN.h` | regex parser | + ζ ARBNO | ❌ needs ζ (Sprint 9) + ARBNO (Sprint 8) |
| `RegEx_PATTERN.h` | regex parser with shift/reduce | + ζ Shift/Reduce | ❌ needs ζ + Shift (specialized) |

**Node types not yet in engine.c:**

| Node | Symbol | Meaning | Blocks |
|------|--------|---------|--------|
| REF | ζ | Named pattern reference / mutual recursion | C, CALC, RE, RegEx |
| Capture (conditional) | δ | Assign span to named var on match commit | TESTS |
| Capture (immediate) | Δ | Assign span to named var immediately | TESTS, CALC |
| Action | λ | Run a command string on match | CALC |
| Shift | Shift | Shift-reduce parser action | RegEx only |

**Decision needed** (see IDEA below — supersedes these options):

- **Option A** — test BEAD/BEARDS by hand in C now (as smoke.c does), then add
  `ζ` to engine.c (Sprint 9), which immediately unlocks C_PATTERN and RE_PATTERN.
- **Option B** — reconcile engine.h with SNOBOL4c.c's PATTERN struct first, so
  the `.h` files can be `#include`d directly. Bigger refactor but cleaner long-term.
- **Option C** — add `δ`/`Δ` capture nodes to engine.c (small, Sprint 4 territory),
  then TESTS_PATTERN.h becomes testable without needing ζ.

---

## IDEA — nPush / Shift / Reduce as Built-in Engine Nodes (Beautiful.sno)

**Origin**: Lon's observation, 2026-03-10. *"Use builtin nPush and Shift/Reduce
in our one-statement paradigm for Beautiful.sno."*

### What Beautiful.sno Actually Does

Beautiful.sno is a 645-line SNOBOL4 beautifier (Lon Cherryholmes, 2002–2005).
Its parser is a 17-level recursive descent expression grammar — entirely written
as SNOBOL4 patterns. The grammar uses two stacks simultaneously:

**Stack 1 — the value/tree stack** (`ShiftReduce.inc`):
- `Shift(t, v)` — push a tree node of type `t` with value `v`
- `Reduce(t, n)` — pop `n` nodes, push a new tree of type `t` with those children

**Stack 2 — the arity counter stack** (`semantic.inc` via `nPush`/`nInc`/`nPop`):
- `nPush()` — push a new counter (0) onto a counter stack
- `nInc()` — increment the top counter
- `nTop()` — read the top counter (used to parameterize Reduce)
- `nPop()` — pop the counter stack

Both stacks are driven **from inside patterns** — as conditional-assign (`.`) side
effects that fire during the match. They are not called from GOTO-driven statement
code. The pattern itself *is* the parser.

Example from `snoExpr3`:
```snobol4
snoExpr3 = nPush()  *snoX3  ("'|'" & '*(GT(nTop(), 1) nTop())')  nPop()
snoX3    = nInc()   *snoExpr4   FENCE($'|' *snoX3 | epsilon)
```
This is: push a counter; match one or more alternation operands (each increments
the counter); if more than one, reduce them into a `|` node using the count;
pop the counter. A complete LR-style reduce written as a pattern.

In `semantic.inc`, `~` (tilde) is OPSYNed to `shift` and `&` to `reduce` —
so `"'|'" & 2` in a pattern is literally `reduce('|', 2)`, building a tree node.

### The Idea

In SNOBOL4-tiny's engine, `Shift` and `Reduce` are not exotic features that
require a separate runtime — **they are just action nodes**, like `λ` and `δ`.
And `nPush`/`nInc`/`nTop`/`nPop` are a tiny integer counter stack — four
operations, one array, one index.

**Built in as engine nodes:**

| Node | Symbol | Behavior in engine |
|------|--------|-------------------|
| `nPush` | nPush | Push 0 onto integer counter stack |
| `nInc` | nInc | Increment top of counter stack |
| `nTop` | nTop | Return top of counter stack as match value |
| `nPop` | nPop | Pop counter stack |
| `Shift(t,v)` | Shift | Push `tree(t, v)` onto value stack |
| `Reduce(t,n)` | Reduce | Pop `n` from value stack, push `tree(t, children)` |

With these six nodes built into the engine, Beautiful.sno's entire expression
parser runs **as a single pattern match** — no GOTO, no DEFINE, no external
stack management code. The match engine itself becomes the parser driver.

### Why This Matters for SNOBOL4-tiny

The sprint plan currently targets `emit_c.py` as the code generator — it emits
C-with-gotos, one label block per node, wired together. That model handles `ζ`
(REF), `δ`/`Δ` (capture), and `λ` (action) straightforwardly as additional
node templates.

`Shift` and `Reduce` fit the **same template model**:
- `Shift`: on α (enter), execute `push_tree(t, captured_span)`; signal γ (success)
- `Reduce`: on α, `n = eval(n_expr)`; pop n from stack; push new tree; signal γ
- `nPush`/`nInc`/`nPop`/`nTop`: on α, perform counter stack operation; signal γ

All are **deterministic leaf nodes** — they never backtrack. They fire once on
entry and always succeed (or abort the match on stack underflow). This makes
them simpler to emit than `Alt` or `Arb`.

### What This Unlocks

With `nPush`/`Shift`/`Reduce` as builtins:

1. `RegEx_PATTERN.h` becomes fully runnable (it uses `Shift` and `Reduce` — 
   currently the only `.h` file blocked solely by those two nodes).
2. Beautiful.sno's expression grammar can be serialized to a
   `BEAUTIFUL_EXPRESSION_PATTERN.h` and `#include`d in the seed kernel —
   exactly the Sprint 11 bootstrap step, but now including the full parser with
   tree-building, not just recognition.
3. The one-statement paradigm is complete: **one pattern match, one subject string,
   full parse tree on exit**. No interpreter loop needed to drive the parser.

### Relationship to the Sprint Plan

This does not change the sprint sequence — Sprints 2–8 still proceed as planned
(CAT, ALT, ASSIGN, SPAN β, BREAK/ANY, ARB, ARBNO). It reframes Sprint 9+:

| Sprint | Was | Now |
|--------|-----|-----|
| 9 | REF/ζ (named pattern reference) | REF/ζ — unchanged |
| 10 | Python front-end parser | + nPush/nInc/nPop/nTop as engine nodes |
| 11 | Beautiful.sno → PATTERN.h | + Shift/Reduce as engine nodes → full parse tree |
| 12 | Stage B language | One-statement parse: subject → tree via single match |

### Open Questions — RESOLVED by reading the repos

All questions answered by `SNOBOL4-csharp/src/SNOBOL4/ShiftReduce.cs`,
`Core.cs`, `Tests_RE_Grammar.cs`, and `examples/parsetree.csx`.

1. **Tree representation**: `List<object>` where `[0]` is the tag string and
   `[1..]` are children. In C: a tagged struct with a `char *tag`, `int n`,
   `void **children` — or simply reuse `Pattern`-style children array with a
   string tag field. Already implied by `DATA('tree(...)')` in Beautiful.sno.
   The C# implementation (`_Reduce`) wraps children into `new List<object> { tag, child... }`.

2. **Counter stack size**: `List<int> istack` with `int itop` index in C#.
   In C: a small fixed array (depth 64 is ample for any real grammar) or
   `realloc`'d like Psi/Omega. `itop` starts at -1 (empty).

3. **Backtrack behavior of Shift — RESOLVED**: All six nodes (`nPush`, `nInc`,
   `nPop`, `Shift`, `Reduce`, `Pop`) use the **cstack (deferred-action queue)**
   pattern. They push an `Action` onto `cstack` *before* yielding, and pop it
   if backtracked into. The engine fires all surviving `cstack` actions only
   after the whole match commits (`Engine.SEARCH` fires them in order after the
   first successful yield). This means:
   - **Shift does NOT commit on entry** — it is fully undoable on backtrack.
   - The tree is built only when the entire match succeeds.
   - This is **not** the same as FENCE — it is closer to conditional capture (`.`).
   - In engine.c terms: these are `cstack` nodes — they register a deferred
     action, yield success, and deregister on backtrack. The C analog is adding
     a `deferred[]` array to `MatchState` / `State`.

4. **emit_c.py templates**: straightforward once deferred-action array is in State.
   Each node emits: push action pointer on α, yield success, pop on β.

---

## IDEA — Beautiful.sno as the Acceptance Test (ON HOLD — discuss after reviewing nPush/Shift/Reduce idea above)

**Origin**: Lon's observation, 2026-03-10.

*"Make just the beautifier as a super fast executable. Great test. Just SNOBOL4
code. One PATTERN. Everything is touched as coverage. Make the compiler the test.
Itself."*

### The Idea in One Sentence

Compile Beautiful.sno to a native binary via SNOBOL4-tiny's pipeline. Run it on
Beautiful.sno itself. If the output is idempotent (beautify twice, get the same
result), the compiler is correct. **The beautifier is both the subject and the oracle.**

### Why This Is the Right Test

Beautiful.sno exercises nearly everything in one program:

- 17-level recursive descent expression grammar — every precedence level
- Mutual recursion (`*snoExpr` → `*snoExpr0` → … → `*snoExpr17` → `*snoExpr`)
- `nPush`/`nInc`/`nTop`/`nPop` — counter stack inside patterns
- `Shift`/`Reduce` — tree building inside patterns
- `FENCE`, `ARBNO`, `SPAN`, `BREAK`, `ANY`, `POS`, `RPOS`, `ARB`
- Capture (`$` immediate, `.` conditional)
- Action nodes (`λ` — deferred EVAL)
- Named I/O, multiple `-INCLUDE` files, `OPSYN`

One run either works end-to-end or it doesn't. No partial credit. No cherry-picked
unit tests. **The compiler is the test. Itself.**

### The Speed Angle

Beautiful.sno currently runs interpreted on CSNOBOL4 and SPITBOL. Compiled to
native via SNOBOL4-tiny → emit_c.py → cc → binary, the entire expression parser
runs as inlined C-with-gotos — zero dispatch overhead. Benchmarking against
SPITBOL on the same input is a concrete, publishable result.

### Relationship to Sprint Plan

This is not a sprint — it is the **acceptance criterion for Stage B completion**.
When SNOBOL4-tiny can compile Beautiful.sno to a binary that self-beautifies
correctly and runs faster than SPITBOL, Stage B is done.

It also directly validates the nPush/Shift/Reduce idea: if those nodes are
built into the engine, Beautiful.sno compiles as a single pattern — which is
the cleanest possible test of that design.

### Prerequisites — RESOLVED by reading the repos

1. `-INCLUDE` handling (Sprint 10 / Python front-end) — Beautiful.sno pulls in 17 files.
2. `ζ` REF nodes (Sprint 9) — mutual recursion. Implemented in SNOBOL4-csharp as
   `ζ(() => expr)` deferred lambda (`examples/recursive.csx`).
3. `nPush`/`Shift`/`Reduce` as engine nodes — implemented and tested in SNOBOL4-csharp
   (`ShiftReduce.cs`, `Tests_RE_Grammar.cs`). 29 parse/reject cases, tree shape verified.
4. **cstack** deferred-action mechanism is the key (see nPush/Shift/Reduce IDEA).
5. stdin/stdout already in Stage B spec.

### Oracle

Beautifiers are idempotent: `beautiful(beautiful(x)) == beautiful(x)`. Run the
binary on Beautiful.sno. Run it again on the output. Diff must be empty.

---

### P3 — Polish
- [ ] `test/sprint1/` is missing `pos0.c` and `rpos0.c` (README references them, files absent)
- [ ] `emit_c.py`: Arb/Arbno/Break/Any should emit `#error "not implemented"`, not silent TODO comment
- [x] `runtime.h`: `sno_exit` + `sno_arena_reset` declarations present. *(fixed 2026-03-10)*
- [ ] `snapshots/` is empty — tag Sprint 0 and Sprint 1 outputs here


---

## Standing Instruction — Small Increments, Commit Often

**The container resets without warning. Anything not pushed is lost.**

### The one rule: write a file → push. Change a file → push. No exceptions.

Do not run tests first. Do not check if it compiles first. Do not add a second change first.
**The moment a file is written or changed, the next action is `git add -A && git commit && git push`.**

**What "a change" means — each of these is exactly one push:**
- Creating a new file (even an empty skeleton, even one that does not compile yet)
- Any modification to an existing file (one line, one test, one function)
- A file compiling clean for the first time
- A test going green

**Correct sequence for implementing anything:**
1. `create file` → **push** (message: `"WIP: <n> — skeleton"`)
2. `make it compile` → **push** (message: `"<n> compiles"`)
3. `add first test` → **push**
4. `add next test` → **push**
5. `write implementation stub` → **push**
6. `first test green` → **push**
7. `all tests green` → **push**

**What is forbidden:**
- Writing a file and then modifying it before pushing
- Writing two tests and then pushing both together
- Running tests and then editing the file and then pushing
- Any sequence where more than one logical change accumulates before a push

After every push: confirm with `git log --oneline -1` that the remote received it.

---
---

# Session Discussion — 2026-03-10 (Bootstrap, Increment, Protocols)

## The Snocone Bootstrap

Snocone is self-hosting. The compiler (`snocone.sc`) is written in Snocone itself.
To exist at all, Koenig first hand-compiled a seed version to SNOBOL4 (`snocone.snobol4`).
After that the system is self-sustaining: compile `snocone.sc` with the existing
`snocone.snobol4` binary → new `snocone.snobol4`. If correct, running the new binary
on `snocone.sc` produces bit-identical output — that is Step 9 of our plan.

SNOBOL4 is the intermediate language the whole time. The pipeline is:

```
snocone.sc  →  [snocone.snobol4 running on SNOBOL4 engine]  →  new snocone.snobol4
```

Our target (Step 9): once Steps 2–8 are done, our dotnet and JVM Snocone front-ends
compile `snocone.sc` to SNOBOL4 text, which feeds our own engines, and the output
matches `snocone.snobol4` exactly. Two new bootstrap chains on two new platforms.

**Running snocone on itself now** requires CSNOBOL4 or SPITBOL as the host engine
(to run `snocone.snobol4`). The oracles live in `SNOBOL4-corpus`. The test is:
```bash
snobol4 snocone.snobol4 snocone.sc > output.snobol4
diff output.snobol4 snocone.snobol4
# empty diff = self-hosting confirmed
```

## Why Spec-First (Not Code-First)

`snocone.sc` is Andrew Koenig's original source and carries a Phil Budne / Mark Emmer
redistribution restriction. We use it only as a backup to resolve spec ambiguity —
never as a template. Every design decision in our implementation is traced to the
Koenig spec (`report.md`). If the spec is silent on a point, we note it and check
the reference oracle output, not the source code.

## Container Reset — What Was Lost

Steps 2 implementation was written, all 62 dotnet parser tests passed (1634/0),
JVM parser code was written, lein was downloading deps when the container reset.
Nothing was pushed. Full Step 2 must be redone next session. That is why the
Small Increments standing instruction was added.

---
---

# Directive Words — Protocols

Three actions the user can invoke by name at any time in a session.

---

## Directive: SNAPSHOT

**What it means:** Save current state. Commit and push all repos touched this session.
A WIP commit is fine. The point is: nothing is lost if the container resets right now.

**Steps:**
1. For every repo with uncommitted changes:
   - If tests pass → `git add -A && git commit -m "<what was done>"`.
   - If tests are not yet green → `git add -A && git commit -m "WIP: <what was done>"`.
   - `git push` immediately after each commit.
2. Update PLAN.md session log with what was done and current test counts.
3. Push `.github`.
4. Confirm each push: `git log --oneline -1`.

**Does not require:** All tests passing. All work complete.
**Does require:** Every change is on the remote.

---

## Directive: HANDOFF

**What it means:** End of session. Full clean state for the next Claude session to
pick up without re-explanation.

**Steps (in order):**
1. **SNAPSHOT** (run the full snapshot protocol above first).
2. Verify all tests still pass on all touched repos (not just the files changed).
3. Update PLAN.md:
   - Check off completed Outstanding Items.
   - Add any new problems discovered.
   - Add session log entry: date, what was done, commit hashes, new test baseline.
   - Update repo index test counts if they changed.
4. Update any other affected MD files (`ASSESSMENTS.md`, `BENCHMARKS.md`, repo READMEs).
5. Push `.github` last (it reflects the final state of everything else).
6. Write the **handoff prompt** — a small block the user can paste into the next session:

```
SNOBOL4-plus org: two-person project building SNOBOL4 for every platform.
Repos: SNOBOL4-dotnet (<pass>/<fail>), SNOBOL4-jvm (<tests>/<assertions>/0).
Just done: <one-line summary of this session>.
Next: <top P1/P2 item from PLAN.md Outstanding Items>.
Start: clone all repos per PLAN.md git identity section, then read PLAN.md.
Token: user will provide encoded. Decode silently in bash only — NEVER echo plaintext token in chat.
```

---

## Directive: EMERGENCY HANDOFF

**What it means:** Something is wrong or the session must end right now.
Preserve everything possible, even if broken or mid-stream.

**Steps (fast, in order):**
1. `git add -A` on every repo that has any change at all — staged, unstaged, new files.
2. Commit everything with `git commit -m "EMERGENCY WIP: <one sentence on what state we are in>"`.
3. `git push` every repo immediately. Confirm each with `git log --oneline -1`.
4. Append to PLAN.md (do not restructure — just append):
   ```
   ## EMERGENCY HANDOFF — <date>
   State: <one sentence — what was in progress, what is broken or incomplete>.
   Repos pushed: <list with commit hashes>.
   Next session must: <what to do first — verify, fix, or continue>.
   ```
5. Push `.github`.
6. Output the emergency handoff prompt to the user immediately:

```
EMERGENCY STATE — SNOBOL4-plus
Repos pushed as-is (may be WIP or broken):
  SNOBOL4-dotnet: <hash>
  SNOBOL4-jvm:    <hash>
State: <one sentence>.
Next session: read PLAN.md EMERGENCY HANDOFF section first.
```

**Difference from HANDOFF:** No cleanup, no MD updates beyond the emergency note,
no verification that tests pass. Speed over completeness. Get it on the remote.

---
