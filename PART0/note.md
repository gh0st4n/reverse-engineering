# PART 0 — Foundations

**English** · [Bahasa Indonesia](note.id.md)

> [!NOTE]
> Source: [PART 0 on YouTube](https://www.youtube.com/watch?v=zDJtr9r2RFs&t=197s)

## Table of Contents

1. [What is Reverse Engineering?](#1-what-is-reverse-engineering)
2. [Why learn this?](#2-why-learn-this)
3. [The core idea: the compiler throws information away](#3-the-core-idea-the-compiler-throws-information-away)
4. [Source to binary: the 4-stage pipeline](#4-source-to-binary-the-4-stage-pipeline)
5. [Two approaches: static vs dynamic](#5-two-approaches-static-vs-dynamic)
6. [When to use which](#6-when-to-use-which)
7. [Lab setup](#7-lab-setup)
8. [Starter toolkit](#8-starter-toolkit)
9. [Practice: breaking `vault`](#9-practice-breaking-vault)
10. [Recap](#10-recap)
11. [Cliffhanger: the 4th way](#11-cliffhanger-the-4th-way)
12. [Quick glossary](#12-quick-glossary)

---

## 1. What is Reverse Engineering?

You have a program but **no source code**. Your job is to rebuild its logic from the compiled binary.

> [!TIP]
> **Think like a detective.** The compiler is a paper shredder: it throws away names (variables/functions), comments, and structure. But **the logic survives**, and that is what you piece back together.

---

## 2. Why learn this?

| Field | What you do |
|---|---|
| **Malware Analysis** | Reverse malicious binaries to see exactly what they do |
| **Firmware & IoT** | Tear apart embedded code: routers, cars, devices |
| **CTF & Crackmes** | Solve reversing challenges and capture the flag |
| **Vulnerability Research** | Reverse software to find exploitable bugs |

---

## 3. The core idea: the compiler throws information away

```
 SOURCE CODE (C)                          BINARY → DISASSEMBLY
┌─────────────────────────────┐          ┌──────────────────────────┐
│ // add two numbers          │          │ sub_401136:              │
│ int add(int a, int b) {     │   ───▶   │     lea eax, [rdi+rsi]   │
│     return a + b;           │          │     ret                  │
│ }                           │          │                          │
└─────────────────────────────┘          └──────────────────────────┘
```

| Lost | Preserved |
|---|---|
| Names (`add`, `a`, `b`) → auto-generated names like `sub_401136` | **Logic**: two values are added and returned |
| Comments | |
| Original structure | |

> [!IMPORTANT]
> Turning a binary back into understandable code is the job.

---

## 4. Source to binary: the 4-stage pipeline

```
 vault.c ──▶ Preprocess ──▶ Compile ──▶ Assemble ──▶ Link ──▶ vault
 (source)      (.i)          (.s)         (.o)       (ELF/.exe) (binary)
```

| # | Stage | What happens | Output |
|---|---|---|---|
| 1 | **Preprocess** | Expands `#include` and `#define` | `.i` (still C) |
| 2 | **Compile** | Turns **C** into **assembly** | `.s` |
| 3 | **Assemble** | Turns assembly into **machine code** | `.o` (object file) |
| 4 | **Link** | Combines the pieces (`.o` + libraries) into the final executable | executable |

> [!NOTE]
> Every RE tool you use only shows the output of **one** of these stages.

### Each stage with `gcc`

```shell
$ gcc -E test.c -o test.i   # 1. preprocess → expanded C
$ gcc -S test.c -o test.s   # 2. compile    → assembly
$ gcc -c test.c -o test.o   # 3. assemble   → machine code (not yet runnable)
$ gcc test.c -o test        # 4. link       → executable
```

Each flag stops the pipeline **one stage earlier**. The `.s` file is where C becomes human-readable machine logic.

---

## 5. Two approaches: static vs dynamic

| | **Static Analysis** | **Dynamic Analysis** |
|---|---|---|
| **Concept** | Read/analyze the binary **without running it** | **Run** the program and observe its behavior live |
| **Pros** | See the whole code structure at once. Safe: no malicious code runs | See register and memory values in real time. Observe real behavior when interacting with the system |
| **Cons** | No runtime values | Only sees code paths that actually execute; hidden or uncalled code stays invisible |
| **Tools** | `strings`, `objdump`, `ghidra` | `gdb`, `ltrace`, `strace` |

---

## 6. When to use which

- **Static first → build the MAP.** What functions exist, and how do they connect?
- **Dynamic to zoom.** Clear up confusion at ONE specific point: what is actually in this register right now?

> [!TIP]
> **Rule of thumb:** static to map the whole, dynamic to zoom in on one spot. A good reverser switches back and forth constantly.

---

## 7. Lab setup

### Virtual Machine

**Why a VM?** When analyzing **real malware**, it only runs inside the VM, so your host system stays safe. Good habits for malware labs (later, not needed for Part 0): take a **snapshot** before running a sample, disable shared folders/clipboard, and use host-only or no networking.

> [!NOTE]
> The Part 0 exercises use our own harmless `vault.c`, so you can do them directly on your machine. Still, build the discipline early.

### Install the tools

Ghidra: download it separately from its [official repo](https://github.com/NationalSecurityAgency/ghidra) (requires Java/JDK) or via your package manager if available. You don't need it for Part 0.

<details>
<summary><b>APT</b> (Debian / Ubuntu / Kali Linux / Parrot)</summary>

```shell
sudo apt update
sudo apt install gcc gdb binutils file xxd ltrace strace
```
</details>

<details>
<summary><b>DNF</b> (Fedora / RHEL / CentOS)</summary>

```shell
sudo dnf install gcc gdb binutils file xxd ltrace strace
```
</details>

<details>
<summary><b>Pacman</b> (Arch Linux)</summary>

```shell
sudo pacman -Syu
sudo pacman -S gcc gdb binutils file xxd ltrace strace
```
</details>

<details>
<summary><b>XBPS</b> (Void Linux)</summary>

```shell
sudo xbps-install -S
sudo xbps-install gcc gdb binutils file xxd ltrace strace
```
</details>

> [!NOTE]
> **binutils** alone already provides `objdump`, `readelf`, `nm`, and `strings`: the main tools for Phases 3 and 4.

---

## 8. Starter toolkit

| Tool | Purpose | Type |
|---|---|---|
| `strings` | Pull human-readable text straight out of binary data | Static |
| `objdump` | **Disassembler**: turns machine code back into assembly | Static |
| `ltrace` | See **library calls** (e.g. `strcmp`, `printf`) while the program runs | Dynamic |
| `strace` | See **system calls** (requests to the kernel) while the program runs | Dynamic |
| `gdb` | **Debugger**: pause, single-step, inspect memory live | Dynamic |
| `ghidra` | Full, free **disassembler + decompiler** (our main IDE) | Static |

---

## 9. Practice: breaking `vault`

### Target: `vault.c`

```c
#include <stdio.h>
#include <string.h>

int main() {
    char input[64];
    const char *secret = "sup3rs3cr3t";
    printf("Enter the vault password: ");
    scanf("%63s", input);
    if (strcmp(input, secret) == 0) {
        printf("ACCESS GRANTED\n");
    } else {
        printf("ACCESS DENIED\n");
    }
    return 0;
}
```

**Looks safe. It isn't.** We'll break it three ways, **no assembly needed**:

1. **Read** the secret
2. **Sniff** it live
3. **Patch** the data

> [!WARNING]
> Exercises 3, 4 and 5 **modify the `vault` file**. To start clean again, recompile: `gcc vault.c -o vault` (or keep a backup: `cp vault vault.bak`). Only reverse programs you own or are explicitly allowed to analyze.

### Exercise 1: Read the secret

```shell
$ gcc vault.c -o vault
$ strings vault | grep sup
sup3rs3cr3t
```

> [!IMPORTANT]
> **THE AHA:** A secret compiled into a binary is **not a secret**. This is exactly why hardcoded credentials are a real finding in a pentest.

### Exercise 2: Sniff it live

```shell
$ ltrace ./vault
printf("Enter the vault password: ")              = 26
__isoc99_scanf(0x55bf3a66402b, 0x7ffe25e3a060, 0, 0) = 1
strcmp("testsalah", "sup3rs3cr3t")                 = 1
puts("ACCESS DENIED")                              = 14
+++ exited (status 0) +++
```

> [!NOTE]
> Real `ltrace` output interleaves with the program's own prints; it is cleaned up here. Addresses like `0x55bf3a66402b` change on every run (ASLR). The compiler also turned `printf("...\n")` into `puts`: the binary is not a mirror of your source (see Exercise 7).

`strcmp(...) = 1` means "not equal" (`0` would mean equal).

> [!IMPORTANT]
> **THE AHA:** Even if the secret is scrambled inside the file, it **must appear as plaintext in memory** at the moment of comparison. Dynamic analysis catches it.

### Exercise 3: Edit a binary by hand

```shell
# Before
$ ./vault        # → ACCESS DENIED

# Same-length swap (13 chars each)
$ sed -i 's/ACCESS DENIED/H4CK3D BY YOU/' vault

# After
$ ./vault        # → H4CK3D BY YOU
```

> [!IMPORTANT]
> **THE AHA:** A binary is just a file of bytes you can rewrite. **Same length matters**: changing the size shifts everything after it and can break the program.

### Exercise 4: Bypass auth, zero assembly

```shell
$ sed -i 's/sup3rs3cr3t/AAAAAAAAAAA/' vault
$ ./vault
Enter the vault password: AAAAAAAAAAA
ACCESS GRANTED
```

> [!IMPORTANT]
> **THE AHA:** You bypassed the authentication check by editing the **DATA**, not the logic: you turned the right answer into an answer you already know.

### Exercise 5: Break the magic bytes

```shell
$ xxd vault | head -1
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............

# Corrupt the first byte (recompile first: gcc vault.c -o vault)
$ printf '\000' | dd of=vault bs=1 seek=0 count=1 conv=notrunc
$ ./vault
bash: ./vault: cannot execute binary file: Exec format error
```

> [!TIP]
> Use `\000` rather than `\x00`: some shells (e.g. `dash`) don't support `\x` escapes in `printf` and would write a literal `\` instead.

> [!IMPORTANT]
> **THE AHA:** Linux recognizes executables by their **magic bytes** (`7f 45 4c 46` = `.ELF`), not by file extension. A first taste of file formats.

### Exercise 6: Endianness with your own eyes

```shell
$ python3 -c "import struct,sys; sys.stdout.buffer.write(struct.pack('<I', 0x41424344))" | xxd
00000000: 4443 4241                                DCBA
```

> [!IMPORTANT]
> **THE AHA:** x86 stores multi-byte values **least-significant byte first**. This byte order in memory is called **little-endian**.

### Exercise 7: The disappearing function

```c
static int compute(void) { return 3 * 4 * 5; }
int main(void)           { return compute(); }
```

```
 gcc -O0  (no optimization)              gcc -O2  (optimized)
┌─────────────────────────────────┐     ┌──────────────────────────────┐
│ compute:                        │     │ main:                        │
│     push rbp                    │     │     mov  eax, 60             │
│     mov  rbp, rsp               │     │     ret                      │
│     mov  eax, 60                │     │                              │
│     pop  rbp                    │     │ ; compute() is gone:         │
│     ret                         │     │ ; inlined into main          │
│ main:                           │     │                              │
│     push rbp                    │     │                              │
│     mov  rbp, rsp               │     │                              │
│     call compute                │     │                              │
│     pop  rbp                    │     │                              │
│     ret                         │     │                              │
└─────────────────────────────────┘     └──────────────────────────────┘
```

Try it: `gcc -S -O0 -masm=intel c.c -o c_O0.s` and the same with `-O2`. (Capital **O**: lowercase `-o` means "output file". The listing above omits boilerplate lines such as `.cfi` directives.)

Two things happened: **constant folding** (the compiler computed `3*4*5 = 60` at compile time, even at `-O0`) and **inlining** (at `-O2`, `compute()` was merged into `main()` and discarded).

> [!IMPORTANT]
> **Machine code is NOT a mirror of your source.** Recognizing what the compiler did is the core skill for everything that follows.

---

## 10. Recap

| # | Way | Tool | Analysis | Result |
|---|---|---|---|---|
| 1 | **Read** | `strings` | Static | The secret sits in plaintext inside the file |
| 2 | **Sniff** | `ltrace` | Dynamic | Caught live in memory at comparison time |
| 3 | **Patch** | `sed` | Data edit | We rewrote the data so an answer we know is accepted |

---

## 11. Cliffhanger: the 4th way

**The 4th way needs... assembly.**

```asm
    call    strcmp          ; compare input with secret → result in eax (0 = equal)
    test    eax, eax        ; is eax == 0 ?
    jne     .denied         ; if NOT 0 (different) → jump to DENIED   ◀── flip THIS
    ; ... ACCESS GRANTED
.denied:
    ; ... ACCESS DENIED
```

**Patch the decision itself.** Flip `jne` to `je` and the logic inverts (wrong passwords get in); NOP the jump out and **any** password is accepted. But now you must be able to read and edit actual instructions.

> [!IMPORTANT]
> That is exactly what **Phase 1 (architecture)** and **Phase 2 (assembly)** unlock.

---

## 12. Quick glossary

| Term | Meaning |
|---|---|
| **Binary** | Compiled program file containing machine code |
| **Disassembler / Decompiler** | Machine code → assembly / machine code → C-like code |
| **Register** | Tiny, ultra-fast storage inside the CPU (`eax`, `rdi`, ...) |
| **Opcode** | The number that encodes an instruction (`75` = `jne`, `74` = `je`) |
| **ELF** | Linux executable format (Windows equivalent: PE) |
| **Magic bytes** | First bytes of a file identifying its type |
| **Endianness** | Byte order of multi-byte values; x86 is little-endian |
| **Patch** | Modify bytes in a binary to change its behavior |
| **NOP** | "Do nothing" instruction (`90` on x86) |
| **Constant folding / Inlining** | Compiler optimizations: precompute constants / merge functions into callers |
| **ASLR** | OS randomization of memory addresses |

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>
