# PART 4 — Static Analysis

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 4 of 10**

> [!NOTE]
> **Source:** [PART 4 on YouTube](https://www.youtube.com/watch?v=sbGsMc9_qvg)
> Technical terms stay in English, but every term is explained in plain language first.
> Glossary: [section 12](#12-glossary).

**Prerequisites:** [PART 3](../PART3/note.md) done — you know sections, symbols, `strip`, and the
difference between a memory address and a file offset.

**After this part you can:** import a binary into Ghidra, navigate its three main panels,
**find the important function even when names are stripped** (via strings + XREFs), read and
clean up pseudo-C, and patch an instruction straight from the GUI.

## Table of contents

- [PART 4 — Static Analysis](#part-4--static-analysis)
  - [Table of contents](#table-of-contents)
  - [1. Why you need a decompiler](#1-why-you-need-a-decompiler)
  - [2. What Ghidra is](#2-what-ghidra-is)
  - [3. Installing Ghidra](#3-installing-ghidra)
  - [4. Projects and importing](#4-projects-and-importing)
    - [The import summary is the ELF header, read for you](#the-import-summary-is-the-elf-header-read-for-you)
  - [5. Auto-analysis](#5-auto-analysis)
  - [6. The three Code Browser panels](#6-the-three-code-browser-panels)
  - [7. Reading Ghidra pseudo-code](#7-reading-ghidra-pseudo-code)
    - [XREFs: the static analyst's main weapon](#xrefs-the-static-analysts-main-weapon)
  - [8. Refactoring: rename, retype, comment](#8-refactoring-rename-retype-comment)
  - [9. Hands-on: breaking vault in Ghidra](#9-hands-on-breaking-vault-in-ghidra)
    - [9.1 Build the targets](#91-build-the-targets)
    - [9.2 Lab 1 — Import and read the metadata as an ELF header](#92-lab-1--import-and-read-the-metadata-as-an-elf-header)
    - [9.3 Lab 2 — Find the main function without a name](#93-lab-2--find-the-main-function-without-a-name)
    - [9.4 Lab 3 — Read the pseudo-C, then clean it up](#94-lab-3--read-the-pseudo-c-then-clean-it-up)
    - [9.5 Lab 4 — Patch an instruction from the GUI](#95-lab-4--patch-an-instruction-from-the-gui)
    - [9.6 Navigation cheat sheet](#96-navigation-cheat-sheet)
  - [10. Recap](#10-recap)
    - [Self-check](#self-check)
  - [11. Cliffhanger: when the decompiler guesses wrong](#11-cliffhanger-when-the-decompiler-guesses-wrong)
  - [12. Glossary](#12-glossary)

---

## 1. Why you need a decompiler

So far we have read everything by hand, line by line:

| Phase | What we did | Tools |
| :---: | :--- | :--- |
| 0 | Broke the secret without assembly | `strings`, `ltrace`, `sed` |
| 1 | Understood registers, flags, stack frames | `gdb` |
| 2 | Read instructions and patched bytes | `objdump`, `dd` |
| 3 | Read the file map: sections, symbols, offsets | `readelf`, `nm` |

`vault` has one function of real logic. Real programs have **thousands**. Reading them all with
`objdump` takes days, and you drown before reaching the part that matters.

> [!TIP]
> Analogy: PART 2 was reading a building **brick by brick**. Ghidra hands you the **full floor
> plan plus a room list**, then guesses "this room looks like a kitchen". You still verify the
> guess, but you no longer count bricks.

What we need: a tool that maps every function at once, then translates assembly back into
C-like code. That is a **decompiler**.

## 2. What Ghidra is

**Ghidra** is a free, open-source reverse engineering framework released by the NSA (the US
security agency). Its two core abilities:

| Ability | What it does | Our manual equivalent |
| :--- | :--- | :--- |
| **Disassembler** | maps every instruction, function, and cross-reference | `objdump -d` (PART 2) |
| **Decompiler** | guesses C-like code from assembly | none — this is new |

> [!IMPORTANT]
> The decompiler's output is **pseudo-code**: C-like, but **not the original source**. Variable
> names, data types, and sometimes control structures are Ghidra's **guesses**. The real source
> was thrown away by the compiler (PART 0) and cannot be recovered exactly.

Because the analysis is static, the binary is **never executed** — the safest way to dissect a
dangerous sample. Still work in a VM (PART 0 discipline): analysis tools themselves can have
bugs when parsing hostile files.

## 3. Installing Ghidra

Ghidra needs a **Java JDK** (17 or 21 depending on the release — check that version's release
notes) and a graphical display.

<details>
<summary><b>APT</b> — Kali Linux (package available)</summary>

```bash
sudo apt update
sudo apt install ghidra
ghidra
```

</details>

<details>
<summary><b>Pacman</b> — Arch Linux</summary>

```bash
sudo pacman -S ghidra
ghidraRun
```

</details>

<details>
<summary><b>Manual</b> — any distro (Fedora, Void, Debian, ...)</summary>

Download the official release (a `.zip`) from
[github.com/NationalSecurityAgency/ghidra/releases](https://github.com/NationalSecurityAgency/ghidra/releases),
then install it under `/opt`:

```bash
# JDK first (pick your package manager)
sudo apt install openjdk-21-jdk        # Debian/Ubuntu/Kali
sudo pacman -S jdk21-openjdk           # Arch
java -version                          # make sure it is on PATH

# put Ghidra in /opt, then add a shortcut to PATH
sudo unzip ghidra_*_PUBLIC_*.zip -d /opt/
sudo ln -s /opt/ghidra_*_PUBLIC/ghidraRun /usr/local/bin/ghidra
ghidra
```

</details>

> [!NOTE]
> Ghidra is not in every distro's official repos. If your package manager lacks it, the manual
> route is actually cleaner: one folder in `/opt`, one symlink, and trivial to remove
> (`rm -rf /opt/ghidra_*` plus deleting the symlink).

> [!TIP]
> For automation later, Ghidra ships a headless mode: `support/analyzeHeadless` runs analysis
> and scripts from the terminal, no GUI.

## 4. Projects and importing

Ghidra works with **projects** rather than opening files directly.

1. **File → New Project** → choose **Non-Shared Project** → Next.
2. Pick a **Project Directory** (where the analysis database lives) and a **Project Name**,
   e.g. `vault-lab` → Finish.
3. **File → Import File** → select the `vault` binary → OK.

> [!IMPORTANT]
> The project stores **your analysis** (new names, comments, types) in a separate database.
> The original binary is **not modified**. This differs from `sed`/`dd` in PART 0–2, which wrote
> straight into the file.

### The import summary is the ELF header, read for you

After import, Ghidra shows a summary:

| Attribute | Value | Where it comes from (PART 3) |
| :--- | :--- | :--- |
| Language ID | `x86:LE:64:default` | `e_machine` + endianness + `EI_CLASS` |
| Processor | x86 | `e_machine = 0x3e` |
| Endian | Little | ELF header byte 5 = `01` |
| Address size | 64 | ELF header byte 4 = `02` |
| Compiler | GCC | inferred from code patterns and `.comment` |
| Format | ELF | magic `7f 45 4c 46` |

Read `x86:LE:64:default` as: **x86** processor, **L**ittle-**E**ndian, **64**-bit, default variant.

> [!IMPORTANT]
> No magic happens at this stage — Ghidra reads the same first 64 bytes you translated by hand
> in PART 3 Lab 1. If the Language ID looks wrong, the header really is unusual (or the file is
> not a plain ELF).

## 5. Auto-analysis

Right-click `vault` in the project window → **Open with Code Browser**. When asked
"…has not been analyzed. Would you like to analyze it now?" → **Yes** → keep the **default**
options → **Analyze**.

What Ghidra does behind the scenes:

- reads sections and segments (the PART 3 map) to decide what is code and what is data;
- follows control flow from the entry point and callees, carving code into **functions**;
- recognizes prologue/epilogue patterns (PART 1) and library function signatures;
- builds **XREFs** (cross-references): who calls whom, and who uses which data;
- runs the decompiler on every function.

> [!NOTE]
> Small binaries analyze in seconds; large ones can take minutes. Defaults are fine for 95% of
> cases; advanced options matter mainly against deliberately obscured code (Phase 8).

## 6. The three Code Browser panels

| Panel | Position | Contents | Analogy |
| :--- | :--- | :--- | :--- |
| **Symbol Tree** | left | Functions, Imports, Exports, Labels, Data Type Manager | the table of contents |
| **Listing** | center | assembly per address, raw bytes, labels, comments | the actual pages |
| **Decompiler** | right | pseudo-C for the selected function | the human-language summary |

All three are **synchronized**: click a function in the Symbol Tree → the Listing jumps to its
address → the Decompiler shows its pseudo-C. Click a line in the Decompiler → the matching
assembly line is highlighted in the Listing.

Supporting windows worth knowing:

| Window | How to open | Purpose |
| :--- | :--- | :--- |
| **Defined Strings** | Window → Defined Strings | every recognized string plus its address (a GUI `strings`, PART 0) |
| **Function Graph** | graph icon in the Listing toolbar | one function's branching as block diagrams |
| **Function Call Graph** | Window → Function Call Graph | who calls whom, one level deep |
| **Memory Map** | Window → Memory Map | sections/segments and image base (PART 3) |

> [!TIP]
> If the Decompiler panel is missing: **Window → Decompile**. Panels can be rearranged by
> dragging; there is no risk of damaging the binary.

## 7. Reading Ghidra pseudo-code

Ghidra's automatic names are consistent. Learn the pattern and pseudo-C becomes readable:

| Name | Meaning |
| :--- | :--- |
| `FUN_00101165` | an unnamed function; the digits are its **address** |
| `LAB_001011c6` | a label, i.e. a jump target inside a function |
| `DAT_0010202b` | unnamed data at that address |
| `PTR_...` | data holding a pointer |
| `local_58` | a local variable; the hex digits are its stack frame offset |
| `param_1` | the function's first argument |
| `iVar1` / `uVar1` | a signed / unsigned temporary variable |
| `undefined8` / `undefined4` | unknown type, 8 / 4 bytes wide |
| `__isoc99_scanf` | the real libc symbol name (the C99 variant of `scanf`) |

Two things that confuse beginners:

> [!NOTE]
> - **Ghidra writes `JNZ`, `objdump` writes `jne`.** Same instruction, different spelling —
>   like the Intel vs AT&T situation in PART 2. Other pairs: `JZ` = `je`, `JLE` = `jle`.
> - **Ghidra addresses are not runtime addresses.** For ELF PIE files Ghidra uses a default
>   *image base* of `0x00100000`. So `FUN_00101165` = `0x1165` in `objdump` (PART 0) =
>   `0x555555555165` when running under gdb (PART 1). Three numbers, one place.

### XREFs: the static analyst's main weapon

An **XREF (cross-reference)** is the list of references to an address: who calls this function,
or which instruction uses this data.

The trick: **you can walk backwards**. From an interesting string → see who uses it → arrive at
the relevant function. Function names may be gone; strings usually are not.

## 8. Refactoring: rename, retype, comment

Three actions that turn pseudo-C from "machine guesswork" into "notes a human can read":

| Action | How | Effect |
| :--- | :--- | :--- |
| **Rename** | right-click a name → Rename Variable/Function (`L`) | `iVar1` → `match`, `FUN_00101165` → `main` |
| **Retype** | right-click → Retype (`Ctrl+L`) | `undefined8` → `int`, `local_58` → `char[64]` |
| **Comment** | right-click a line → Set Comment (`;`) | your note appears in **both** Listing and Decompiler |

> [!IMPORTANT]
> Renaming, retyping, and commenting change only the **project database**, not the binary.
> Prove it: `sha256sum vault` before and after — identical hash. The only action that changes
> bytes is **patching** (Lab 4), and even that reaches the file only when you export.

> [!TIP]
> Name things by **behavior**, not by guesswork: `check_password`, `decrypt_config`,
> `send_beacon`. Good names are documentation that survives until you come back next week.

## 9. Hands-on: breaking vault in Ghidra

### 9.1 Build the targets

```bash
gcc -g -O0 vault.c -o vault
cp vault vault.stripped
strip --strip-all vault.stripped
sha256sum vault vault.stripped
```

We deliberately use the **stripped** copy for part of the lab so you experience the real
condition: function names gone (PART 3 Lab 4), logic still there.

> [!NOTE]
> All names, addresses, and screenshots below are **examples**. Different Ghidra and gcc
> versions may produce slightly different names or offsets. The pattern is what matters.

### 9.2 Lab 1 — Import and read the metadata as an ELF header

**Goal:** prove the import summary is just PART 3, read automatically.

Import `vault.stripped` into the project → note Language ID, Endian, Address size, Format →
compare with:

```bash
readelf -h vault.stripped | grep -E 'Class|Data|Machine|Type'
```

```text
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
```

Match them one by one against the table in section 4.

> [!IMPORTANT]
> **THE AHA:** Ghidra is not a black box. Every column in the import summary traces back to
> bytes you already read by hand in PART 3.

### 9.3 Lab 2 — Find the main function without a name

**Goal:** locate `main` in a stripped binary without guessing addresses.

1. **Window → Defined Strings**.
2. Find `sup3rs3cr3t` → double-click. The Listing jumps to address `00102010` (in `.rodata`).
3. Right-click that address → **References → Show References to Address**.
4. One reference appears from `FUN_00101165` → double-click to jump there.

```text
Address     String
00102010    sup3rs3cr3t

XREF to 00102010:
  FUN_00101165  0010116d  LEA RDI=>s_sup3rs3cr3t_00102010
```

Compare with PART 3: that string really does live in `.rodata` at offset `0x2010`, and `0x1165`
is the address of `main` back when it still had a name.

Now name it: click `FUN_00101165` → press `L` → type `main` → OK.

> [!TIP]
> Alternative route when strings are encrypted: look at **Symbol Tree → Imports** for
> interesting callees (`strcmp`, `memcmp`, `strstr`) and check their XREFs. Comparison functions
> are almost always called from wherever the decision is made.

> [!IMPORTANT]
> **THE AHA:** Names can be stripped, but **references are hard to hide**. String → XREF →
> function is the fastest path to a program's core, and it is pure static analysis: the program
> never ran.

### 9.4 Lab 3 — Read the pseudo-C, then clean it up

**Goal:** read the decompiler's output and turn it into human notes.

Click `main` → the Decompiler shows (example):

```c
undefined8 main(void)
{
  int iVar1;
  char local_58 [64];
  char *local_10;

  local_10 = "sup3rs3cr3t";
  printf("Enter the vault password: ");
  __isoc99_scanf(&DAT_0010202b,local_58);
  iVar1 = strcmp(local_58,local_10);
  if (iVar1 == 0) {
    puts("ACCESS GRANTED");
  }
  else {
    puts("ACCESS DENIED");
  }
  return 0;
}
```

Compare with the original source in PART 0. Four normal differences:

| Difference | Why |
| :--- | :--- |
| `undefined8 main(void)` | Ghidra does not know the return type yet (`int`) |
| `local_58`, `local_10` | variable names were discarded; these are stack offsets (PART 1) |
| `local_10 = "sup3rs3cr3t"` then used | the compiler spilled the pointer to the stack first (`[rbp-0x8]`, PART 0) |
| `&DAT_0010202b` | the format string `"%63s"` has no name yet |

Clean it up with rename + retype:

| Before | After | Action |
| :--- | :--- | :--- |
| `undefined8 main` | `int main` | retype return (`Ctrl+L`) |
| `iVar1` | `match` | rename (`L`) |
| `local_58` | `input` | rename |
| `local_10` | `secret` | rename |
| `DAT_0010202b` | `fmt_password` | rename label |

Result:

```c
int main(void)
{
  int match;
  char input [64];
  char *secret;

  secret = "sup3rs3cr3t";
  printf("Enter the vault password: ");
  __isoc99_scanf(&fmt_password,input);
  match = strcmp(input,secret);
  if (match == 0) {
    puts("ACCESS GRANTED");
  }
  else {
    puts("ACCESS DENIED");
  }
  return 0;
}
```

Add a comment (`;`) on the `strcmp` line: `authentication decision — see PART 2 Lab 4`.

```bash
sha256sum vault.stripped
```

The hash is **unchanged** since 9.1.

> [!IMPORTANT]
> **THE AHA:** Decompilation is not recovering source code, it is **building understanding**.
> The reverser's job is replacing machine guesses with meaningful names — all stored in the
> project, not in the binary.

### 9.5 Lab 4 — Patch an instruction from the GUI

**Goal:** perform the `75` → `74` patch from PART 2 Lab 4, but through Ghidra.

1. In the Listing, jump to address `001011b6` (press `G`, type the address).
2. You will see `JNZ LAB_001011c6` — the same `jne` from PART 2.
3. Right-click the instruction → **Patch Instruction** (`Ctrl+Shift+H`) → change `JNZ` to `JZ` → Enter.
4. Watch the Decompiler: the condition is now inverted (`if (match != 0)`).
5. Save the project (`Ctrl+S`), then **File → Export Program** → format **Original File** →
   save as `vault.ghidra-patched`.

Verify in the terminal:

```bash
xxd -s 0x11b6 -l 2 vault.stripped
xxd -s 0x11b6 -l 2 vault.ghidra-patched
chmod +x vault.ghidra-patched
./vault.ghidra-patched
```

```text
000011b6: 750e                                     u.
000011b6: 740e                                     t.
Enter the vault password: nope
ACCESS GRANTED
```

> [!NOTE]
> Export format names vary between Ghidra versions. When in doubt, use the guaranteed method
> from PART 2: take the offset from Ghidra, then `printf '\164' | dd ...`. Ghidra to **find**,
> `dd` to **write**.

> [!WARNING]
> Patch Instruction changes bytes in the **project database** first; the original file changes
> only when you export. Export under a new name so the original binary stays intact.

> [!IMPORTANT]
> **THE AHA:** GUI and command line do exactly the same thing: replace one byte. The difference
> is that Ghidra shows you **the effect on the pseudo-C** immediately, so you know which
> decision you are inverting.

### 9.6 Navigation cheat sheet

| Shortcut | Action |
| :--- | :--- |
| `G` | Go To address |
| `L` | rename label / function / variable |
| `Ctrl+L` | retype (change data type) |
| `;` | set comment |
| `Ctrl+Shift+H` | patch instruction |
| `Ctrl+S` | save project |
| `Ctrl+Z` | undo |
| double-click | jump to a reference / function |

> [!NOTE]
> These are default bindings and may differ across versions or if you customized them. Every
> action is always available through the right-click menu.

## 10. Recap

| Concept | What to remember |
| :--- | :--- |
| Why Ghidra | manual analysis does not scale to thousands of functions |
| Disassembler vs decompiler | mapping instructions vs guessing C-like code |
| Pseudo-code | a guess, not the original source; names and types can be wrong |
| Project | stores your analysis; the binary is untouched |
| Import summary | the ELF header (PART 3), read automatically |
| Three panels | Symbol Tree (contents), Listing (assembly), Decompiler (pseudo-C) |
| Auto names | `FUN_`, `LAB_`, `DAT_`, `local_`, `undefined8` |
| XREF | the backwards path: string → who uses it → core function |
| Refactoring | rename/retype/comment = documentation, not binary modification |
| Patching | the only byte-changing action; export to write it to a file |
| Static limits | no runtime values → Phase 5 |

### Self-check

You are ready for Phase 5 if you can answer:

- [ ] How do disassembler and decompiler differ, and why is pseudo-code not the real source?
- [ ] Where does Ghidra get the Language ID `x86:LE:64:default`?
- [ ] What are the Symbol Tree, Listing, and Decompiler for?
- [ ] How do you find the main function in a stripped binary without guessing addresses?
- [ ] What do `FUN_00101165`, `local_58`, and `undefined8` mean?
- [ ] Why is the binary's `sha256sum` unchanged after renaming every variable?
- [ ] Why does Ghidra write `JNZ` while `objdump` writes `jne`?
- [ ] What information can static analysis **never** give you?

## 11. Cliffhanger: when the decompiler guesses wrong

Ghidra looks very confident. But notice what it cannot know:

- Register contents **while the program runs** — it sees code, not execution.
- Values that only exist at runtime: decryption results, network data, user input.
- The true data types when code is optimized (`-O2`, PART 0 Exercise 7) or obscured.
- Whether a branch is ever actually taken.

A concrete example: pseudo-C shows `strcmp(input, secret)`, but if `secret` is filled by a
decryption routine at runtime, Ghidra will only show `FUN_001012a0(...)` — and the file alone
will never tell you its contents.

> [!IMPORTANT]
> That is **Phase 5 — Dynamic Analysis**: run the program under `gdb`, break at the addresses
> you already found in Ghidra, and read the real values. Static gives you the map; dynamic gives
> you the truth.

> [!NOTE]
> When a program is deliberately hard to analyze — **obfuscation** (scrambled code), **packing**
> (compressed/encrypted code), and **anti-analysis** (debugger detection) — that is **Phase 8**,
> not now. Our `vault` has no protection at all.

## 12. Glossary

| Term | Plain meaning |
| :--- | :--- |
| Ghidra | free, open-source RE framework from the NSA: disassembler + decompiler |
| Disassembler | turns machine code into assembly |
| Decompiler | guesses high-level, C-like code from assembly |
| Pseudo-code | decompiler output: C-like, but a guess, not the real source |
| Project | the database where Ghidra stores your analysis |
| Code Browser | the main working window for one binary |
| Auto-analysis | automatic mapping of functions, data, and references |
| Symbol Tree | panel listing functions, imports, labels, and data types |
| Listing | panel showing assembly per address, with raw bytes |
| Decompiler window | panel showing pseudo-C for the selected function |
| Defined Strings | list of recognized strings with their addresses |
| XREF (cross-reference) | the list of "who uses this address" |
| Function Graph | block diagram of one function's branching |
| Call Graph | diagram of who calls whom |
| Image base | base address where Ghidra maps the binary (default `0x00100000`) |
| `FUN_` | automatic name for an unnamed function |
| `LAB_` | automatic name for a label (jump target) |
| `DAT_` | automatic name for unnamed data |
| `local_` | a local variable, named from its stack offset |
| `param_` | a function argument |
| `undefined8` | unknown type, 8 bytes wide |
| Rename | giving functions/variables readable names |
| Retype | correcting a data type the decompiler guessed |
| Refactoring | cleaning up names and structure without changing behavior |
| Patch Instruction | replacing an instruction (bytes) directly in Ghidra |
| Export Program | writing your changes out to a new file |
| Stripped | a binary without symbol tables (PART 3) |
| Headless analyzer | Ghidra's GUI-less mode for automation |
| Obfuscation | scrambling code to make it hard to read (Phase 8) |
| Packing | compressing/hiding the real code inside a file (Phase 8) |
| Anti-analysis | detecting debuggers/sandboxes and evading them (Phase 8) |

---

[← PART 3](../PART3/note.md) · [Roadmap](../INDEX.md) · **Next:** PART 5 — Dynamic Analysis
([ID notes](../PART5/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>