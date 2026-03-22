# FRONTEND-PROLOG.md — Tiny-Prolog Frontend (L3)

Tiny-Prolog is a frontend for snobol4x targeting the x64 ASM backend first,
then C and JVM/NET once the design is proven.

*Session state → TINY.md. Milestone dashboard → PLAN.md §Prolog Frontend.*

---

## Design Philosophy

Prolog is a first-class citizen of the IR — not a guest of SNOBOL4.
New term nodes, new clause nodes, new unification primitive. The emitter
gets new `case` branches that lower to Byrd box (α/β/γ/ω) sequences.
The backends (x64, C, JVM, NET) see only Byrd box output — same as today.

**No kludge to SNOBOL4 primitives.** Whatever Prolog needs, it gets as a
native node type with native lowering rules.

---

## Why Prolog fits the Byrd Box model

Byrd invented the four-port box *for Prolog* — snobol4ever already uses it
for SNOBOL4 patterns. The symmetry is exact:

| Port | SNOBOL4 pattern | Prolog SLD resolution |
|------|-----------------|-----------------------|
| α    | proceed         | try next clause head  |
| β    | resume          | retry on backtrack    |
| γ    | succeed         | head unified, enter body |
| ω    | fail            | all clauses exhausted |

Cut (`!`) maps to FENCE: β becomes unreachable. No new mechanism needed.

---

## Practical Subset (target)

**In scope:**
- Horn clauses: `head :- body.`
- Unification: `foo(X, bar(X)).`
- Arithmetic via `is/2`: `+`, `-`, `*`, `//`, `mod`
- Comparison: `</2`, `>/2`, `=:=/2`, `=\=/2`, `=</2`, `>=/2`
- Structural equality: `=/2`, `\=/2`
- Cut: `!`
- Lists: `[H|T]` notation, `[a,b,c]` sugar
- Builtins: `write/1`, `writeln/1`, `nl/0`, `read/1`
- Meta: `call/1`, `call/N` (N≤4)
- Term inspection: `functor/3`, `arg/3`, `=../2` (univ)
- Type tests: `atom/1`, `integer/1`, `float/1`, `var/1`, `nonvar/1`, `compound/1`
- Control: `true/0`, `fail/0`, `halt/0`, `halt/1`

**Deferred (post-corpus):**
- `assert/retract` (dynamic DB)
- `setof/3`, `bagof/3`, `findall/3`
- `module/2`
- Constraint logic (CLP)
- DCG notation
- Exception handling (`throw/catch`)

---

## Term Representation — `TERM_t`

New type, independent of `DESCR_t`. Lives in `src/frontend/prolog/term.h`.

```c
typedef enum {
    TT_ATOM,      /* 'foo'  — interned string index        */
    TT_VAR,       /* X      — slot index into env frame    */
    TT_COMPOUND,  /* f(a,b) — functor + arity + args[]     */
    TT_INT,       /* 42                                    */
    TT_FLOAT,     /* 3.14                                  */
    TT_REF        /* bound variable — pointer to target    */
} TermTag;

typedef struct Term {
    TermTag tag;
    union {
        int         atom_id;
        int         var_slot;   /* compile-time slot in env DATA block */
        struct {
            int          functor; /* atom_id of functor name */
            int          arity;
            struct Term **args;
        } compound;
        long        ival;
        double      fval;
        struct Term *ref;       /* TT_REF: dereference chain */
    };
} Term;
```

List `[H|T]` sugar: lowered to `compound{ functor='.', arity=2, args=[H,T] }`.
Nil `[]`: `atom_id` for the empty-list atom.

---

## Environment Frame (per-invocation DATA block)

Each clause gets a compile-time `EnvLayout` — a fixed-size array of `Term*` slots,
one per distinct variable. The env frame IS the T2 DATA block for Prolog clauses:

```c
typedef struct EnvLayout {
    int   n_vars;           /* number of distinct variables in clause  */
    int   n_args;           /* arity of the head predicate             */
    int   trail_mark_slot;  /* slot index reserved for trail mark      */
} EnvLayout;
```

At runtime, `r12` points at the live env frame (T2 convention). Variable slot `k`
is at `[r12 + k*8]`.

