# PART 3 — Executable Formats

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 3 of 10**

> [!NOTE]
> **Source:** written to follow the [roadmap](../INDEX.md). The video labelled "PART 3"
> ([NefasNimlIk](https://www.youtube.com/watch?v=NefasNimlIk)) actually covers `cmp`/`test`/jumps
> and patching — that material lives in [PART 2](../PART2/note.md).
> Technical terms stay in English, but every term is explained in plain language first.
> Glossary: [section 12](#12-glossary).

**Prerequisites:** [PART 2](../PART2/note.md) done — you can read disassembly and have patched a
byte with `dd`.

**After this part you can:** read an ELF header, tell sections and segments apart, know which
section holds code/strings/function names, **compute a file offset from a memory address**
instead of guessing, explain what `strip` removes, and recognize the basic layout of PE (Windows).

## Table of contents

- [PART 3 — Executable Formats](#part-3--executable-formats)
  - [Table of contents](#table-of-contents)
  - [1. Why file formats?](#1-why-file-formats)
  - [2. The ELF header: the first 64 bytes](#2-the-elf-header-the-first-64-bytes)
  - [3. Sections and segments](#3-sections-and-segments)
  - [4. Sections you must recognize](#4-sections-you-must-recognize)
  - [5. Memory address vs file offset](#5-memory-address-vs-file-offset)
    - [PIE vs non-PIE](#pie-vs-non-pie)
  - [6. Symbols and stripping](#6-symbols-and-stripping)
  - [7. Dynamic linking: PLT and GOT](#7-dynamic-linking-plt-and-got)
  - [8. PE (Windows) in brief](#8-pe-windows-in-brief)
  - [9. Hands-on: reading ELF](#9-hands-on-reading-elf)
    - [9.1 Build the targets](#91-build-the-targets)
    - [9.2 Lab 1 — Translate the header by hand](#92-lab-1--translate-the-header-by-hand)
    - [9.3 Lab 2 — Find the secret's home](#93-lab-2--find-the-secrets-home)
    - [9.4 Lab 3 — Prove the address-to-offset formula](#94-lab-3--prove-the-address-to-offset-formula)
    - [9.5 Lab 4 — The names that vanish (and the ones that survive)](#95-lab-4--the-names-that-vanish-and-the-ones-that-survive)
  - [10. Recap](#10-recap)
    - [Self-check](#self-check)
  - [11. Cliffhanger: from file to C code](#11-cliffhanger-from-file-to-c-code)
  - [12. Glossary](#12-glossary)

---

## 1. Why file formats?

In PART 2 we patched `vault` using `offset = address - Addr + Off`, and the two happened to be
equal. Four questions were left open:

- Why can a memory address and a position in the file differ?
- How does `objdump` know the function is called `add`?
- Where exactly does `sup3rs3cr3t` live?
- What is in the 60 bytes after the magic `7f 45 4c 46`?

Every answer lives in the **file header**.

> [!TIP]
> Analogy: a binary is a building. The **header** is the plan in the lobby (where the entrance
> is, how many floors). **Sections** are numbered rooms for the builders (kitchen, storage,
> office). **Segments** are the blocks actually moved in when the building is used. Adjacent
> rooms with the same permissions travel as one block.

An executable is not a raw pile of code. It carries a map telling the OS what is code, what is
data, where to start, and which libraries to connect.

## 2. The ELF header: the first 64 bytes

**ELF** (Executable and Linkable Format) is Linux's executable format. Its first 64 bytes are
the **ELF header** (`Elf64_Ehdr`).

```bash
xxd -l 64 vault
```

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 6010 0000 0000 0000  ..>.....`.......
00000020: 4000 0000 0000 0000 0000 0000 0000 0000  @...............
00000030: 0000 0000 4000 3800 0d00 4000 1f00 1e00  ....@.8...@.....
```

Reading the first 16 bytes (`e_ident`):

| Offset | Bytes | Meaning |
| :---: | :--- | :--- |
| 0–3 | `7f 45 4c 46` | magic: `\x7fELF` (the PART 0 lab) |
| 4 | `02` | class: `01` = 32-bit, `02` = **64-bit** |
| 5 | `01` | endianness: `01` = **little-endian**, `02` = big |
| 6 | `01` | ELF version |
| 7 | `00` | OS/ABI: `00` = System V |
| 8–15 | `00 ...` | padding |

The rest are **little-endian** numbers (PART 0 Exercise 6 — that is why `3e00` reads as `0x003e`):

| Offset | Size | Field | Example | Meaning |
| :---: | :---: | :--- | :--- | :--- |
| 16 | 2 | `e_type` | `0300` | `02` = EXEC (fixed addresses), `03` = **DYN** (PIE) |
| 18 | 2 | `e_machine` | `3e00` | `0x3e` = 62 = **x86-64** |
| 24 | 8 | `e_entry` | `6010...` | execution start address = `0x1060` |
| 32 | 8 | `e_phoff` | `4000...` | **program header** position in file = 64 |
| 40 | 8 | `e_shoff` | — | **section header** position in file |
| 52 | 2 | `e_ehsize` | `4000` | ELF header size = 64 bytes |
| 56 | 2 | `e_phnum` | `0d00` | number of program headers = 13 |
| 60 | 2 | `e_shnum` | `1f00` | number of section headers = 31 |

The human-readable version:

```bash
readelf -h vault
```

```text
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Entry point address:               0x1060
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13624 (bytes into file)
  Number of program headers:         13
  Number of section headers:         31
```

> [!IMPORTANT]
> **The entry point is not `main`.** The first thing executed is `_start` (startup code from
> libc), which sets up the runtime and only then calls `main`. Start reading at the entry point
> and you will be reading startup code, not program logic.

> [!NOTE]
> `Type: DYN` does not mean this file is a library. Modern binaries are built as **PIE**
> (Position-Independent Executable) so ASLR can randomize them. Compare with `Type: EXEC` in Lab 3.

## 3. Sections and segments

An ELF file carries **two maps for two different readers**:

| | Sections | Segments |
| :--- | :--- | :--- |
| For whom | the linker and analysis tools | the OS **loader** at runtime |
| Command | `readelf -S` | `readelf -l` |
| Contents | numbered rooms (`.text`, `.rodata`, ...) | blocks mapped into memory |
| Count | many (dozens) | few (usually 4 `LOAD` blocks) |

```text
   SECTIONS (builder's map)           SEGMENTS (mover's map)
   .interp .dynsym .dynstr .rela  ──▶  LOAD  R      (read only)
   .init .plt .text .fini         ──▶  LOAD  R E    (read + execute)
   .rodata .eh_frame              ──▶  LOAD  R      (read only)
   .data .bss .got                ──▶  LOAD  RW     (read + write)
```

```bash
readelf -SW vault      # -W = wide, one line per section (used in PART 2)
readelf -lW vault
```

```text
  Type   Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg  Align
  LOAD   0x000000 0x0000000000000000 0x0000000000000000 0x000628 0x000628 R    0x1000
  LOAD   0x001000 0x0000000000001000 0x0000000000001000 0x0001d5 0x0001d5 R E  0x1000
  LOAD   0x002000 0x0000000000002000 0x0000000000002000 0x0001ac 0x0001ac R    0x1000
  LOAD   0x002db8 0x0000000000003db8 0x0000000000003db8 0x000258 0x000260 RW   0x1000
```

Note how this matches PART 1's memory map: `R E` → Text segment, `RW` → Data/BSS. The loader
does not care about section names; it reads permissions (`Flg`) and copies blocks.

> [!IMPORTANT]
> Look at the last row: `FileSiz 0x258` but `MemSiz 0x260`. Those extra 8 bytes are **`.bss`** —
> globals without initial values. It **occupies no space in the file** (section type `NOBITS`)
> yet is allocated zero-filled in memory. That is why a file can be smaller than its memory
> footprint.

## 4. Sections you must recognize

| Section | Contents | Why it matters for RE |
| :--- | :--- | :--- |
| `.text` | machine code | the main target for disassembly and patching |
| `.rodata` | constants and string literals (**read-only**) | where hardcoded passwords live |
| `.data` | globals/statics with initial values | configuration, mutable variables |
| `.bss` | globals/statics without initial values | absent from the file, present in memory |
| `.plt` / `.got` | bridge to library functions | explains `call strcmp@plt` |
| `.dynsym` / `.dynstr` | symbols and names for dynamic linking | **survive** `strip` |
| `.symtab` / `.strtab` | full symbol and name tables | the source of `add`, `main` labels in `objdump` |
| `.rela.dyn` / `.rela.plt` | relocation records | where the loader must patch addresses |
| `.init` / `.fini` | code running before/after `main` | a place to hide logic (Phase 8) |
| `.comment` | compiler version | quick fingerprint: which GCC built this |
| `.debug_*` | debug info (DWARF), only with `-g` | local variable names and line numbers |

> [!TIP]
> A fast triage order on an unknown binary: `readelf -h` (type and architecture) → `readelf -SW`
> (which sections exist) → `strings -t x` (interesting text plus offsets) → `nm` (any names left?)
> → only then `objdump`.

## 5. Memory address vs file offset

There are **two coordinate systems** you must never mix up:

| | File offset | Virtual address (VMA) |
| :--- | :--- | :--- |
| Counted from | byte 0 of the **file** | the load base in **memory** |
| Used by | `xxd`, `dd`, hex editors | `objdump`, `gdb` |
| Example | `0x11b6` | `0x5555555551b6` (at runtime) |

The conversion, using section data:

```text
file offset = address - sh_addr + sh_offset
              (Addr)          (Off)      ← two columns from readelf -SW
```

Or using segments (more accurate for what is actually loaded):

```text
file offset = address - p_vaddr + p_offset
```

Why were they equal in PART 2? Because the loader requires `p_offset` and `p_vaddr` to be
**congruent modulo the page size** (`0x1000`). In small binaries this often makes `Addr == Off`
for `.text` — a coincidence you must **prove**, not assume.

### PIE vs non-PIE

| | `-no-pie` (EXEC) | modern default (DYN / PIE) |
| :--- | :--- | :--- |
| Runtime address | fixed, e.g. `0x401136` | random base + offset, e.g. `0x555555555136` |
| ASLR | does not randomize code | randomizes the whole binary |
| Example in these notes | `sub_401136` in PART 0 | `0x5555555551b6` in PART 1 |

> [!IMPORTANT]
> To patch the **file**, always use the **file offset**, never a runtime address. The address
> `0x5555555551b6` from gdb means nothing to `dd`; what matters is `0x11b6`.

## 6. Symbols and stripping

A **symbol** is a name tied to an address: a function (`main`, `add`) or a global variable.
Symbols live in `.symtab`, their name strings in `.strtab`.

```bash
nm vault | grep ' T '     # T = global code (text) symbol
readelf -sW vault | head -20
```

```text
0000000000001135 T add
000000000000114f T main
```

`strip` throws those tables away:

```bash
cp vault vault.stripped
strip --strip-all vault.stripped
nm vault.stripped
```

```text
nm: vault.stripped: no symbols
```

> [!IMPORTANT]
> This **refines** PART 0's claim that "the compiler throws names away". More precisely:
> - machine code genuinely does not use names — the compiler emits addresses and offsets;
> - **function names may still be stored** in `.symtab` until the file is `strip`ped;
> - **local variable names** only exist if you compiled with `-g` (DWARF debug info);
> - library function names (`strcmp`) remain in `.dynsym` even after stripping, because dynamic
>   linking needs them at runtime.

That is why `objdump` still shows `<strcmp@plt>` on a stripped binary, while `<main>` becomes a
bare address like `0x114f`.

## 7. Dynamic linking: PLT and GOT

`vault` does not contain `strcmp`'s code. That code lives in **libc**, a shared library loaded
when the program runs.

```bash
readelf -dW vault | grep NEEDED
ldd vault
```

```text
 0x0000000000000001 (NEEDED)  Shared library: [libc.so.6]
	linux-vdso.so.1 (0x00007ffd...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

The chain behind `call 1050 <strcmp@plt>`:

```text
call .plt  ──▶  jmp [GOT]  ──▶  strcmp's address in libc
   (stub)        (pointer table, filled by the loader)
```

- **PLT** (Procedure Linkage Table) = a small code stub in `.plt`, one per library function.
- **GOT** (Global Offset Table) = a pointer table in `.got`; the loader writes real addresses
  here. By default this is *lazy binding*: filled on first call.

> [!WARNING]
> `ldd` runs the dynamic loader, so on an **untrusted** binary it can execute code. For malware
> samples use `readelf -d` or `objdump -p`, which only read the file.

> [!TIP]
> A writable GOT is why techniques like *GOT overwrite* exist. Park that term; it becomes
> relevant in Phases 5 and 9.

## 8. PE (Windows) in brief

Windows uses **PE** (Portable Executable). Same concepts, different field names:

| Concept | ELF (Linux) | PE (Windows) |
| :--- | :--- | :--- |
| Magic | `7f 45 4c 46` (`.ELF`) | `4d 5a` (`MZ`) then `50 45 00 00` (`PE\0\0`) |
| Header | ELF header (64 bytes) | DOS header → `e_lfanew` → PE header |
| Execution start | `e_entry` | `AddressOfEntryPoint` (RVA) |
| Address base | random if PIE | `ImageBase` (usually `0x140000000` for EXEs) |
| Code | `.text` | `.text` |
| Constants/strings | `.rodata` | `.rdata` |
| Globals | `.data` / `.bss` | `.data` |
| Resources | — | `.rsrc` (icons, dialogs, version info) |
| Relocations | `.rela.*` | `.reloc` |

PE addresses are **RVAs** (Relative Virtual Address) — relative to `ImageBase`:

```text
runtime address = ImageBase + RVA
file offset     = RVA - VirtualAddress + PointerToRawData
```

That second formula is **exactly** the ELF formula from section 5 with different column names.

```bash
objdump -x sample.exe | head -40      # works for PE too
```

Other tools: `pev`/`pedump` (Linux), CFF Explorer or Detect It Easy (Windows).

> [!NOTE]
> The labs here use ELF because our lab is Linux. The mindset is identical for PE: read the
> header → find the section → convert the address → only then patch.

## 9. Hands-on: reading ELF

### 9.1 Build the targets

```bash
gcc -g -O0 vault.c -o vault
gcc -g -O0 frame.c -o frame
cp vault vault.bak
```

> [!NOTE]
> `bash` blocks = terminal, `text` = output. All numbers below are **examples**; recompute them
> on your machine and always verify with `xxd` before writing anything.

### 9.2 Lab 1 — Translate the header by hand

**Goal:** prove that PART 0's "magic bytes" are only the start of a structure.

```bash
readelf -h vault
xxd -l 16 vault
```

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
```

Match it against the table in section 2, then answer: 32 or 64 bit? Little or big endian? EXEC
or DYN? How many sections? All of it is written in those 64 bytes.

> [!IMPORTANT]
> **THE AHA:** A header is not magic — it is a C struct written raw into a file. Once you can
> read an offset table, you can read any format (ELF, PE, PNG, ZIP).

### 9.3 Lab 2 — Find the secret's home

**Goal:** know exactly where `sup3rs3cr3t` lives, not just "strings found it".

```bash
strings -t x vault | grep sup3rs3cr3t
readelf -SW vault | grep -E '\.rodata|\.text'
```

```text
   2010 sup3rs3cr3t
  [16] .rodata PROGBITS 0000000000002000 002000 000042 00   A  0  0  4
```

`strings -t x` prints the **file offset** in hex. Offset `0x2010` falls inside `.rodata`, which
starts at `0x2000` and is `0x42` bytes long — consistent. Look inside:

```bash
readelf -p .rodata vault
xxd -s 0x2010 -l 16 vault
```

```text
String dump of section '.rodata':
  [     4]  Enter the vault password:
  [    20]  sup3rs3cr3t
  [    2c]  %63s
  [    31]  ACCESS GRANTED
  [    40]  ACCESS DENIED
00002010: 7375 7033 7273 3363 7233 7400 2536 3373  sup3rs3cr3t.%63s
```

Notice the `00` after `sup3rs3cr3t`: that is the **null terminator**, C's end-of-string marker.

> [!IMPORTANT]
> **THE AHA:** `strings` is not a magic tool — it just sweeps the file for runs of printable
> characters. The secret is there because the compiler stores string literals verbatim in
> `.rodata`. Every string you patched in PART 0 (`ACCESS DENIED`) lives here too.

### 9.4 Lab 3 — Prove the address-to-offset formula

**Goal:** stop guessing offsets when patching, and see what PIE changes.

```bash
gcc -g -O0 -no-pie vault.c -o vault.nopie
readelf -h vault.nopie | grep -E 'Type|Entry'
objdump -d -M intel vault.nopie | grep -A3 'strcmp@plt'
readelf -SW vault.nopie | grep '\.text'
```

```text
  Type:                              EXEC (Executable file)
  Entry point address:               0x401050
    401196:	e8 c5 fe ff ff       	call   401030 <strcmp@plt>
    40119b:	85 c0                	test   eax,eax
    40119d:	75 0e                	jne    4011ad <main+0x61>
  [14] .text PROGBITS 0000000000401050 001050 0001a2 00  AX  0  0  16
```

Compute: `offset = 0x40119d - 0x401050 + 0x1050 = 0x119d`. Prove it before writing:

```bash
xxd -s 0x119d -l 2 vault.nopie
```

```text
0000119d: 750e                                     u.
```

If `750e` shows up, the formula holds. Compare with the PIE build:

```bash
readelf -h vault | grep Type
```

```text
  Type:                              DYN (Position-Independent Executable file)
```

> [!NOTE]
> Those `0x401...` addresses in a non-PIE build explain `sub_401136` from PART 0's example. That
> number is not a universal constant — it is the signature of an EXEC binary without ASLR.

> [!IMPORTANT]
> **THE AHA:** `Addr` and `Off` are often equal, but the **reason** is page congruence, not luck.
> With this formula you can patch any binary, including ones where address and offset differ
> substantially.

### 9.5 Lab 4 — The names that vanish (and the ones that survive)

**Goal:** see for yourself what `strip` removes and what stays.

```bash
nm frame | grep ' T '
objdump -d -M intel frame | grep -E '^[0-9a-f]+ <' | head -5
cp frame frame.stripped
strip --strip-all frame.stripped
nm frame.stripped
objdump -d -M intel frame.stripped | grep -E '^[0-9a-f]+ <' | head -5
ls -l frame frame.stripped
```

```text
0000000000001135 T add
000000000000114f T main
0000000000001135 <add>:
000000000000114f <main>:
nm: frame.stripped: no symbols
0000000000001040 <printf@plt>:
-rwxr-xr-x 1 user user 17K frame
-rwxr-xr-x 1 user user 14K frame.stripped
```

The `add` function is still there — not one byte of its code changed. Only its **name label** is
gone.

> [!IMPORTANT]
> **THE AHA:** Stripping deletes the name map, not the logic. A reverser can still read the
> function and then **give it a new name** (`sub_401136` → `verify_password`). That is daily work
> in Phases 4 and 6.

## 10. Recap

| Concept | What to remember |
| :--- | :--- |
| ELF header | 64 bytes: magic, class, endianness, EXEC/DYN, entry point, table positions |
| Entry point | `_start`, not `main` |
| Sections | the linker/tool map: `.text`, `.rodata`, `.data`, `.bss`, `.symtab` |
| Segments | the loader map: `LOAD` blocks with `R` / `R E` / `RW` permissions |
| `.bss` | `NOBITS`: no file space, exists in memory (`MemSiz > FileSiz`) |
| Two coordinates | file offset (`xxd`/`dd`) vs virtual address (`objdump`/`gdb`) |
| Conversion | `offset = address - Addr + Off` (section) or `- p_vaddr + p_offset` (segment) |
| PIE vs EXEC | PIE = ASLR-randomized addresses; EXEC (`-no-pie`) = fixed `0x401...` |
| Symbols | `.symtab` removed by `strip`; `.dynsym` survives → `strcmp@plt` still visible |
| PLT/GOT | the bridge to libraries; the GOT is filled by the loader at runtime |
| PE | same ideas: `MZ` → `PE\0\0`, RVA, `ImageBase`, `.rdata`/`.rsrc` |

### Self-check

You are ready for Phase 4 if you can answer:

- [ ] What do bytes 4 and 5 of the ELF header mean?
- [ ] Why is the entry point not `main`, and what lives there?
- [ ] How do sections and segments differ, and who reads each?
- [ ] Why can `MemSiz` be larger than `FileSiz`?
- [ ] Which section holds string literals, and why can `strings` find them?
- [ ] How do you turn address `0x40119d` into a file offset — and why must you never feed a
      runtime address to `dd`?
- [ ] What is lost and what survives after `strip`?
- [ ] What do the PLT and GOT do when the program calls `strcmp`?

## 11. Cliffhanger: from file to C code

You now have the full map: header, sections, symbols, and offset arithmetic. But every analysis
so far was **line by line**, by hand.

Next question: if the names are gone and the program has 500 functions, do we read them all one
at a time?

No. Some tools read this ELF structure for us, map every function, build control-flow graphs,
and then **guess the C code back**.

> [!IMPORTANT]
> That is **Phase 4 — Static Analysis** with Ghidra: from reading bytes to reading *decompiled
> C*, and renaming functions that lost their names.

## 12. Glossary

| Term | Plain meaning |
| :--- | :--- |
| ELF | Linux executable format (Executable and Linkable Format) |
| PE | Windows executable format (Portable Executable) |
| Magic bytes | leading bytes identifying a file type (ELF = `7f 45 4c 46`, PE = `MZ`) |
| ELF header | first 64 bytes: file identity plus positions of its tables |
| `e_type` | executable kind: `EXEC` (fixed addresses) or `DYN` (PIE) |
| Entry point | first address executed (`_start`, not `main`) |
| Section | a named "room" in the file, for the linker and analysis tools |
| Segment | a block the loader actually maps into memory, with permissions |
| `LOAD` | segment type whose contents are copied into memory |
| `.text` | section holding machine code |
| `.rodata` | section for constants and string literals, read-only |
| `.data` / `.bss` | globals with initial values / without (zero-filled) |
| `NOBITS` | section type storing no bytes in the file (`.bss`) |
| `FileSiz` / `MemSiz` | size in the file / size in memory |
| `.symtab` / `.strtab` | symbol table and its name strings |
| `.dynsym` / `.dynstr` | symbols for dynamic linking, survive stripping |
| Symbol | a name tied to an address (function or variable) |
| `nm` | tool listing a binary's symbols |
| `strip` | removes symbol tables and debug info from a file |
| Stripped | a binary without symbol tables |
| DWARF / `.debug_*` | debug info (`-g`): local variable names and line numbers |
| Relocation | a record of addresses the loader must patch when loading |
| PLT | Procedure Linkage Table: stub code calling library functions |
| GOT | Global Offset Table: pointer table filled by the loader |
| Lazy binding | GOT filling deferred until a function is first called |
| Dynamic linking | connecting to libraries at runtime instead of at compile time |
| `ldd` | lists required libraries (careful with untrusted files) |
| PIE | a binary loadable at random addresses (supports ASLR) |
| ASLR | OS randomization of memory addresses |
| File offset | a byte position counted from the start of the file |
| VMA | virtual address while the program runs |
| RVA | address relative to `ImageBase` (PE terminology) |
| `ImageBase` | PE's preferred load address |
| `.rsrc` / `.reloc` | PE resource / relocation sections |
| `readelf` | tool for reading ELF headers, sections, segments, symbols |

---

[← PART 2](../PART2/note.md) · [Roadmap](../INDEX.md) · **Next:** PART 4 — Static Analysis
([ID notes](../PART4/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>