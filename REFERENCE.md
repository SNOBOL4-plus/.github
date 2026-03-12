# SNOBOL4-plus — Reference

Lookup tables and proven facts. Nothing from memory. No guessing.
Sections: Keywords · String Escapes · SPITBOL Landscape · Sprint 20 Compiland Map

---

# Keywords — Proven Behavior

> Every cell proven by live test runs on 2026-03-10.
> Three systems: CSNOBOL4 (`snobol4 -f`), SPITBOL (`spitbol -b`), SNOBOL4-tiny.
> `✓` works as documented · `✗` absent/fails · `!` surprising — read notes · `?` not yet tested

## Default Values

| Keyword | CSNOBOL4 | SPITBOL | SNOBOL4-tiny | Notes |
|---------|----------|---------|--------------|-------|
| `&STNO` | `2` | `2` | `?` | First readable value |
| `&STCOUNT` | `0` **!** | `2` ✓ | `?` | **BROKEN IN CSNOBOL4 — always 0** |
| `&STLIMIT` | `-1` ✓ | `2147483647` ! | `50000` ! | CSNOBOL4 unlimited; SPITBOL MAX_INT; tiny wrong default |
| `&LASTNO` | `4` ✓ | `4` ✓ | `?` | Previous statement number |
| `&FNCLEVEL` | `0` ✓ | `0` ✓ | `?` | Zero at top level |
| `&FTRACE` | `0` ✓ | `0` ✓ | `?` | Zero = disabled |
| `&ANCHOR` | `0` ✓ | `0` ✓ | `0` stub | Zero = unanchored |
| `&FULLSCAN` | `0` ✓ | `1` ! | `0` stub | **SPITBOL defaults 1; CSNOBOL4 defaults 0** |
| `&TRIM` | `0` ✓ | `1` ! | `1` stub | **SPITBOL defaults 1; CSNOBOL4 defaults 0** |
| `&ERRLIMIT` | `0` ✓ | `0` ✓ | `?` | Zero = abort on first error |
| `&ERRTYPE` | `0` ✓ | `0` ✓ | `?` | Zero = no error |
| `&ABEND` | `0` ✓ | `0` ✓ | `?` | Zero = normal exit |
| `&DUMP` | `0` ✓ | `0` ✓ | `?` | Zero = no dump |
| `&MAXLNGTH` | `4294967295` | `16777216` ! | `524288` ! | **All three differ: CSNOBOL4=4G, SPITBOL=16M, tiny=512K** |
| `&CASE` | `0` ! | `1` ✓ | `?` | CSNOBOL4 `&CASE=0` even with `-f` flag — `-f` ≠ `&CASE=1` |

## Write Behavior

| Keyword | CSNOBOL4 | SPITBOL | Notes |
|---------|----------|---------|-------|
| `&ERRLIMIT`, `&ANCHOR`, `&ABEND`, `&DUMP`, `&STLIMIT` | ✓ R/W | ✓ R/W | |
| `&STCOUNT` | `RO` ✓ | `RO` ✓ | Assignment silently ignored |
| `&STNO`, `&FNCLEVEL` | `RO` ✓ | `RO` ✓ | Assignment silently ignored |

## TRACE Types

| Type | CSNOBOL4 | SPITBOL | Notes |
|------|----------|---------|-------|
| `TRACE(var,'VALUE')` | `!` fires once only | ✓ fires each assignment | CSNOBOL4 one-shot bug |
| `TRACE(lbl,'LABEL')` | ✓ | ✓ | Both fire on branch |
| `TRACE('&STNO','KEYWORD')` | **✗ zero output** | **✗ error 198** | **Broken on both** |
| `TRACE(fn,'CALL')` | **✗ recurses/segfaults** | not tested | Dangerous |

**TRACE output stream**: CSNOBOL4 → stderr. SPITBOL → stdout. The diff monitor must separate these.

**TRACE KEYWORD is non-functional on both oracles for all tested targets.** Exhaustively verified: every `&`-prefixed and bare-named KEYWORD variant produces zero output on CSNOBOL4, error 198 on SPITBOL. Use VALUE trace on a probe variable instead.

## Critical Findings

1. **`&STCOUNT` broken in CSNOBOL4** — always 0. Binary search via `&STLIMIT = &STCOUNT + N` does not work on CSNOBOL4. Use literal values: `&STLIMIT = 500`, `&STLIMIT = 250`, etc. SPITBOL correctly increments it.

