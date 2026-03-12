# JCON.md — Jcon Source Architecture & SNOBOL4 Byrd Box Port Guide

> **Read this when**: starting Phase 0 (shared IR), Phase 1 (JVM backend),
> or Phase 2 (MSIL backend). Not needed for Sprint 20 (tiny T_CAPTURE work).

---

## What Jcon Is

**Jcon** — Gregg Townsend + Todd Proebsting, University of Arizona, 1999.  
Source: `https://github.com/proebsting/jcon` — public domain, 1,196 commits.  
Language: Icon translator (written in Icon) + Java runtime (88 `.java` files).  
Paper: *"A New Implementation of the Icon Language"* (`https://www2.cs.arizona.edu/icon/jcon/impl.pdf`)

Jcon is the exact artifact promised in Proebsting's Byrd Box paper
(*"Simple Translation of Goal-Directed Evaluation"*, 1996): a working
Icon → JVM bytecode compiler built on the four-port Byrd Box model.

The paper's final sentence: *"These new techniques will be the basis for a new Icon
compiler that will translate Icon to Java bytecodes."* — Proebsting, 1996.
Jcon is that compiler. SNOBOL4-plus is doing the same for SNOBOL4, in 2026.

---

## Repository Layout

```
jcon/
├── tran/           ← The translator (written in Icon)
│   ├── ir.icn          ← IR record types (48 lines) — THE VOCABULARY
│   ├── irgen.icn       ← AST → IR chunks (1,559 lines) — THE FOUR PORTS
│   ├── gen_bc.icn      ← IR → JVM bytecode (2,038 lines) — THE EMITTER
│   ├── bytecode.icn    ← .class file serializer (1,770 lines) — REPLACED BY ASM
│   ├── parse.icn       ← Icon parser
│   ├── lexer.icn       ← Icon lexer
│   ├── optimize.icn    ← IR optimizer
│   └── linker.icn      ← Multi-file linker
├── jcon/           ← The Java runtime (88 .java files)
│   ├── vDescriptor.java    ← Abstract base: null return = failure
│   ├── vValue.java         ← Concrete values (strings, ints, reals, ...)
│   ├── vVariable.java      ← Assignable lvalues
│   ├── vClosure.java       ← Suspended generator: PC field + saved locals
│   ├── vProc*.java         ← Compiled procedures (vProc0..vProc9, vProcV)
│   └── f*.java             ← Built-in functions (fIO, fScan, fList, ...)
├── bmark/          ← Benchmarks
├── demo/           ← Demo programs
└── test/           ← Test suite
```

---

## The Translator Pipeline

```
Icon source
    ↓  lexer.icn + parse.icn
AST (a_* record types)
    ↓  irgen.icn
IR chunks (ir_chunk records, four ports per node)
    ↓  optimize.icn
Optimized IR
    ↓  gen_bc.icn
JVM bytecode objects (j_* records)
    ↓  bytecode.icn (j_writer_j_ClassFile)
.class files
```

---

## `ir.icn` — The IR Vocabulary (48 lines)

This is the complete IR. Memorize it.

```icon
record ir_chunk(label, insnList)        ← one basic block: label + instruction list

record ir_Tmp(name)                     ← temporary variable (JVM local slot)
record ir_TmpLabel(name)                ← temporary label variable (int local for indirect goto)
record ir_Label(value)                  ← a basic block label

record ir_Var(coord, lhs, name)         ← named variable reference
record ir_Key(coord, lhs, name, failLabel)  ← keyword (&subject, &pos, ...)
record ir_IntLit / ir_RealLit / ir_StrLit / ir_CsetLit  ← literals

record ir_Move(coord, lhs, rhs)
record ir_MoveLabel(coord, lhs, label)  ← save a label into a TmpLabel (for indirect goto)
record ir_Deref(coord, lhs, value)
record ir_Assign(coord, target, value)

record ir_Goto(coord, targetLabel)      ← unconditional goto
record ir_IndirectGoto(coord, targetTmpLabel)  ← goto saved-in-variable (for Alt backtrack)
record ir_Succeed(coord, expr, resumeLabel)    ← yield a value (resumeLabel = null → return)
record ir_Fail(coord)                   ← return null (failure)
record ir_Unreachable(coord)

record ir_OpFunction / ir_Call / ir_ResumeValue / ir_MakeList
record ir_EnterInit(coord, startLabel)  ← static initializer guard
record ir_Create / ir_CoRet / ir_CoFail ← co-expressions (not needed for SNOBOL4 patterns)
record ir_ScanSwap(coord, subject, pos) ← string scanning environment swap
```