---

## Trail

One global trail per execution. The ω port unwinds to the mark saved at clause entry.

```c
typedef struct Trail {
    Term  ***stack;    /* array of pointers to bound Term* slots */
    int      top;
    int      capacity;
} Trail;

void trail_push(Trail *t, Term **slot);    /* called on every binding */
void trail_unwind(Trail *t, int mark);     /* called at ω port        */
```

LIFO discipline matches backtracking exactly. No GC needed.

---

## New IR Node Types

These live between the Prolog parser and the Byrd box emitter.
The backends never see them — fully lowered before emission.

| Node | Meaning |
|------|---------|
| `PL_CLAUSE(head, body[])` | One Prolog clause |
| `PL_CHOICE(clauses[])` | All clauses for one functor/arity |
| `PL_CALL(functor, arity, args[])` | Body goal — call another predicate |
| `PL_UNIFY(t1, t2)` | Unification; ω on failure |
| `PL_CUT` | Commit — seal β of enclosing choice |
| `PL_TRAIL_MARK` | Save trail top into env frame slot |
| `PL_TRAIL_UNWIND` | Restore trail to saved mark |
| `PL_TERM_LOAD(slot)` | Load variable from env frame |
| `PL_TERM_STORE(slot)` | Store binding + push trail |
| `PL_IS(slot, arith_expr)` | Arithmetic eval; bind; ω if non-numeric |

---

## Byrd Box Wiring — Clause Selection

For `foo/1` with two clauses, the emitter lays out:

```
P_FOO_α:
    PL_TRAIL_MARK              ; save trail.top into env[trail_mark_slot]
    PL_UNIFY(env[0], ATOM_a)   ; try clause 1 head; failure → P_FOO_β
    <body of clause 1>
    jmp [ret_γ]

P_FOO_β:
    PL_TRAIL_UNWIND            ; undo bindings from clause 1 attempt
    PL_TRAIL_MARK              ; fresh mark for clause 2
    PL_UNIFY(env[0], ATOM_b)   ; try clause 2 head; failure → P_FOO_ω
    <body of clause 2>
    jmp [ret_γ]

P_FOO_ω:
    PL_TRAIL_UNWIND
    jmp [ret_ω]
```

Cut (`!`) seals the β slot so it jumps directly to ω — exactly how FENCE works
in the pattern engine. No new mechanism needed.

---

## Source Layout

```
src/frontend/prolog/
    pl_lex.c          Hand-rolled lexer
    pl_parse.c        Hand-rolled recursive-descent parser -> ClauseAST
    pl_lower.c        ClauseAST -> PL_* IR nodes; variable slot assignment
    pl_emit.c         PL_* nodes -> Byrd box a/b/g/w emission
    pl_unify.c        Runtime unify() + trail_push/unwind
    pl_atom.c         Atom interning table
    pl_builtin.c      write/nl/read/functor/arg/=.. etc.
    term.h            TERM_t definition
    pl_ir.h           PL_* node type definitions
    pl_runtime.h      Trail, EnvLayout, entry-point declarations
```

Build: `-pl` flag selects this frontend. Composes with `-asm`/`-c`/`-jvm`/`-net`.

---

## Driver Flags

```
snobol4x -pl -asm  foo.pl    ->  foo.s   (x64 NASM)
snobol4x -pl -c    foo.pl    ->  foo.c   (C backend)
```

---

## Corpus — Prolog Ladder

Mirrors the SNOBOL4 10-rung ladder. Acceptance test for each milestone.

```
Rung 1:  hello       write('hello'), nl.
Rung 2:  facts       deterministic fact lookup, no backtracking
Rung 3:  unify       head unification, compound terms
Rung 4:  arith       is/2, integer arithmetic
Rung 5:  backtrack   member/2 — first backtracking program
Rung 6:  lists       append/3, length/2, reverse/2
Rung 7:  cut         cut in member/2 and deterministic predicates
Rung 8:  recursion   fibonacci/2, factorial/2
Rung 9:  builtins    write/read/functor/arg/=..
Rung 10: programs    word puzzle solver (Lon's programs)
```