2. **SPITBOL TRACE → stdout; CSNOBOL4 TRACE → stderr.** Diff monitor must handle both streams separately.

3. **`TRACE(...,'KEYWORD')` non-functional on both systems.** Per-statement heartbeat must come from VALUE trace on a probe variable, or STCOUNT reads (SPITBOL only), or OUT sync.

4. **Default value differences** (see table above): `&FULLSCAN`, `&TRIM`, `&MAXLNGTH`, `&STLIMIT`.

5. **`&CASE=0` in CSNOBOL4 despite `-f` flag** — `-f` is not the same as `&CASE=1`.

## SNOBOL4-tiny vs Oracle Behavior

| Feature | Needed | tiny current |
|---------|--------|-------------|
| `&STCOUNT` | Increment per statement (SPITBOL model) | Increments internally but not readable |
| `&STLIMIT` | Default -1 or MAX_INT; enforced | Enforced; default 50000 — wrong |
| `&STNO` | Read-only, current stmt number | COMM only, not readable |
| TRACE VALUE | Fire on assignment to watched var | Not implemented |
| TRACE stream | SPITBOL→stdout, CSNOBOL4→stderr | tiny→stderr via COMM |

---

# String Escapes — SNOBOL4 · C · Python

**Purpose**: authoritative conversion table for `emit_c_stmt.py`. Apply the correct rule at every language boundary. Nothing from memory.

## Rule 1: SNOBOL4 — No Escape Sequences

Backslash has **no special meaning** in SNOBOL4 string literals. Confirmed in CSNOBOL4 `syn.c` (`DQLITB`/`SQLITB` states) and SPITBOL `sbl.asm` (`scn18` loop). Both oracles verified 2026-03-10.

| Want in string | SNOBOL4 literal |
|----------------|----------------|
| Any printable char | Write literally |
| Backslash `\` | `'\'` — just the char, size=1 |
| Newline | Cannot be in literal. Use `nl = CHAR(10)` |
| Tab | Cannot be in literal. Use `tb = CHAR(9)` |
| Single-quote `'` | Use `"'"` or concatenate |
| Double-quote `"` | Use `'"'` or concatenate |

**`'\n'` in SNOBOL4 = TWO characters: backslash + n. NOT a newline.**

## Rule 2: C — Backslash IS the Escape Character

| Want | C literal | Trap |
|------|-----------|------|
| `\` | `"\\"` | Must double |
| `"` | `"\""` | Must escape in `"..."` |
| newline | `"\n"` | |
| tab | `"\t"` | |
| null | `"\0"` | |
| `\X` not in table | **STRAY BACKSLASH — compile error** | |

## Rule 3: Python — Backslash IS the Escape Character

Same as C for the common cases. `r"..."` raw strings pass backslash literally.

## Rule 4: The Conversion Function

```python
def sno_val_to_c_literal(s: str) -> str:
    """
    Convert a Python str (raw SNOBOL4 value) to a C string literal body.
    Apply EXACTLY ONCE at the Python→C boundary. Never call twice.
    """
    result = []
    for ch in s:
        if ch == '\\':   result.append('\\\\')   # SNOBOL \ → C \\
        elif ch == '"':  result.append('\\"')     # SNOBOL " → C \"
        elif ch == '\n': result.append('\\n')
        elif ch == '\r': result.append('\\r')
        elif ch == '\t': result.append('\\t')
        elif ch == '\0': result.append('\\0')
        else:            result.append(ch)
    return '"' + ''.join(result) + '"'