**SNOBOL4 Byrd Box subset** (the only nodes we need for pattern compilation):

```
ir_chunk, ir_Label, ir_Tmp, ir_TmpLabel,
ir_Goto, ir_IndirectGoto,
ir_Succeed (= match success, advance cursor),
ir_Fail    (= match failure, return null),
ir_Move, ir_MoveLabel,
ir_IntLit, ir_StrLit
```

---

## `irgen.icn` — The Four-Port Encoding (key patterns)

Every AST node `p` gets four labels via `ir_init(p)`:
- `p.ir.start`   — α: initial entry
- `p.ir.resume`  — β: re-entry on backtrack
- `p.ir.success` — γ: inherited from parent (where to go on success)
- `p.ir.failure` — ω: inherited from parent (where to go on failure)

These are wired with `ir_Goto`. Example — `ir_a_Alt` (alternation `e1 | e2`):

```icon
suspend ir_chunk(p.ir.start,  [ ir_Goto(c, p.eList[1].ir.start) ])
/bounded & suspend ir_chunk(p.ir.resume, [ ir_IndirectGoto(c, t) ])

# each branch on success: save resume label, goto parent success
suspend ir_chunk(p.eList[i].ir.success, [
    ir_MoveLabel(c, t, p.eList[i].ir.resume),
    ir_Goto(c, p.ir.success)
])
# each branch on failure: try next branch
suspend ir_chunk(p.eList[i].ir.failure, [ir_Goto(c, p.eList[i+1].ir.start)])

# last branch failure → parent failure
suspend ir_chunk(p.eList[-1].ir.failure, [ ir_Goto(c, p.ir.failure)])
```

The `ir_IndirectGoto` + `ir_TmpLabel` + `ir_MoveLabel` trio IS the backtracking
mechanism. No stack. No continuation. Just a saved label + indirect goto.

---

## `gen_bc.icn` — IR → JVM (key mechanisms)

### Label resolution
```icon
bc_ir2bc_labels[ir_label] := j_label()   # j_label() = ASM Label object
```
Every `ir_Label` gets exactly one `j_label()`. Forward references work because
ASM resolves labels at `visitMaxs` time.

### Transfer (goto)
```icon
procedure bc_transfer_to(s, p)
    case type(p) of {
    "ir_Label":    put(s, j_goto_w(\bc_ir2bc_labels[p]))
    "ir_TmpLabel": bc_gen_rval(s, p)         # push int value
                   put(s, j_goto_w(bc_switch_label))  # → tableswitch
    }
end
```

### Resumable function dispatch (the computed-goto replacement)
```icon
switch := j_tableswitch(0, deflab, 1, *bc_indirect_targets+1, ...)
put(s, bc_switch_label)
put(s, switch)
```
At function entry, if `save_restore_flag` (function has a resume point):
```
aload_0 → getfield PC → tableswitch → Label_1 / Label_2 / ...
```
This is exactly the `switch(entry) { case ALPHA: ... case BETA: ... }` pattern
from `test_sno_3.c` — JVM edition.

### Failure = null return
```icon
procedure bc_gen_ir_Fail(s, p)
    put(s, j_aconst_null())
    put(s, j_areturn())
end
```
`null` is failure. Every generated method returns `vDescriptor` (or null).
Our equivalent: method returns `int len` (-1 = failure) or two int locals.

---

## SNOBOL4 Port — What Changes, What Stays

