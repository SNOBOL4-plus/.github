# SPITBOL Landscape — Distributions, Owners, Repos

*Recorded 2026-03-10 — researched from GitHub and public sources*

---

## The Lineage

**Robert B. K. Dewar** and **Ken Belcher** (Illinois Institute of Technology)
created SPITBOL/360 in 1969 — the first true compiler for SNOBOL4.
Written in IBM/360 assembly ("aggressive assembly" — every trick in the book).

Dewar and **Anthony P. McCann** then rewrote it in **MINIMAL** — a portable
assembly language — producing **Macro SPITBOL**. This is the version all
modern distributions descend from. ~28,000 lines: 2,000 comments defining
MINIMAL, 5,000 data declarations, 21,000 lines of code. Every line commented.
Executable under 150KB.

**Mark Emmer** (Catspaw, Inc.) took over in 1987. Maintained it until 1994,
porting to Mac, Windows, Unix. Sold it as a mail-order product.
Released Windows NT version 1.30.22 in December 2003. Files transferred
to Dave Shields in 2009.

**Dave Shields** (IBM / NYU / HARDBOL Software, Scarsdale NY) became
maintainer in 2009. First open-source Linux release June 2012 under GPLv2.
Staged at `hardbol/spitbol`, then moved to `spitbol/` GitHub org.

**Cheyenne Wills** (cheyenne.wills@gmail.com) inherited from Shields
and is the current maintainer of the x86_64 Linux version. Contact
for the Debian packaging request (2024). In contact with Mark Emmer
on future direction. Working on ARM64 port. Has source for VM/CMS 370
version not yet published.

---

## GitHub Organization: `github.com/spitbol`

Owner/maintainer: **Cheyenne Wills**

| Repo | What | Language | Stars | URL |
|------|------|----------|-------|-----|
| `x64` | **Current main — x86_64 Linux/Unix** | MINIMAL + C + ASM | 274 | https://github.com/spitbol/x64 |
| `x32` | i386 32-bit Linux/Unix | MINIMAL + C | 22 | https://github.com/spitbol/x32 |
| `360` | SPITBOL 360 — original IBM 360 (EBCDIC) | IBM/360 ASM | 54 | https://github.com/spitbol/360 |
| `windows-nt` | Windows NT v1.30.22 (Dec 2003, Mark Emmer) | C | 8 | https://github.com/spitbol/windows-nt |
| `88-source` | Micro SPITBOL-386 source (reduced memory) | C | — | https://github.com/spitbol/88-source |
| `88-binary` | Micro SPITBOL-386 binary | — | — | https://github.com/spitbol/88-binary |
| `spitbol-docs` | Documentation — Green Book, SPITBOL Manual | — | 13 | https://github.com/spitbol/spitbol-docs |
| `pal` | Macro SPITBOL in Go (experimental) | Go | — | https://github.com/spitbol/pal |

---

## The x64 Version — What We Care About

**`github.com/spitbol/x64`** — this is the one.

- x86_64 only, Unix/Linux/FreeBSD
- Build requires: `gcc` + `nasm` (Netwide Assembler)
- Binary at `./bin/sbl` (base version used to bootstrap)
- Bootstrap test: builds itself 3 times, diffs outputs
- Install: `make install` → `/usr/local/bin/spitbol`
- **v4.0 behavior changes** (important for our oracle):
  - `&ANCHOR` defaults to **1** (was null) — anchored match by default
  - `&TRIM` defaults to **1** (was null) — trim trailing blanks by default
  - `&CASE` defaults to **0** (case-sensitive, no folding) — use `-F` to fold
  - These differ from CSNOBOL4 defaults — **documented in KEYWORD_GRID.md**

**Binary available**: `./bin/sbl` in the repo — ready to run without building.

---

## The Dewar Version — Historical Context

Dewar's involvement:
- SPITBOL/360 (1969) — Dewar + Belcher, IIT
- Macro SPITBOL (1970s) — Dewar + McCann, in MINIMAL
- Copyright 1987–2012: Robert B. K. Dewar and Catspaw, Inc. (Mark Emmer)
- GPL release: SPITBOL/360 → November 2001; Macro SPITBOL → April 17, 2009
- Dewar is the intellectual author. Shields/Wills are the current stewards.

There is no separate "Dewar distribution" on GitHub. Dewar's work lives
inside the `spitbol/x64` codebase as the MINIMAL source (`s.min` / `v37.min`).

---

## Dave Shields — HARDBOL