```

## Rule 5: The Layering Trap

**Never apply escape conversion twice.** `emit_as_str()` produces valid C. Do not pass it through another replace/escape function. The moment a Python `str` becomes a C source token, it is done.

## Quick Reference Card

```
SNOBOL4       Python str    C literal
────────────────────────────────────
\             \\            \\
"             "             \"
\n (2 chars)  \\n (2 chars) \\n  (still 2 chars in C)
newline       \n (1 char)   \n   (C newline escape)
tab           \t (1 char)   \t   (C tab escape)
```

---

# Linux SNOBOL/SPITBOL Binary Landscape

## Available Implementations

| System | Version | Linux x86-64 | Source | Binary | Get it |
|--------|---------|:------------:|--------|--------|--------|
| **CSNOBOL4** | 2.3.3 (Jan 2024) | ✅ | C (portable) | build from source or distro pkg | http://www.snobol4.org/csnobol4/curr/ |
| **SPITBOL x64** | 4.0f | ✅ | MINIMAL + C + nasm | `./bin/sbl` pre-built in repo | https://github.com/spitbol/x64 |
| **SPITBOL x32** | — | ❌ | MINIMAL + C | 32-bit ELF only — qemu-i386 segfaults | https://github.com/spitbol/x32 |
| **SNOBOL5** | beta 5.0 (Aug 2024) | ✅ | x86-64 ASM + SIL macros (public domain) | statically linked binary, no GitHub | http://snobol5.org/ |
| **SPITBOL 360** | historic | ❌ | IBM/360 ASM | IBM/360 only | https://github.com/spitbol/360 |
| **SPITBOL NT** | 1.30.22 (Dec 2003) | ❌ | C + DOS extender | 32rtm, Wine can't run it | https://github.com/spitbol/windows-nt |

**On this machine**: CSNOBOL4 2.3.3 at `/usr/local/bin/snobol4`, SPITBOL x64 4.0f at `/usr/local/bin/spitbol`.

**SNOBOL5 note**: Oregon SNOBOL5 (Viktors Berstis) is an updated Minnesota SNOBOL4 — native x86-64 Linux binary, beta quality, binary-only (no GitHub source repo). Not yet evaluated as oracle.

---

## Build Requirements

| System | Build deps | Build command | Time |
|--------|-----------|---------------|------|
| CSNOBOL4 | `gcc`, `make` | `./configure && make && make install` | ~1 min |
| SPITBOL x64 | `gcc`, `nasm` | `make && ./sanity-check` | ~2 min |
| SNOBOL5 | none (binary-only) | `wget https://snobol5.org/snobol5 && chmod 555 snobol5` | instant |
| SNOBOL5 (source) | `nasm`, Windows for full build | gens5.cmd (Windows-centric) — Linux final steps only | not straightforward |

---

## Command-Line Switches Comparison

### CSNOBOL4 (`snobol4 [options] file`)

| Flag | Effect |
|------|--------|
| `-b` | Disable startup banner and termination output |
| `-B` | Force banner and termination output |
| `-c` | Disable case folding (identifiers stay mixed case) — same as `&CASE=1` |
| `-d N` | Dynamic storage region: N descriptors (suffix `k`=×1024, `m`=×1048576) |
| `-e` | Toggle running programs with compilation errors |
| `-E` | (see `-e`) |
| `-f` | Toggle case folding (same as `-c`) |
| `-h` | Show help/usage + default sizes, exit |
| `-I dir` | Add dir to include search path |
| `-l [file]` | Enable listing output to file (default: stderr) |
| `-M` | Treat remaining args as filenames (read until END) |
| `-n` | Toggle execution after compilation (compile only) |
| `-P N` | Pattern match stack: N descriptors (suffix `k`/`m`) |
| `-r` | Toggle reading INPUT from post-END data in source file |
| `-s` | Toggle SPITBOL extensions |
| `-S N` | Interpreter stack: N descriptors |
| `-t` | Toggle termination statistics |
| `-u string` | Set HOST(0) return value |
| `-U` | Disable stdio buffering |
| `-v` | Show version and exit |
| `-x` | No standard include search path (only `-I` dirs) |
| `--` | End of option processing |

**Invocation we use**: `snobol4 -f -P256k file.sno`
(`-f` = disable case folding so identifiers are case-sensitive; `-P256k` = 256K pattern stack)

---

### SPITBOL x64 (`spitbol [options] file`)

| Flag | Effect |
|------|--------|
| `-f` | Don't fold lower-case names to UPPER CASE (SNOBOL4 compatibility) |
| `-F` | Force fold lower-case to UPPER CASE |
| `-e` | Don't send error messages to terminal |
| `-l` | Generate source listing |
| `-c` | Generate compilation statistics |
| `-x` | Generate execution statistics |
| `-a` | Like `-lcx` (listing + both stats) |
| `-p` | Long listing format (generates form feeds) |
| `-z` | Use standard listing format |
| `-h` | Write SPITBOL header to stdout |
| `-n` | Suppress execution (compile only) |
| `-b` | Run in batch mode — `&TRIM` and `&ANCHOR` default on |
| `-mN` | Max size (words) of created object (default 8192) |
| `-sN` | Max stack space in words (default 2048) |
| `-iN` | Increment size for dynamic area growth (default 4096) |
| `-dN` | Max allocated dynamic area in words (default 256K) |
| `-u string` | String retrievable via `HOST(0)` |
| `-o file` | Write listing/stats/dump to file; OUTPUT to stdout |