### What stays (same concept, different syntax)
| Jcon | SNOBOL4 port |
|------|--------------|
| `ir_chunk` / four ports | same |
| `ir_Goto` / `ir_IndirectGoto` | same |
| `ir_Succeed` | cursor advance + goto parent success |
| `ir_Fail` | `len = -1` + goto parent failure |
| `ir_TmpLabel` + `ir_MoveLabel` | Arbno/Alt backtrack PC save |
| `tableswitch` on `entry` int | named pattern `α/β` dispatch |
| `j_label()` objects wired by ASM | ASM `Label` objects — identical |

### What we don't need
- `vDescriptor` / `vClosure` / `vProc` hierarchy — our value is `str_t`
- Co-expressions (`ir_Create/CoRet/CoFail`) — SNOBOL4 has no coroutines
- The full `bytecode.icn` serializer — ASM handles `.class` format
- `ir_Key` / `ir_ScanSwap` — SNOBOL4 string scanning uses globals Σ/Δ/Ω

### What is simpler
- No dynamic typing — subject is `char[]`, cursor is `int`
- No garbage collection — patterns are compiled code, not heap objects
- Failure = `len == -1` — not `null` object return through JVM dispatch
- Named patterns ARE methods, not `vProc` subclasses

### What is harder
- Pattern composition nodes (`Seq`/`Alt`/`Arbno`) must wire four ports
  for each primitive too — Jcon's IR handles this uniformly; ours must too
- `Arbno` needs a stack (array of saved cursors) — `int[]` local, same as
  `_23_a[64]` in `test_sno_2.c`

---

## Phase 0 — Python IR Dataclasses

Port `ir.icn` to Python. Target: `SNOBOL4-tiny/src/ir/byrd_ir.py` (~60 lines).

```python
from dataclasses import dataclass, field
from typing import Optional, List, Union

@dataclass
class Label:
    name: str

@dataclass
class TmpLabel:
    name: str

@dataclass
class Chunk:
    label: Label
    insns: List  # list of IR nodes

# Primitives
@dataclass
class Lit:     s: str
@dataclass
class Span:    charset: str
@dataclass
class Break:   charset: str
@dataclass
class Any:     charset: str
@dataclass
class Notany:  charset: str
@dataclass
class Pos:     n: int
@dataclass
class Rpos:    n: int

# Composition
@dataclass
class Seq:     left: object; right: object
@dataclass
class Alt:     left: object; right: object
@dataclass
class Arbno:   child: object
@dataclass
class Call:    name: str

# Match top-level
@dataclass
class Match:   subject: str; pattern: object

# Control
@dataclass
class Goto:       target: Label
@dataclass
class IndirectGoto: target: TmpLabel
@dataclass
class Succeed:    resume: Optional[Label] = None
@dataclass
class Fail:       pass
@dataclass
class MoveLabel:  dst: TmpLabel; src: Label
```

---

## Phase 1 — JVM Backend Sketch

Target: `SNOBOL4-tiny/src/codegen/emit_jvm.py`

```python
from org.objectweb.asm import ClassWriter, MethodVisitor, Label, Opcodes

class JvmEmitter:
    def emit_pattern(self, name: str, node) -> bytes:
        """Compile a pattern node to a .class file. Returns bytecode."""
        cw = ClassWriter(ClassWriter.COMPUTE_FRAMES)
        cw.visit(Opcodes.V11, Opcodes.ACC_PUBLIC, name, ...)
        mv = cw.visitMethod(Opcodes.ACC_PUBLIC, "match", "(II[C)I", ...)
        # ... emit four-port chunks ...
        mv.visitMaxs(0, 0)
        mv.visitEnd()
        return cw.toByteArray()

    def emit_chunk(self, mv, chunk: Chunk, label_map: dict):
        mv.visitLabel(label_map[chunk.label])
        for insn in chunk.insns:
            self.emit_insn(mv, insn, label_map)

    def emit_insn(self, mv, insn, label_map):
        match type(insn).__name__:
            case "Goto":
                mv.visitJumpInsn(Opcodes.GOTO, label_map[insn.target])
            case "IndirectGoto":
                # load int from local, tableswitch
                ...
            case "Fail":
                mv.visitInsn(Opcodes.ICONST_M1)   # len = -1
                mv.visitInsn(Opcodes.IRETURN)
            case "Succeed":
                # store advanced cursor, goto success label
                ...
```

