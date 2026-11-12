# reverse-engineering

**English** · [Bahasa Indonesia](README.id.md)

My journey learning **Reverse Engineering (RE)** — taking a program apart to understand how
it works when all you have is the compiled binary, not the source code. Notes are written
from zero, every part comes with hands-on practice, and every technical term is explained
in plain language before it gets used. Beginner friendly.

## Contents

- [reverse-engineering](#reverse-engineering)
  - [Contents](#contents)
  - [Who is this for](#who-is-this-for)
  - [How to use these notes](#how-to-use-these-notes)
  - [Roadmap](#roadmap)
  - [Repository structure](#repository-structure)
  - [Requirements](#requirements)
  - [Naming conventions](#naming-conventions)
  - [Scope \& ethics](#scope--ethics)

## Who is this for

- You are comfortable in a terminal (`cd`, `ls`, running a command), but have never read
  assembly or opened a debugger.
- You know a bit of C, or are willing to learn just enough of it along the way.
- You want to understand *why* a program behaves the way it does — not just run tools.

## How to use these notes

1. **Start at PART 0.** Set the lab up first (compiler, debugger, binutils) so every later
   command actually works.
2. **Follow the parts in order.** Each one assumes the previous part is done.
3. **Type the commands yourself.** Change a value, recompile, look at what changed — that
   is where the learning happens.
4. **Keep both languages in sync.** `note.md` (English) and `note.id.md` (Indonesian) are
   the same material, not different content.

## Roadmap

Full roadmap: **[INDEX.md](INDEX.md)** — 10 phases (PART 0 → PART 9).
Currently written: PART 0 → PART 6 (Indonesian drafts); English notes are catching up.

| Part | Topic | Video | Notes |
| :--- | :--- | :---: | :--- |
| **0** | **Foundations** — the RE mindset + first hands-on win | [watch](https://www.youtube.com/watch?v=zDJtr9r2RFs&t=197s) | [EN](PART0/note.md) · [ID](PART0/note.id.md) |
| **1** | **Architecture Fundamentals** — CPU, registers, memory, the stack | [watch](https://www.youtube.com/watch?v=eEcXSvd8PCM) | [EN](PART1/note.md) · [ID](PART1/note.id.md) |
| **2** | **Assembly Language** — read x86-64 disassembly fluently | [watch](https://www.youtube.com/watch?v=HkMetTR67XU) | [EN](PART2/note.md) · [ID](PART2/note.id.md) |
| **3** | **Executable Formats** — anatomy of ELF (Linux) & PE (Windows) | [watch](https://www.youtube.com/watch?v=NefasNimlIk) | [EN](PART3/note.md) · [ID](PART3/note.id.md) |
| **4** | **Static Analysis** — map the whole binary without running it (Ghidra) | [watch](https://www.youtube.com/watch?v=sbGsMc9_qvg) | [EN](PART4/note.md) · [ID](PART4/note.id.md) |
| **5** | **Dynamic Analysis** — run it and watch it live (gdb) | [watch](https://www.youtube.com/watch?v=iuYRHUH51yA) | [EN](PART5/note.md) · [ID](PART5/note.id.md) |
| ↳ 5.1 | **Lab: Buffer Overflow** — watch the stack break, then ret2win | [watch](https://www.youtube.com/watch?v=UVdciB72aEk) | [EN](PART5/BufferOverflow/note.md) · [ID](PART5/BufferOverflow/note.id.md) |
| **6** | **Reversing Patterns** — spot compiler-generated shapes (`if`, loops, `switch`) | [watch](https://www.youtube.com/watch?v=fgWMBp4TVdU) | [EN](PART6/note.md) · [ID](PART6/note.id.md) |
| 7–9 | CrackMes & Challenges · Anti-reversing & Packing · Specialization — *planned* | – | – |

## Repository structure

```
.
├── README.md            # this file (English)
├── README.id.md         # Indonesian version
├── INDEX.md             # roadmap, 10 phases (English)
├── INDEX.id.md          # roadmap (Indonesian)
├── PART0/               # Foundations
│   ├── note.md
│   └── note.id.md
├── PART1/
│   └── note.id.md
├── PART2/
│   └── note.id.md
├── PART3/
│   └── note.id.md
├── PART4/
│   └── note.id.md
├── PART5/
│   ├── note.id.md
│   └── BufferOverflow/  # focused lab inside PART 5
│       └── note.id.md
└── PART6/
    └── note.id.md
```

## Requirements

A Linux machine (x86-64) with:

| Tool | What it is for |
| :--- | :--- |
| `gcc` | Compiles the C practice targets |
| `gdb` | Debugger — pauses a running program and lets you inspect it |
| `binutils` | `objdump`, `readelf`, `strings`, `nm` — binary inspection tools |
| `ltrace` | Shows library calls a program makes (e.g. `strcmp`, `printf`) |
| `strace` | Shows requests a program makes to the kernel (open, read, write) |
| `xxd` | Hex viewer — shows the raw bytes of a file |

From Phase 4 you also need **Ghidra** + a Java JDK — install steps in
[PART 4](PART4/note.md#3-installing-ghidra).

Install (pick your package manager):

```bash
# Arch / pacman
sudo pacman -S base-devel gdb binutils ltrace strace xxd

# Debian / Ubuntu
sudo apt install build-essential gdb binutils ltrace strace xxd

# Fedora
sudo dnf install gcc gdb binutils ltrace strace vim-common   # xxd ships in vim-common
```

Step-by-step lab setup, including verification, is in
**[PART 0 → Lab setup](PART0/note.md#7-lab-setup)**.

## Naming conventions

- `PART<N>/` — one phase of the roadmap.
- `note.md` — English notes. `note.id.md` — Indonesian notes (`*.id.md` = Indonesian).
- **Every part must have both files**, with identical structure and headings.
- A deep-dive lab lives in its own subfolder and follows the same rule,
  e.g. `PART5/BufferOverflow/note.md` + `note.id.md`.

## Scope & ethics

- Everything here is for **learning**: binaries you compiled yourself, crackmes, or targets
  you have explicit permission to analyze.
- No notes here target software you do not own or are not authorized to test.
- Commands can crash a program on purpose — run them in a VM or a disposable environment.

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>