**Invocation we use**: `spitbol -b file.sno`

---

### SNOBOL5 (`snobol5 [options] [files] [:user data]`)

| Flag | Effect |
|------|--------|
| `--work=N[b\|k\|m\|g]` | Workspace size |
| `-ex` | Execute even if there are compile errors |
| `-s` | Produce statistics |
| `-d` | Set `&DUMP` to 1 (dump variables at termination) |
| `-v` | Produce source listing and statistics |
| `--help` | Show help and exit |
| `-nn filename [attribs]` | Associate file with I/O unit `nn` (default unit is 5) |

**File attributes**: `-a` (ASCII text), `-b` (binary), `-tabx`/`-ntabx` (tab handling), `-r` (replace on write), `-ap` (append), `-vl` (variable length read), `-fl` (fixed length), `-i`/`-ni` (include/snopath processing), `-std` (stdin/stdout/stderr by name), `-cr`/`-ncr` (carriage control).

**Invocation example**: `snobol5 -v myprogram.sno -1 datafileA.txt -b`

**No `-f` case-fold flag**: SNOBOL5 does not have `&CASE`. Case folding behavior differs from both CSNOBOL4 and SPITBOL — `&CASE` is an unknown keyword (Error 7). Lower-case identifiers appear to be handled differently.

**No `-INCLUDE` in the traditional sense**: uses `snopath` environment variable for include file resolution.

---

## SNOBOL5 Assessment (2026-03-11)

**Binary**: statically linked ELF64, stripped, 970K. `wget https://snobol5.org/snobol5`. Works immediately.

