# SESSION.md — Live Handoff

> This file is fully self-contained. A new Claude reads this and nothing else to start working.
> Updated at every HANDOFF. History lives in SESSIONS_ARCHIVE.md.

---

## Active Session

| Field | Value |
|-------|-------|
| **Repo** | SNOBOL4-tiny |
| **Sprint** | `beauty-runtime` (sprint 3/4 toward M-BEAUTY-FULL) |
| **Milestone** | M-BEAUTY-FULL |
| **HEAD** | `4e2712d` — WIP: E_DEREF impl + decl dedup — sno_output call still broken in emit_byrd.c COND handler |

---

## Sprint 3 Status — one linker error blocking binary

Three things were implemented this session and are correct:

1. **`sno_match_pattern_at()`** — new runtime function in `snobol4_pattern.c` + declared in `snobol4.h`.
   Cursor-aware anchored match: takes `(SnoVal pat, const char *subject, int subj_len, int cursor)`,
   returns new cursor position on success or -1 on failure.

2. **`E_DEREF` in `emit_byrd.c`** — `case E_DEREF:` now emits real Byrd box code:
   calls `sno_var_get(varname)` then `sno_match_pattern_at(...)`, saves/restores cursor
   across alpha/beta ports. No longer epsilon.

3. **Cross-pattern static-decl dedup** — `fn_seen[]` + `byrd_fn_scope_reset()` in `emit_byrd.c`.
   `byrd_fn_scope_reset()` called at start of `emit_fn()` and `emit_main()` in `emit.c`.
   Eliminates "redeclaration of var_part with no linkage" gcc errors.

---

## One Next Action — Fix `sno_output` call in emit_byrd.c

### The linker error

```
undefined reference to `sno_output'
```

### Where it is

`src/sno2c/emit_byrd.c` around line 826 — the `emit_imm` function's OUTPUT special case:

```c
if (strcasecmp(varname, "OUTPUT") == 0) {
    B("    { str_t _s = { %s + %s, (int64_t)(%s - %s) };\n", ...);
    B("      sno_output(_s); }\n");   // ← WRONG: sno_output(str_t) does not exist
}
```

### The fix

`sno_output` does not exist. The real API (from `snobol4.h` line ~300) is:

```c
void sno_output_val(SnoVal v);      // OUTPUT = SnoVal
void sno_output_str(const char *s); // OUTPUT = C string
```

Replace the broken emission with `sno_output_str` using a null-terminated copy,
OR construct a SnoVal and call `sno_output_val`. Simplest fix:

```c
if (strcasecmp(varname, "OUTPUT") == 0) {
    B("    { int64_t _len = %s - %s;\n", cursor, start_var);
    B("      char *_os = malloc(_len + 1); memcpy(_os, %s + %s, _len); _os[_len] = 0;\n", subj, start_var);
    B("      sno_output_str(_os); free(_os); }\n");
}
```

Or use `sno_output_val` with `SNO_STR_VAL` — but that requires a null-terminated string too.
The malloc approach is clean and correct. Add `#include <stdlib.h>` is already present in
generated files via snobol4.h.

### After fixing

```bash
cd /home/claude/SNOBOL4-tiny
make -C src/sno2c
CORPUS=/home/claude/SNOBOL4-corpus
INC=$CORPUS/programs/inc
BEAUTY=$CORPUS/programs/beauty/beauty.sno
./src/sno2c/sno2c -I $INC $BEAUTY > /tmp/beauty_full.c
R=src/runtime
gcc -O0 -g -I $R/snobol4 -I $R \
    /tmp/beauty_full.c $R/snobol4/snobol4.c \
    $R/snobol4/snobol4_inc.c $R/snobol4/snobol4_pattern.c \
    $R/engine_stub.c -lgc -lm -o /tmp/beauty_full_bin
/tmp/beauty_full_bin < $BEAUTY > /tmp/beauty_out.sno
echo "exit=$?  lines=$(wc -l < /tmp/beauty_out.sno)"
# Target: 790 lines, exit 0
```

### Oracle comparison

```bash
/home/claude/snobol4-2.3.3/snobol4 -f -P256k -I $INC $BEAUTY < $BEAUTY > /tmp/beauty_oracle.sno
diff /tmp/beauty_oracle.sno /tmp/beauty_out.sno
# Sprint 4 fires when diff is empty
```

---

## CRITICAL: What Next Claude Must NOT Do

- Do NOT fix bugs in sno_pat_* / engine.c — retired from compiled path.
- Do NOT rewrite emit_byrd.c wholesale — only fix sno_output call.
- Do NOT reset byrd_uid_ctr — continuity fix is intentional.
- Do NOT remove fn_seen[] / byrd_fn_scope_reset() — fixes real gcc errors.
- Do NOT write the TOKEN into any file.

---

## Container State (clone fresh each session)

    apt-get install -y m4 libgc-dev
    git config --global user.name "LCherryholmes"
    git config --global user.email "lcherryh@yahoo.com"
    TOKEN=TOKEN_SEE_LON
    git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-tiny.git
    git clone https://$TOKEN@github.com/SNOBOL4-plus/SNOBOL4-corpus.git
    git clone https://$TOKEN@github.com/SNOBOL4-plus/.github.git dotgithub

    # Build CSNOBOL4 — tarball in uploads as snobol4-2_3_3_tar.gz
    cp /mnt/user-data/uploads/snobol4-2_3_3_tar.gz .
    tar xzf snobol4-2_3_3_tar.gz && cd snobol4-2.3.3
    ./configure --prefix=/home/claude/snobol4-install && make -j$(nproc)
    cd ..

    cd SNOBOL4-tiny && make -C src/sno2c

---

## Pivot Log

| Date | What changed | Why |
|------|-------------|-----|
| 2026-03-15 | sno_output linker error blocking binary | emit_byrd.c COND OUTPUT case uses nonexistent fn |
| 2026-03-15 | fn_seen[] + byrd_fn_scope_reset() — cross-pattern decl dedup | static redecl errors in Gen/Qize |
| 2026-03-15 | E_DEREF implemented with sno_match_pattern_at() | beauty uses *varname ~100x |
| 2026-03-14 | E_DEREF identified as root cause of Internal Error | beauty uses *var ~100x |
| 2026-03-14 | sno_output sig fixed (str_t→ptr+len), decl dedup across fn scope | gcc clean |
| 2026-03-14 | binary links and exits 0 (91d097c) | 3 gcc bugs fixed |
| 2026-03-15 | M-COMPILED-BYRD fired (560c56a) | engine_stub.c + ALL OK |
| 2026-03-15 | uid continuity fix (735c456) | duplicate labels across multiple patterns |
| 2026-03-15 | ARB scan wrap (735c456) | substring scan semantics |
| 2026-03-15 | emit.c wired — byrd_emit_pattern() called (1c2062a) | compiled path active |
| 2026-03-13 | emit_byrd.c written committed (cb3f97e) | C port of Python pipeline complete |