**`github.com/hardbol/spitbol`** — staging area, now redirects to `spitbol/spitbol`
(which is itself an older repo; the canonical current one is `spitbol/x64`).

Dave Shields is at GitHub as `daveshields`. His repos include:
- `hardbol/spitbol` — old staging area, now says "repo lives at spitbol/spitbol"
- `daveshields` — personal repos including Emmer's 2009 file transfer

---

## Other Implementations (Not Macro SPITBOL)

| Name | What | Where |
|------|------|-------|
| CSNOBOL4 | Our oracle — interpreter, not compiler | http://www.snobol4.org / `/usr/local/bin/snobol4` on this machine |
| SNOBOL4+ | Catspaw commercial dialect (Emmer) | Windows binary, historical |
| spipatjs | SPITBOL pattern matching in JavaScript | https://github.com/philbudne/spipatjs |
| SETL4 | SPITBOL + set theory | https://github.com/setl4/setl4 |

---

## For This Project — What Matters

| System | Role | Status |
|--------|------|--------|
| CSNOBOL4 (`snobol4`) | **Primary oracle** — Level 1/2/3 reference | Installed at `/usr/local/bin/snobol4` |
| SPITBOL x64 (`spitbol`) | Secondary reference — confirmed NOT usable as oracle for `beauty_run.sno` (fails at line 630, error 021) | Available at `github.com/spitbol/x64` |
| SPITBOL x64 binary | Can install from repo `./bin/sbl` without building | Clone + copy |

**SPITBOL as oracle is disqualified for Level 3** (beauty_run.sno) because:
- Does not handle `-INCLUDE` directives the same way
- Error 021 at `END` statement — indirect function call semantic difference
- See `MONITOR.md` § Current Monitor State

CSNOBOL4 is the sole authoritative oracle. Documented.

---

## How to Install SPITBOL x64 (for reference)

```bash
git clone https://github.com/spitbol/x64
cd x64
# Option A: use pre-built binary
cp bin/sbl /usr/local/bin/spitbol

# Option B: build from source (requires nasm)
apt-get install nasm
make
make install
```

---

*Sources: github.com/spitbol (all repos), groups.io/g/spitbol,
Debian RFP bug#1061167, Vice/Motherboard article (2024),
daveshields.wordpress.com, snobol4.org*

---

## Live Inventory — What Actually Runs On This Machine

*Tested 2026-03-10 on Ubuntu 24.04 x86_64*

### Running

| # | System | Version | Binary | Source | Notes |
|---|--------|---------|--------|--------|-------|
| 1 | **CSNOBOL4** | 2.3.3 (May 2025) | `/usr/local/bin/snobol4` | pre-installed | **Primary oracle** |
| 2 | **SPITBOL x64** | 4.0f | `/usr/local/bin/spitbol` | pre-installed | Secondary reference |
| 3 | **SPITBOL x64** (from source) | 4.0f | `/home/claude/spitbol-x64/sbl` | `github.com/spitbol/x64` — built with `make`, passed sanity check |

All three produce identical output on pattern tests. ✓

### Not Running (yet)

| # | System | Why | Path to fix |
|---|--------|-----|-------------|
| 4 | SPITBOL x32 (i386) | `Exec format error` — pure 32-bit ELF, no 32-bit kernel | `qemu-i386` segfaults; needs investigation |
| 5 | SPITBOL 88 (DOS, LZEXE) | DOSBox-X needs display/X11 | Run in DOSBox-X with display |
| 6 | SPITBOL Windows NT 1.30.22 | 32rtm DOS extender, Wine can't run it | Needs proper DOS extender support |

### What This Means For The Monitor

We have **three oracles** that all agree. This is valuable:

- Run all three on a Level 1 test. If they all produce the same trace → high confidence.
- If CSNOBOL4 and SPITBOL disagree on something → dialect difference, document it.
- The binary (SNOBOL4-tiny) must match all three on the agreed behavior.

The installed SPITBOL (`/usr/local/bin/spitbol`) is the same version as the
one built from source (`spitbol-x64/sbl`) — both v4.0f. The installed one
is the canonical reference; the source build is for experimentation.

### Install Commands (for reproducibility)

```bash
# CSNOBOL4 — already installed
# Source: http://www.snobol4.org/csnobol4/

# SPITBOL x64 — already installed
# Source: github.com/spitbol/x64

# Build SPITBOL x64 from source:
git clone https://github.com/spitbol/x64 spitbol-x64
cd spitbol-x64
apt-get install nasm
make
./sanity-check   # must pass before trusting the build
# binary at ./sbl
```
