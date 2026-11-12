# The Roadmap — 10 Phases

**English** · [Bahasa Indonesia](INDEX.id.md)

The path from *"never opened a debugger"* to *"can take apart a protected program"*.
Each phase is one folder (`PART<N>/`), and every folder holds the same material in two
languages: `note.md` (English) and `note.id.md` (Indonesian).

New here? Read the [README](README.md) first, then start at Phase 0.

## How to read this map

- **Order matters.** Every phase assumes the previous one is done. Phase 0 is not optional —
  it sets up the lab all later commands depend on.
- **"What you will learn"** is written in plain language; technical terms stay, but they are
  explained on first use.
- **Notes** links point to the material itself. `EN soon` = Indonesian draft exists, English
  version is being written. `–` = folder not created yet.

**Status legend:** ✅ Ready (both languages, tidy) · 🚧 Draft (notes exist, being tidied /
translated) · 🗓 Planned (not started)

## Stages at a glance

| Stage | Phases | After this stage you can… |
| :--- | :---: | :--- |
| **Foundations** | 0 | Set up a lab and reverse your first small program end to end |
| **Machine & language** | 1–3 | Read assembly and know where every piece of a program lives |
| **Analysis** | 4–6 | Map a binary statically and dynamically, and recognize the original code shapes |
| **Applied RE** | 7–9 | Beat protected crackmes and pick a specialization |

## Phases

| Phase | Topic | What you will learn | Notes | Status |
| :---: | :--- | :--- | :--- | :---: |
| **0** | **Foundations** | The reverse engineering mindset + your first hands-on win | [EN](PART0/note.md) · [ID](PART0/note.id.md) | ✅ Ready |
| **1** | **Architecture Fundamentals** | CPU, registers (tiny ultra-fast storage inside the CPU), memory, and the stack (temporary memory used when functions call each other) | [EN](PART1/note.md) · [ID](PART1/note.id.md) | ✅ Ready |
| **2** | **Assembly Language** | Read x86-64 disassembly (the human-readable text of a binary's instructions) fluently | [EN](PART2/note.md) · [ID](PART2/note.id.md) | ✅ Ready |
| **3** | **Executable Formats** | Anatomy of an executable file: ELF (Linux) and PE (Windows) | [EN](PART3/note.md) · [ID](PART3/note.id.md) | ✅ Ready |
| **4** | **Static Analysis** | Ghidra: map the whole binary without ever running it | [EN](PART4/note.md) · [ID](PART4/note.id.md) | ✅ Ready |
| **5** | **Dynamic Analysis** | gdb: run the program and watch it live, one step at a time | [EN](PART5/note.md) · [ID](PART5/note.id.md) | ✅ Ready |
| ↳ 5.1 | **Lab: Buffer Overflow** | Watch the stack break in the debugger, then redirect execution (ret2win) | [EN](PART5/BufferOverflow/note.md) · [ID](PART5/BufferOverflow/note.id.md) | ✅ Ready |
| **6** | **Reversing Patterns** | Spot compiler-generated shapes (`if`, loops, `switch`) inside assembly | [EN](PART6/note.md) · [ID](PART6/note.id.md) | ✅ Ready |
| **7** | **CrackMes & Challenges** | Build a keygen (license-key generator) and patch real targets | – | 🗓 Planned |
| **8** | **Anti-reversing & Packing** | Unpack compressed/scrambled programs and defeat anti-debug protection | – | 🗓 Planned |
| **9** | **Specialization** | Pick a lane: malware, firmware, mobile, or exploitation | – | 🗓 Planned |

## Folder rules

- One phase = one folder: `PART0/` … `PART9/`.
- Every phase folder must contain **both** `note.md` and `note.id.md`, with matching headings.
- A deep-dive lab gets its own subfolder and follows the same rule,
  e.g. `PART5/BufferOverflow/`.
- Full conventions and the current tree: [README → Naming conventions](README.md#naming-conventions).

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>