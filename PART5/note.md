# PART 5 — Dynamic Analysis

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 5 of 10**

> [!NOTE]
> **Source:** written to follow the [roadmap](../INDEX.md). The video labelled "PART 5"
> ([iuYRHUH51yA](https://www.youtube.com/watch?v=iuYRHUH51yA)) covers **packing** — that material
> lives in [PART 8](../PART8/note.md).
> Glossary: [section 9](#9-glossary).

**Prerequisites:** [PART 1](../PART1/note.md) (gdb basics) and [PART 4](../PART4/note.md)
(Ghidra, XREFs, image base).

**After this part you can:** translate a Ghidra address into a runtime address, use watchpoints
to catch memory changes, read values static analysis cannot see, and triage quickly with
`ltrace`/`strace`.

## Contents

- [PART 5 — Dynamic Analysis](#part-5--dynamic-analysis)
  - [Contents](#contents)
  - [1. Why dynamic?](#1-why-dynamic)
  - [2. The dynamic toolkit and who does what](#2-the-dynamic-toolkit-and-who-does-what)
  - [3. Bridging addresses: Ghidra to gdb](#3-bridging-addresses-ghidra-to-gdb)
  - [4. gdb commands you have not used yet](#4-gdb-commands-you-have-not-used-yet)
  - [5. Hands-on](#5-hands-on)
    - [5.1 Lab 1 — Turn a Ghidra address into a breakpoint](#51-lab-1--turn-a-ghidra-address-into-a-breakpoint)
    - [5.2 Lab 2 — Read values that are not in the file](#52-lab-2--read-values-that-are-not-in-the-file)
    - [5.3 Lab 3 — Watchpoint: who overwrites the return address?](#53-lab-3--watchpoint-who-overwrites-the-return-address)
  - [6. Limits of dynamic analysis](#6-limits-of-dynamic-analysis)
  - [7. Recap](#7-recap)
    - [Self-check](#self-check)
  - [8. Cliffhanger: from values to patterns](#8-cliffhanger-from-values-to-patterns)
  - [9. Glossary](#9-glossary)

---

## 1. Why dynamic?

PART 4 ended with a question Ghidra cannot answer: **what is in this variable while the program
runs?** Static analysis sees code, not execution.

| Question | Static | Dynamic |
| :--- | :---: | :---: |
| Which functions exist? | ✅ | ⚠️ only those called |
| Who calls whom? | ✅ | ✅ |
| What is in this register **now**? | ❌ | ✅ |
| What did runtime decryption produce? | ❌ | ✅ |
| Which branch actually executes? | ❌ | ✅ |
| Which files/network does it touch? | ⚠️ inferred | ✅ |

> [!TIP]
> PART 0's analogy again: static = reading the floor plan; dynamic = walking inside and watching
> the occupants. Ghidra gives you a much better plan — still a plan.

## 2. The dynamic toolkit and who does what

| Tool | Shows | When to use |
| :--- | :--- | :--- |
| `ltrace` | **library** calls (`strcmp`, `malloc`) | 10-second triage: any string comparison? |
| `strace` | **system calls** (open, read, write, connect) | triage: which files, which connections? |
| `gdb` | registers, memory, instruction flow | precise questions at one spot |
| `/proc/<pid>/maps` | a process's memory regions | finding the load base, spotting libraries |

Quick triage before opening gdb:

```bash
ltrace ./vault 2>&1 | grep -E 'strcmp|memcmp|strncmp'
strace -f ./vault 2>&1 | grep -E 'openat|read|write' | head -10
```

> [!WARNING]
> `ltrace`/`strace` **execute** the program. For untrusted samples, work in an isolated VM with
> no network (PART 0). For analysis without execution use `readelf -d` (PART 3) or Ghidra (PART 4).

## 3. Bridging addresses: Ghidra to gdb

This is the most-used daily skill: you find something in Ghidra, then want to inspect it live.
**The numbers differ.**

| Source | Example address | Why |
| :--- | :--- | :--- |
| `objdump` (PART 2) | `0x11b6` | relative offset inside the binary |
| Ghidra (PART 4) | `0x001011b6` | default image base `0x00100000` + offset |
| `gdb` at runtime (PIE) | `0x5555555551b6` | random ASLR base + offset |
| `gdb` at runtime (`-no-pie`) | `0x4011b6` | fixed base, no randomization |

The formula:

```text
runtime address = load base + (Ghidra address - 0x00100000)
```

> [!NOTE]
> gdb disables ASLR by default (`set disable-randomization on`), so the PIE base often looks like
> `0x555555554000`. Outside gdb it changes on every run.

## 4. gdb commands you have not used yet

| Command | Purpose |
| :--- | :--- |
| `watch *(long*)($rbp+8)` | **watchpoint**: stop when that memory value **changes** |
| `rwatch` / `awatch` | stop on read / on read or write |
| `break *0x... if $rdi == 0` | conditional breakpoint |
| `catch syscall openat` | stop when a given system call is made |
| `info proc mappings` | memory regions plus load base |
| `starti` | stop at the very first instruction |
| `set var $eax = 0` | change a value at runtime (PART 1) |
| `display/i $rip` | print the next instruction at every stop |
| `find $rsp, $rsp+0x200, "sup3r"` | search memory for a pattern |
| `dump binary memory out.bin A B` | write a memory range to a file (used in PART 8) |

> [!TIP]
> A watchpoint answers "who changed this value?" — far more efficient than guessing and setting
> dozens of breakpoints.

## 5. Hands-on

### 5.1 Lab 1 — Turn a Ghidra address into a breakpoint

**Goal:** break at an address you found statically.

From PART 4: `vault`'s decision is at Ghidra `0x001011b6` → offset `0x11b6`.

```bash
gcc -g -O0 vault.c -o vault
gdb -q ./vault
```

```gdb
set disassembly-flavor intel
starti
info proc mappings
```

```text
      Start Addr       End Addr   Size  Offset  Perms  objfile
    0x555555554000   0x555555555000  0x1000    0x0  r--    /path/vault
    0x555555555000   0x555555556000  0x1000 0x1000  r-x    /path/vault
```

Base = `0x555555554000`. Compute: `0x555555554000 + 0x11b6 = 0x5555555551b6`.

```gdb
break *0x5555555551b6
continue
x/i $rip
```

```text
Breakpoint 1, 0x00005555555551b6 in main ()
0x5555555551b6 <main+81>:  jne    0x5555555551c6 <main+97>
```

> [!IMPORTANT]
> **THE AHA:** Ghidra and gdb point at the **same instruction** with different numbers. Once you
> can compute the base, static findings become usable at runtime.

### 5.2 Lab 2 — Read values that are not in the file

**Goal:** see data that only exists while the program runs.

```gdb
break strcmp
continue
x/s $rdi
x/s $rsi
info registers eax
finish
```

```text
0x7fffffffe0a0: "nope"
0x402010:       "sup3rs3cr3t"
```

If the secret were filled by runtime decryption, the file would only hold random-looking bytes —
yet here it must still appear as plaintext (the PART 0 Exercise 2 principle).

> [!IMPORTANT]
> **THE AHA:** Static analysis gives you the **map**; dynamic analysis gives you the **values**.
> They are not alternatives but partners: static to find the place, dynamic to read it.

### 5.3 Lab 3 — Watchpoint: who overwrites the return address?

**Goal:** catch a memory overwrite the instant it happens. Use `vuln.c` from the
[Buffer Overflow lab](BufferOverflow/note.md).

```bash
gcc -g -O0 -fno-stack-protector -no-pie vuln.c -o vuln
gdb -q ./vuln
```

```gdb
set disassembly-flavor intel
break *main+0x2a
run
watch *(long*)($rbp+8)
continue
```

```text
Hardware watchpoint 2: *(long*)($rbp+8)
Old value = 0x7ffff7e0a2c0
New value = 0x4141414141414141
```

gdb stops **on the exact instruction** writing to the return-address slot and shows the value
before and after.

> [!IMPORTANT]
> **THE AHA:** A watchpoint turns "where is the bug?" into "show me the moment this data
> changes". This is the direct bridge into the
> [Buffer Overflow lab](BufferOverflow/note.md).

## 6. Limits of dynamic analysis

| Limit | Explanation |
| :--- | :--- |
| Only executed paths | uncalled code stays invisible |
| Needs the right input | hidden branches need specific input/conditions |
| Can be detected | anti-debug spots the debugger and lies (Phase 8) |
| Risky | the code really runs → isolated VM required |
| Not self-sufficient | without a static map you do not know where to stop |

> [!NOTE]
> That is why real workflows loop: Ghidra → gdb → Ghidra. Each round adds names, comments, and
> understanding.

## 7. Recap

| Concept | What to remember |
| :--- | :--- |
| Static vs dynamic | map vs values; used as a pair |
| Triage | `ltrace` (library), `strace` (syscalls) before gdb |
| Addresses | runtime = base + (Ghidra address − `0x00100000`) |
| Base | from `info proc mappings`; gdb disables ASLR by default |
| Watchpoint | `watch *(long*)(addr)` → stop when the value changes |
| Conditional break | `break *addr if condition` |
| `catch syscall` | stop on a specific system call |
| Limits | executed paths only, detectable, risky |

### Self-check

- [ ] What information can only be obtained at runtime?
- [ ] How do you compute a runtime address from a Ghidra address?
- [ ] Why are addresses stable in gdb but not outside it?
- [ ] When do you use a watchpoint instead of a breakpoint?
- [ ] How do `ltrace` and `strace` differ, and when do you use each?
- [ ] Why does dynamic analysis still need static analysis?

## 8. Cliffhanger: from values to patterns

You can now find the place (static) and read its value (dynamic). What is still expensive is
**volume**: reading 200 functions one by one remains exhausting.

The way out is not another tool but **pattern recognition**: compilers emit repeating shapes for
`if`, loops, `switch`, and function calls. Once you recognize them you skip the boilerplate and
go straight to the core.

> [!IMPORTANT]
> That is **Phase 6 — Reversing Patterns**.

## 9. Glossary

| Term | Plain meaning |
| :--- | :--- |
| Dynamic analysis | analyzing a program while it runs |
| Runtime value | a real value that only exists during execution |
| `ltrace` | shows library function calls |
| `strace` | shows system calls to the kernel |
| System call | a program's request to the kernel (open, read, write) |
| `/proc/<pid>/maps` | a process's memory region list |
| Load base | base address where the binary is loaded |
| Image base | the base Ghidra uses (default `0x00100000`) |
| ASLR | OS address randomization |
| `disable-randomization` | the gdb option that turns ASLR off while debugging |
| Breakpoint | a stop point at an address/instruction |
| Conditional breakpoint | a breakpoint active only when a condition holds |
| Watchpoint | a stop point triggered when a memory value changes |
| `rwatch` / `awatch` | watchpoint on reads / on both |
| `catch syscall` | stop when a given system call is made |
| `starti` | stop at the program's first instruction |
| `info proc mappings` | view memory regions and bases in gdb |
| `find` | search memory for a byte/string pattern |
| `dump binary memory` | write a memory range to a file |
| Code path | a path of code actually executed |
| Triage | a quick pass to decide where to analyze |
| Anti-debug | a program detecting the debugger (Phase 8) |

---

[← PART 4](../PART4/note.md) · [Roadmap](../INDEX.md) · **Lab:**
[Buffer Overflow](BufferOverflow/note.md) · **Next:** PART 6 — Reversing Patterns
([ID notes](../PART6/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>