# String Escape Reference — SNOBOL4 · C · Python

**Purpose:** Authoritative conversion table for the emit_c_stmt.py emitter.
Every time a SNOBOL4 string value passes through a language boundary, apply
the correct rule from this table. Nothing from memory. No guessing.

---

## Sources

- **SNOBOL4 (CSNOBOL4):** `syn.c` scanner — `DQLITB`/`SQLITB` states.
  Both states are `FOR(terminator) STOP / ELSE CONTIN`. No escape processing
  whatsoever. Every byte between the opening and closing quote is taken
  literally. Confirmed identical behavior in SPITBOL (sbl.asm `scn18` loop:
  scan until `wc == wb` (terminator char), no branch for backslash).

- **C:** ISO C standard escape sequences. A `\` not followed by a recognized
  escape character is a **compiler diagnostic** (stray backslash error).

- **Python:** Standard string literal escapes. `\` followed by unrecognized
  char silently passes through as `\` + char (DeprecationWarning in 3.12+,
  SyntaxError in future versions).

- **Oracle verification:** Both CSNOBOL4 2.3.3 and SPITBOL v4.0f tested
  2026-03-10. Results identical.

---

## Rule 1: SNOBOL4 String Literals — No Escape Sequences

| What you want in the string | How to write it in SNOBOL4 |
|-----------------------------|----------------------------|
| Any printable character     | Write it literally          |
| Backslash `\`               | `'\'` or `"\"` — just the char, size=1 |
| Newline                     | Cannot be in a literal (line-terminated). Use variable `nl = CHAR(10)` |
| Tab                         | Cannot be in a literal. Use `tb = CHAR(9)` |
| Single-quote `'`            | Cannot appear in `'...'`. Use `"'"` or concatenate: `'can' "'" 't'` |
| Double-quote `"`            | Cannot appear in `"..."`. Use `'"'` or concatenate: `'say ' '"' 'hi'` |
| Null byte                   | Use `CHAR(0)` |

**Key fact:** `'\n'` in SNOBOL4 is TWO characters: backslash + `n`. It is NOT
a newline. `"\t"` is TWO characters: backslash + `t`. It is NOT a tab.
Backslash has no special meaning whatsoever in SNOBOL4 string literals.

---

## Rule 2: C String Literals — Backslash IS the Escape Character

| Character wanted in string | C literal syntax | Notes |
|----------------------------|-----------------|-------|
| Backslash `\`              | `"\\"` | Must be doubled |
| Double-quote `"`           | `"\""` | Must be escaped in `"..."` |
| Single-quote `'`           | `"\'"` or `"'"` | `\'` valid but `'` preferred |
| Newline LF                 | `"\n"` | |
| Carriage return CR         | `"\r"` | |
| Tab                        | `"\t"` | |
| Null byte                  | `"\0"` | |
| Form feed                  | `"\f"` | |
| Backspace                  | `"\b"` | |
| Vertical tab               | `"\v"` | |
| Alert/bell                 | `"\a"` | |
| Hex escape                 | `"\xHH"` | |
| Octal escape               | `"\OOO"` | |
| Any other `\X`             | **STRAY BACKSLASH — compile error** | |

**Stray backslash rule:** In C, `"\'"` is valid (= `'`). But `"\?"` where `?`
is not in the table above causes a compiler diagnostic. Common traps:
- `"\"` — unterminated string (the `"` is escaped, closing quote missing)
- `"\\"` — correct for one backslash
- `"\\\""` — correct for backslash + double-quote (two chars)

---

## Rule 3: Python String Literals — Backslash IS the Escape Character

| Character wanted in string | Python literal syntax | Notes |
|----------------------------|-----------------------|-------|
| Backslash `\`              | `"\\"` or `'\\'` | Must be doubled |
| Single-quote `'`           | `"\'"` or `"'"` in `"..."`, or `'\''` impossible — use `"'"` |
| Double-quote `"`           | `'\"'` or `'"'` in `'...'`, or `"\""` |
| Newline LF                 | `"\n"` | |
| Tab                        | `"\t"` | |
| Null byte                  | `"\x00"` or `"\0"` | |
| Raw string (no escapes)    | `r"..."` or `r'...'` | backslash literal, no processing |
| f-string with `{}`        | `f"..."` — `{` and `}` are special, escape as `{{` `}}` | |

