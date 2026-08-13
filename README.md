# ARMv8-A Toolchain (Assembler + Emulator)

**C17 · POSIX · Make · No external dependencies**

A from-scratch two-pass assembler and a fetch–decode–execute emulator for a 
subset of the ARMv8-A instruction set, built over a month by
a team of four as a university systems project.

> **Why no source here:** this project was built for a university
> low level C-based project, so the source can't be
> published on GitHub for academic integrity reasons. This page exists
> instead: an architecture writeup for anyone who
> wants to see how it works and how it was built. 

Please do contact me (aaronjacob815@gmail.com) if you want me to walk you through the real source or send it over, I'd love to share!

---

## What it does

Two command-line binaries, built from ~2,400 lines of C17:

| Binary | Input | Output | Job |
|---|---|---|---|
| `assemble` | `program.s` (ARMv8 assembly) | `program.bin` (little-endian machine code binary file) | Two-pass assembler: resolves labels, encodes instructions |
| `emulate` | `program.bin` | Final register/memory states | Fetch–decode–execute loop over a modeled ARMv8 CPU |

The emulator models 31 general-purpose 64-bit registers, a 64-bit PC, the
`N`/`Z`/`C`/`V` PSTATE flags, and 2 MiB of flat memory. Supported instruction
classes: arithmetic, logical, move (wide immediate), multiply, load/store
(five addressing modes), and branches (unconditional, register, conditional).

---

## Emulator: fetch–decode–execute

```mermaid
flowchart LR
    A["fetch: read 4 bytes @ PC, little-endian"] --> B{"decode: dispatch on op0 (bits 28-25)"}
    B -->|"DP immediate"| C1["execute_dp_imm: arithmetic, wide move"]
    B -->|"DP register"| C2["execute_dp_reg: arithmetic, logical, multiply"]
    B -->|"load / store"| C3["execute_sdt: 5 addressing modes"]
    B -->|"branch"| C4["execute_branch: uncond, reg, conditional"]
    C1 --> D{"advance PC (+4 or branch target)"}
    C2 --> D
    C3 --> D
    C4 --> D
    D -->|"next word is HALT (0x8a000000)"| E["dump registers, PSTATE, non-zero memory"]
    D -->|"else"| A
```

Decode and execute are strictly separated: **decode only extracts bit
fields**, **execute is the only layer with side effects** on CPU/memory
state. This split kept four people working on different instruction
families in parallel without stepping on each other's code.

---

## Assembler: two-pass pipeline

```mermaid
flowchart TB
    subgraph Pass1["Pass 1 - symbol_table.c"]
        S1["scan every source line"] --> S2["record each label definition -> byte address"]
        S2 --> S3[("symbol table")]
    end
    subgraph Pass2["Pass 2 - assemble.c"]
        T1["walk code lines again"] --> T2["parse(): tokenise, build IR"]
        T2 --> T3["encode(): resolve labels via symtab, emit 32-bit word"]
        T3 --> T4["write_word_little_endian()"]
    end
    Pass1 --> Pass2
    S3 -.label lookups.-> T3
    T4 --> OUT[("program.bin")]
```

The intermediate representation (`ir.h`) was a cool datatype: it was a tagged union keyed by
instruction kind (data processing / single data transfer / branching /
directive). Pseudo-instructions (`mov`, `cmp`, `cmn`, `tst`, `neg`, `mul`,
etc.) are rewritten in-place into their canonical encoded form (e.g. `cmp`
becomes `subs` with a discarded destination register), so the encode layer
only ever has to handle "real" instruction shapes.

If encoding fails partway through a file, `atexit(discard_output)` removes
the partially-written binary, so no corrupt `.bin` is ever left on disk.

---

## Module layout

```mermaid
flowchart TB
    subgraph emulator["src/emulator/"]
        E1["emulate.c - main loop"] --> E2["decode.c - bit-field dispatch"]
        E2 --> E3["execute.c - instruction semantics"]
        E3 --> E4["cpu.c - register read/write, zero-register convention"]
        E1 --> E5["load.c - binary loader"]
        E2 --> E6["utils.c - bits(), sign_extend()"]
        E3 --> E6
    end
    subgraph assembler["src/assembler/"]
        A1["assemble.c - pass 1 + pass 2 driver"] --> A2["symbol_table.c"]
        A1 --> A3["parse.c, parse_sdt.c, parse_branch.c"]
        A3 --> A4["encode.c"]
        A4 --> A5["write_binary.c"]
        A1 --> A6["load_assembly.c"]
        A3 --> A7["helpers.c"]
        A4 --> A7
    end
```

---

## Supported instructions

| Category | Mnemonics |
|---|---|
| Arithmetic | `add`, `adds`, `sub`, `subs`, `cmp`, `cmn`, `neg`, `negs` |
| Logical | `and`, `ands`, `bic`, `bics`, `eor`, `eon`, `orr`, `orn`, `tst`, `mvn` |
| Move | `mov`, `movn`, `movk`, `movz` |
| Multiply | `madd`, `msub`, `mul`, `mneg` |
| Load / store | `ldr`, `str` — unsigned offset, pre-/post-indexed, register offset, PC-relative literal |
| Branch | `b`, `br`, `b.eq`, `b.ne`, `b.ge`, `b.lt`, `b.gt`, `b.le`, `b.al` |
| Directive | `.int` (raw 32-bit word — used to emit the halt instruction and literal data) |

Arithmetic/logical instructions accept an optional shifted second operand
(`lsl`, `lsr`, `asr`, `ror`), e.g. `add x0, x1, x2, lsl #3`.

---

## Sample program

```asm
; count x0 up to 5
movz x0, #0
movz x1, #5
loop:
add x0, x0, #1
subs x2, x1, x0
b.ne loop
.int 0x8a000000   ; halt
```

```bash
./build/assemble loop.s loop.bin
./build/emulate  loop.bin
# → dumps x0 = 0x5, all other registers, PC, PSTATE, and any touched memory
```

---

## Engineering details

- **Language / standard:** C17, compiled with `-Wall -Wextra -Wpedantic`
- **Build system:** a hand-written `Makefile` with automatic dependency
  tracking (`-MMD -MP`), separate `assemble` / `emulate` / `clean` targets
- **Platform:** POSIX (Linux/macOS) — uses `getline`, `strtok_r`, `strdup`
- **Dependencies:** none. No external libraries, just the C standard
  library and POSIX APIs
- **Error handling philosophy:** malformed assembly and out-of-range
  memory access fail fast (`assert`/`exit`) with a clear message rather than
  silently producing wrong output
- **Testing:** validated against a spec-conformance test suite (592 tests
  passing on the emulator alone) plus hand-written integration programs run
  end-to-end through `assemble` → `emulate`

## Project stats

- ~2,400 lines of C across 26 source/header files
- 4-person team, ~300 commits, built primarily during a one-week sprint
- Full codebase with commit history exists in the private repository

## Team

Aaron Jacob, Hussain Waseem, Sebastian Kember, Advit Arora

---

*Recruiters/engineers: happy to share the full private repository or walk
through the source directly — reach out via
[aaronjacob815@gmail.com](mailto:aaronjacob815@gmail.com).*