**Source**: `s5source.zip` — public domain. 26,454 lines total. Core is `MAIN.A/B/C/D` (6,613 lines of SIL macro code, dated 1969–1982, the original Macro SNOBOL4 SIL code), `snobol5.asm` (1,333 lines, x86-64 NASM, Berstis's SIL interpreter), plus ~40 `.inc` files for runtime. Build is Windows-centric (`gens5.cmd`); Linux final step only. **Not straightforwardly buildable on Linux alone.**

**Architecture**: SIL macros (the same 1960s/70s Bell Labs SIL used by the original Macro SNOBOL4) implemented by a custom x86-64 assembler runtime. This is the same lineage as CSNOBOL4 (which implements SIL in C) and SPITBOL (which implements SIL in MINIMAL → ASM). SNOBOL5 goes directly SIL → x86-64 ASM.

**SNOBOL4 compatibility**: High. Standard patterns, SPAN/BREAK/ARB/ARBNO/FENCE/BAL, DEFINE/RETURN/FRETURN, TABLE/ARRAY, DATA/FIELD, EVAL(), CONVERT() all work correctly. N-queens (92 solutions) produces correct output.

**Key differences from CSNOBOL4/SPITBOL**:
- **64-bit integers and strings** — `&STLIMIT` = 9223372036854775807, `&MAXLNGTH` = 64G, integers are true 64-bit
- **`&CASE` does not exist** — case folding handled differently; `&CASE` → Error 7 "Unknown keyword"
- **`&STCOUNT` is read-only** — write attempt → Error 8 (unlike CSNOBOL4 where it silently returns 0, SNOBOL5 increments it correctly and protects it)
- **OPSYN limited to single characters** — `++` → "Illegal character in element"; single-char operators work
- **`CODE()` does not work** — `=?` assignment-to-code syntax produces syntax error
- **`LOAD()`/`UNLOAD()` not yet implemented** — documented as planned
- **Rich new function library**: trig (SIN/COS/TAN/ATAN), bit/hex/byte conversions, GETENV, SYSTEM, SORT, RAND, DATE, TIME (nanoseconds), SLEEP, REV, SPANNNOT, CENTER, SEEK, IO_FINDUNIT, ISLABEL

**Performance** (FIB(20) recursive, 2026-03-11): ~13 ms. Competitive with CSNOBOL4.

**Oracle fitness**: Untested against beauty.sno (corpus not available this session). Primary risk: `CODE()` broken, `&CASE` missing, OPSYN operator restrictions. For programs that don't use those features, likely viable.

**Verdict**: A serious, working SNOBOL4 implementation with genuine 64-bit improvements. Not a drop-in oracle replacement — `&CASE` and `CODE()` gaps are significant. Worth keeping in the inventory. Build situation is a limitation.

---

## Key Behavioral Differences (All Three)

| Behavior | CSNOBOL4 | SPITBOL x64 | SNOBOL5 |
|----------|----------|-------------|---------|
| `&ANCHOR` default | 0 | **1** (v4.0+) | 0 |
| `&TRIM` default | 0 | **1** (v4.0+) | 0 |
| `&CASE` | keyword exists | keyword exists | **unknown keyword (Error 7)** |
| `&FULLSCAN` default | 0 | 1 | 0 |
| `&STCOUNT` | **broken — always 0** | increments correctly | increments correctly; **write → Error 8 (read-only)** |
| `&STLIMIT` default | -1 (unlimited) | MAX_INT (32-bit) | **9223372036854775807 (64-bit MAX)** |
| `&MAXLNGTH` | 4G | 16M | **64G (64-bit)** |
| Integer size | 32-bit | 32-bit | **64-bit** |
| TRACE output stream | stderr | **stdout** | ? (not tested) |
| `-INCLUDE` | ✅ | ✅ (different END handling) | ✅ (via `snopath` env var) |
| BLOCKS extension | ✅ | ❌ | ❌ |
| LOAD() plugin | ✅ | ❌ | ❌ (planned) |
| Shebang `#!/...` | ✅ | ✅ | ? |
| OPSYN | ✅ (multi-char ok) | ✅ | ✅ (single-char only — `++` → Error "Illegal character") |
| TABLE / ARRAY / DATA | ✅ | ✅ | ✅ |
| EVAL() | ✅ | ✅ | ✅ |
| CODE() | ✅ | ✅ | ❌ (syntax error — `=?` not recognized) |
| CONVERT() | ✅ | ✅ | ✅ |
| 64-bit integers | ❌ | ❌ | ✅ (`9000000000 * 2 = 18000000000`) |
| New functions | — | — | HEX/BIT/BYTES conversions, trig, GETENV, SORT, SYSTEM, RAND, DATE, TIME, SLEEP, REV, SPANNNOT, INT, REAL, STR, CENTER, SEEK, IO_FINDUNIT, ISLABEL |
| Implementation | C interpreter | MINIMAL+ASM compiler | SIL macros → x86-64 ASM |
| Oracle fitness (beauty.sno) | ✅ **primary** | ❌ error 021 at END | **untested** (corpus not available this session) |

---

# SPITBOL Landscape

## The Lineage

**Robert Dewar** + Ken Belcher (IIT, 1969) — SPITBOL/360, IBM/360 assembly.
Dewar + Anthony McCann — rewrote in **MINIMAL** (portable assembly) → Macro SPITBOL. ~28,000 lines. Executable under 150KB.
**Mark Emmer** (Catspaw, Inc.) — maintained 1987–2003. Windows NT v1.30.22 December 2003. Files to Dave Shields 2009.
**Dave Shields** (IBM/NYU/HARDBOL) — first open-source Linux release June 2012 under GPLv2.
**Cheyenne Wills** — current maintainer of x86_64 version. Working on ARM64. Contact for Debian packaging.

## GitHub: `github.com/spitbol`

| Repo | What | URL |
|------|------|-----|
| `x64` | **Current — x86_64 Linux/Unix** | https://github.com/spitbol/x64 |
| `x32` | i386 32-bit | https://github.com/spitbol/x32 |
| `360` | Original IBM 360 | https://github.com/spitbol/360 |
| `windows-nt` | Windows NT v1.30.22 (Emmer, 2003) | https://github.com/spitbol/windows-nt |
| `spitbol-docs` | Green Book, SPITBOL Manual | https://github.com/spitbol/spitbol-docs |

## The x64 Version — What We Use

Build: `gcc` + `nasm`. Binary at `./bin/sbl`. Install: `make install` → `/usr/local/bin/spitbol`.

**v4.0 defaults** (differ from CSNOBOL4):
- `&ANCHOR` defaults to 1 (was null)
- `&TRIM` defaults to 1 (was null)
- `&CASE` defaults to 0 (case-sensitive)

**SPITBOL is disqualified as oracle for `beauty.sno` (Level 3)** — error 021 at `END`, indirect function call semantic difference. CSNOBOL4 is the sole authoritative oracle for Sprint 20.

## What Runs On This Machine

| System | Binary | Role |
|--------|--------|------|
| CSNOBOL4 2.3.3 | `/usr/local/bin/snobol4` | **Primary oracle** |
| SPITBOL x64 4.0f | `/usr/local/bin/spitbol` | Secondary reference |

Not running: SPITBOL x32 (32-bit ELF, qemu-i386 segfaults), SPITBOL-88 (DOSBox needs display), SPITBOL NT (Wine can't run 32rtm).

## Install Commands

```bash
# SPITBOL x64 from source:
git clone https://github.com/spitbol/x64 spitbol-x64
cd spitbol-x64
apt-get install nasm
make
./sanity-check
cp sbl /usr/local/bin/spitbol
```

---

# Sprint 20 — Compiland Reachability

*Generated 2026-03-10 from `beauty.sno → beautiful.c` analysis.*

## Pattern Constructors (snobol4_pattern.c) — 19/19 ✅

All implemented: `sno_pat_alt`, `sno_pat_any_cs`, `sno_pat_arbno`, `sno_pat_assign_cond` (53 uses), `sno_pat_assign_imm`, `sno_pat_break_`, `sno_pat_cat` (116 uses), `sno_pat_epsilon`, `sno_pat_len`, `sno_pat_lit`, `sno_pat_pos`, `sno_pat_ref`, `sno_pat_rpos`, `sno_pat_rtab`, `sno_pat_span`, `sno_pat_user_call`, `sno_match_pattern` (80 uses), `sno_match_and_replace`, `sno_var_as_pattern`.

## Runtime API (snobol4.c) — 41/41 ✅

All implemented. High-use: `sno_to_str` (1,772 uses), `sno_apply` (623), `sno_concat` (758), `sno_var_get` (1,387), `sno_var_set` (414).

## Inc-Layer C Functions (snobol4_inc.c)

| Function | Source | Status |
|----------|--------|--------|
| `lwr`, `upr` | case.inc | ✅ |
| `Gen`, `GenTab`, `GenSetCont`, `IncLevel`, `DecLevel`, `SetLevel`, `GetLevel` | Gen.inc | ✅ |
| `match` | match.inc | ✅ |
| `Qize` | Qize.inc | ✅ |
| `T8Pos`, `TDump` | trace/TDump.inc | ✅ |
| `TZ`, `XDump` | omega/XDump.inc | ✅ |
| `assign` | assign.inc | ✅ |
| `icase` | case.inc | ⚠️ MISSING |
| `IsSnobol4` | is.inc | ⚠️ MISSING |
| `Push`, `Pop` | stack.inc | ⚠️ MISSING |
| `TopCounter` | counter.inc | ⚠️ MISSING |
| `SqlSQize` | Qize.inc | ⚠️ MISSING |
| `TLump`, `TValue` | TDump.inc | ⚠️ MISSING (stub ok) |
| `Visit`, `bVisit`, `Equal`, `Equiv`, `Find`, `Insert` | tree.inc | ⚠️ MISSING |

**10 missing → add to `snobol4_inc.c` before beautiful.c compiles correctly.**

## SNOBOL4 DEFINE'd Functions

Called via `sno_apply("pp", ...)` etc. These are SNOBOL4 functions compiled into beautiful.c as C goto-label blocks. **Sprint 21 deliverable**: `emit_c_stmt.py` emits each DEFINE'd function as a proper C function with its own RETURN/FRETURN labels, not just a goto block.

Key functions: `pp` (pretty-printer, recursive), `ss` (stringize), `pp_*` / `ss_*` (dispatch variants), tree field accessors (`c`, `t`, `v`, `n`), `nPush`/`nPop`/`nInc` (counter stack).

## DATA() Types Used

| Spec | Status |
|------|--------|
| `tree(t,v,n,c)` | ✅ sno_data_define'd at startup |
| `link(next,value)` | ✅ DEFINE'd in beautiful.c |
| `link_counter(next,value)` | ✅ DEFINE'd |
| `link_tag(next,value)` | ✅ DEFINE'd |

## Summary

| Category | Needed | Done | Missing |
|----------|-------:|-----:|--------:|
| Pattern constructors | 19 | 19 | 0 |
| Runtime API | 41 | 41 | 0 |
| Inc C functions | 30 | 20 | **10** |
| SNOBOL4 DEFINE dispatch | ~40 | 0 | ~40 (Sprint 21) |

**To compile beautiful.c: add 10 missing inc registrations.**
**To run correctly: DEFINE dispatch — Sprint 21.**