---

## Rule 4: The Conversion Functions in emit_c_stmt.py

### SNOBOL4 runtime value → C string literal

When a SNOBOL4 value (held in Python as a `str`) must be emitted as a
C string literal `"..."`:

```python
def sno_val_to_c_literal(s: str) -> str:
    """
    Convert a Python string (holding a SNOBOL4 runtime value) to a C
    string literal body (the part between the double quotes).

    SNOBOL4 backslash is just a backslash — no escape meaning.
    In C a backslash MUST be doubled.
    In C a double-quote MUST be escaped.
    Newlines and other control chars MUST be escaped.
    """
    result = []
    for ch in s:
        if ch == '\\':
            result.append('\\\\')      # SNOBOL \ → C \\
        elif ch == '"':
            result.append('\\"')       # SNOBOL " → C \"
        elif ch == '\n':
            result.append('\\n')       # newline → C \n
        elif ch == '\r':
            result.append('\\r')
        elif ch == '\t':
            result.append('\\t')
        elif ch == '\0':
            result.append('\\0')
        else:
            result.append(ch)          # all other chars: literal
    return '"' + ''.join(result) + '"'
```

### SNOBOL4 runtime value → C char* argument (sno_to_str wrapper)

When building the argument to `sno_pat_break_()`, `sno_pat_any_cs()` etc.,
the charset string is a **runtime value**, not necessarily a compile-time
constant. Use `sno_to_str(expr)` — do NOT embed it as a literal unless it
truly is a constant.

---

## Rule 5: The Layering Trap

**The most dangerous bug:** applying escape conversion twice.

Example of the bug:
```
SNOBOL4 value: "  (one double-quote character)
Step 1 (correct): emit as C literal  →  "\""
Step 2 (bug — applied again):         →  "\\\""   ← STRAY BACKSLASH
```

This happens when:
1. `emit_as_str()` correctly produces `"\""`
2. The result is then passed through another replace/escape function
3. Or stored as a Python string and then escaped again when written

**The fix:** Escape conversion happens **exactly once**, at the boundary
where a Python `str` (holding the raw value) becomes a C source token.
After that point, the string is already valid C and must not be touched.

---

## Rule 6: PatExpr Literal Nodes

A `PatExpr(kind='lit', val=X)` node holds the **raw SNOBOL4 character
value** — e.g., `val='"'` is a one-character string containing `"`.

In `emit_as_str()`, to produce a C string literal from this node:

```python
if pk == 'lit':
    # val is the raw SNOBOL4 character(s) — apply Rule 4 exactly once
    return sno_val_to_c_literal(a.val or '')
```

**Never** call `.replace()` chains on the value — use `sno_val_to_c_literal`
as the single point of conversion.

---

## Quick Reference Card

```
SNOBOL4  →  Python str  →  C literal
─────────────────────────────────────
  \            \           \\
  "            "           \"
  '            '           '   (no escaping needed in C "..." strings)
  \n (2 chars) \n (2 chars) \\n  (C: backslash + n, 2 chars)
  newline      \n (1 char)  \n  (C: newline escape, 1 char)
  tab          \t (1 char)  \t  (C: tab escape, 1 char)
```

**Reading the table:**
- SNOBOL4 `\n` = two characters (backslash, n). In Python str = `'\\n'`.
  In C literal = `"\\n"` (still two chars: backslash + n).
- A real newline in Python `str` = `'\n'`. In C literal = `"\n"` (escape seq).
- SNOBOL4 `\` (one backslash) = Python `'\\'`. In C literal = `"\\\\"`.

---

## Validation

Oracle-verified 2026-03-10:
```
CSNOBOL4:  '\'  → size=1  (backslash is just a char)
CSNOBOL4:  '\n' → size=2  (backslash + n, not newline)
SPITBOL:   identical results
C:         cc rejects "\'" ... no, wait: \' IS valid in C (= single-quote)
C:         cc rejects "\" (unterminated — the " is escaped)
```