ASM is available in SNOBOL4-jvm's classpath. For the Python emitter,
use `jython` or ship ASM as a subprocess — TBD.

**Alternative**: Emit Jasmin assembly (`.j` text files), run `jasmin` to
produce `.class`. Simpler for prototyping. Lower performance. Same IR.

---

## Phase 2 — MSIL Backend Sketch

Target: `SNOBOL4-tiny/src/codegen/emit_msil.py`

```python
import clr
from System.Reflection.Emit import AssemblyBuilder, TypeBuilder, MethodBuilder, ILGenerator, OpCodes, Label

class MsilEmitter:
    def emit_pattern(self, name: str, node) -> bytes:
        ab = AssemblyBuilder.DefineDynamicAssembly(...)
        mb = ab.DefineDynamicModule(name)
        tb = mb.DefineType(name, ...)
        method = tb.DefineMethod("match", ..., [int, int, char[]], int)
        il = method.GetILGenerator()
        # ... emit four-port chunks ...
        t = tb.CreateType()
        ab.Save(name + ".dll")

    def emit_insn(self, il, insn, label_map):
        match type(insn).__name__:
            case "Goto":
                il.Emit(OpCodes.Br, label_map[insn.target])
            case "Fail":
                il.Emit(OpCodes.Ldc_I4_M1)
                il.Emit(OpCodes.Ret)
            ...
```

**Alternative**: Emit `.il` text (MSIL assembly), run `ilasm` to produce `.dll`.
Same relationship as Jasmin → .class. Use for prototyping.

---

## Key Reference Files

| File | What to read |
|------|-------------|
| `/home/claude/jcon/tran/ir.icn` | Complete IR vocabulary (48 lines) |
| `/home/claude/jcon/tran/irgen.icn` | Four-port encoding for Alt, Seq, Arbno, Scan |
| `/home/claude/jcon/tran/gen_bc.icn` | IR → JVM: `bc_transfer_to`, `bc_gen_ir_Succeed/Fail`, `bc_nextval_code` (tableswitch) |
| `/home/claude/jcon/tran/bytecode.icn` | JVM opcode record types — reference for ASM equivalents |
| `/home/claude/ByrdBox/ByrdBox/test_sno_2.c` | Gold standard compiled C — what JVM/MSIL output must match in structure |
| `/home/claude/ByrdBox/ByrdBox/test_sno_3.c` | Named patterns as functions with α/β entry dispatch |
| `/home/claude/ByrdBox/ByrdBox/byrd_box.py` | Current Python Byrd Box implementation — Phase 0 starts here |
| Proebsting paper PDF | `/home/claude/ByrdBox/ByrdBox/Simple Translation of Goal Directed Evaluation.pdf` |
| Jcon paper | `https://www2.cs.arizona.edu/icon/jcon/impl.pdf` |

---

## The Commit Promise (do not lose this)

When `beautiful` compiles itself through `snoc` and `diff pass1.txt pass2.txt` is empty,
Claude Sonnet 4.6 writes the commit message. Recorded at `c5b3e99`.

That is Sprint 20. Phase 0/1/2 (Byrd Box JVM + MSIL) begin after Sprint 20
or in parallel if Sprint 20 is blocked.

---

*Created 2026-03-12, Session 15. Update when Phase 0 IR is implemented or
when Phase 1/2 emitters reach first smoke test.*

---

## Revised Box Architecture (Session 16, Lon)

> This supersedes the earlier design where temporaries were passed as
> allocated blocks into Byrd Box functions.

**Locals live inside the box.** Each box is self-contained: data + code
together. When `*X` fires at match time, the box is copied and the code
is relocated. That copy is the new instance's independent local storage.
Duplication is fine — fast, cache-hot.

**Memory layout** — two parallel linear sections:

```
DATA:  [ box0 | box1 | box2 | ... ]
TEXT:  [ box0 | box1 | box2 | ... ]
```

Box N in DATA corresponds to box N in TEXT. Sequential = cache-friendly.

**`byrd_ir.py` (Phase 0)** must carry `locals` inside the `ByrBox` node,
not as a separate allocation. `CopyRelocate` is the IR node for `*X`.
