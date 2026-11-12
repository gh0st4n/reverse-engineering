# PART 0 — Foundations

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 0 of 10**

> [!NOTE]
> **Source:** [PART 0 on YouTube](https://www.youtube.com/watch?v=zDJtr9r2RFs&t=197s)
> Written for beginners. Technical terms stay in English so you get used to real-world
> vocabulary, but every term is explained in plain language first.
> Glossary: [section 12](#12-quick-glossary).

**Prerequisites:** Linux on x86-64 (a VM is fine) and basic terminal use (`cd`, `ls`, running
a command). No assembly knowledge required.

**After this part you can:** explain how source code becomes a binary, choose between static
and dynamic analysis, and defeat a small password check three different ways — without
reading a single line of assembly.

## Table of contents

- [PART 0 — Foundations](#part-0--foundations)
  - [Table of contents](#table-of-contents)
  - [1. What is reverse engineering?](#1-what-is-reverse-engineering)
  - [2. Why learn this?](#2-why-learn-this)
  - [3. The core idea: the compiler throws information away](#3-the-core-idea-the-compiler-throws-information-away)
  - [4. Source to binary: the 4-stage pipeline](#4-source-to-binary-the-4-stage-pipeline)
    - [Running each stage with `gcc`](#running-each-stage-with-gcc)
  - [5. Two approaches: static vs dynamic](#5-two-approaches-static-vs-dynamic)
  - [6. When to use which](#6-when-to-use-which)
  - [7. Lab setup](#7-lab-setup)
    - [7.1 Virtual machine](#71-virtual-machine)
    - [7.2 Install the tools](#72-install-the-tools)
    - [7.3 Verify the install](#73-verify-the-install)
  - [8. Starter toolkit](#8-starter-toolkit)
  - [9. Practice: breaking `vault`](#9-practice-breaking-vault)
    - [Exercise 1 — Read the secret](#exercise-1--read-the-secret)
    - [Exercise 2 — Sniff it live](#exercise-2--sniff-it-live)
    - [Exercise 3 — Edit a binary by hand](#exercise-3--edit-a-binary-by-hand)
    - [Exercise 4 — Bypass auth, zero assembly](#exercise-4--bypass-auth-zero-assembly)
    - [Exercise 5 — Break the magic bytes](#exercise-5--break-the-magic-bytes)
    - [Exercise 6 — Endianness with your own eyes](#exercise-6--endianness-with-your-own-eyes)
    - [Exercise 7 — The disappearing function](#exercise-7--the-disappearing-function)
  - [10. Recap](#10-recap)
    - [Self-check](#self-check)
  - [11. Cliffhanger: the 4th way](#11-cliffhanger-the-4th-way)
  - [12. Quick glossary](#12-quick-glossary)

---

## 1. What is reverse engineering?

**Reverse engineering (RE)** means understanding how a program works *without* having its
source code.

- **Source code** — what the programmer wrote, e.g. a `.c` file. Readable by humans.
- **Binary** — the compiled program (`.exe` on Windows, ELF on Linux). It contains
  **machine code**: strings of numbers the CPU understands, not humans.

You have the binary only. Your job is to rebuild the program's logic from it.

> [!TIP]
> Think like a detective. The **compiler** (the program that turns source code into a binary)
> works like a paper shredder: variable names, function names, comments, and code structure
> are thrown away. The logic survives — and the logic is what you piece back together.

## 2. Why learn this?

| Field | What you do |
| :--- | :--- |
| **Malware analysis** | Take apart malicious programs to see exactly what they do |
| **Firmware & IoT** | Dig into code embedded in hardware: routers, cars, CCTV cameras. *Firmware* = the program baked into a device |
| **CTF & crackmes** | *CTF* (Capture The Flag) is a security competition; a *crackme* is a small program built to be broken as practice. Goal: solve the challenge and capture the *flag* (proof you did it) |
| **Vulnerability research** | Reverse software to find bugs that can be exploited |

## 3. The core idea: the compiler throws information away

Left: the function as the programmer wrote it. Right: what you see when you take the binary
apart — the **disassembly** (machine code translated back into assembly text).

```text
 SOURCE CODE (C)                          BINARY → DISASSEMBLY
┌─────────────────────────────┐          ┌──────────────────────────┐
│ // add two numbers          │          │ sub_401136:              │
│ int add(int a, int b) {     │   ───▶   │     lea eax, [rdi+rsi]   │
│     return a + b;           │          │     ret                  │
│ }                           │          │                          │
└─────────────────────────────┘          └──────────────────────────┘
```

| Lost | Survives |
| :--- | :--- |
| Names (`add`, `a`, `b`) → auto-generated labels like `sub_401136` | The logic: two values are added and the result is returned |
| Comments | The order of operations and the branching |
| Original structure (types, formatting, layout) | Constants and strings used by the program |

Reading that assembly snippet (details in Phases 1 and 2):

- `rdi` and `rsi` are **registers** — tiny, ultra-fast storage inside the CPU. On Linux
  x86-64, a function's first and second arguments are placed there.
- `lea eax, [rdi+rsi]` computes `rdi + rsi` and stores it in `eax`, the register that holds a
  function's return value.
- `ret` = done, go back to whoever called this function.

> [!IMPORTANT]
> Turning a binary back into code a human can understand — that is the job.

## 4. Source to binary: the 4-stage pipeline

```text
 vault.c ──▶ Preprocess ──▶ Compile ──▶ Assemble ──▶ Link ──▶ vault
 (source)      (.i)          (.s)         (.o)      (ELF/.exe) (binary)
```

| # | Stage | What happens | Output |
| :---: | :--- | :--- | :--- |
| 1 | **Preprocess** | Expands `#include` and `#define` (pastes header contents, replaces macros) | `.i` (still C) |
| 2 | **Compile** | Turns C into assembly | `.s` |
| 3 | **Assemble** | Turns assembly into machine code | `.o` (object file) |
| 4 | **Link** | Combines the pieces (`.o` + libraries) into the final executable | executable |

> [!NOTE]
> Every RE tool shows you the output of one of these stages. `objdump` and Ghidra work on the
> final result (stage 4) and try to translate it backwards.

### Running each stage with `gcc`

```bash
gcc -E test.c -o test.i   # 1. preprocess → expanded C
gcc -S test.c -o test.s   # 2. compile    → assembly
gcc -c test.c -o test.o   # 3. assemble   → machine code (not runnable yet)
gcc test.c -o test        # 4. link       → executable
```

Each flag (`-E`, `-S`, `-c`) stops the pipeline one stage earlier. The `.s` file is where C
becomes machine logic that a human can still read.

> [!TIP]
> Try it: write a tiny `test.c`, run `gcc -S test.c`, then open `test.s` in a text editor.

## 5. Two approaches: static vs dynamic

Analogy: **static** = reading the blueprint of a house without entering it.
**Dynamic** = walking inside and watching the people who live there.

| | Static analysis | Dynamic analysis |
| :--- | :--- | :--- |
| **Concept** | Read/analyze the binary **without running it** | **Run** the program and observe its behavior live |
| **Pros** | See the whole code structure at once. Safe: no malicious code executes | See register and memory **values in real time**; see real behavior as it interacts with the system |
| **Cons** | No **runtime values** (the actual values while running) | Only sees **code paths** that actually execute; hidden or uncalled code stays invisible |
| **Tools** | `strings`, `objdump`, `ghidra` | `gdb`, `ltrace`, `strace` |

## 6. When to use which

- **Static first → build the MAP.** Which functions exist, and how do they connect?
- **Dynamic to zoom.** Clear up confusion at ONE specific point: *"what is actually inside
  this register right now?"*

> [!TIP]
> Rule of thumb: static to map the whole, dynamic to zoom in on one spot. A good reverser
> switches back and forth constantly.

## 7. Lab setup

### 7.1 Virtual machine

**Why a VM?** A VM is a computer inside your computer. When you analyze real malware, it runs
only inside the VM, so your main system stays safe.

Good habits for a malware lab (for later — not needed for Part 0):

- Take a **snapshot** (a restore point) before running a sample.
- Disable shared folders and clipboard sharing between VM and host.
- Use host-only networking, or no network at all.

> [!NOTE]
> Part 0 exercises use our own harmless `vault.c`, so you can run them directly on your
> machine. Still, build the discipline early.

### 7.2 Install the tools

<details>
<summary><b>APT</b> — Debian / Ubuntu / Kali Linux / Parrot</summary>

```bash
sudo apt update
sudo apt install gcc gdb binutils file xxd ltrace strace python3
```

</details>

<details>
<summary><b>DNF</b> — Fedora / RHEL / CentOS</summary>

```bash
sudo dnf install gcc gdb binutils file ltrace strace python3 vim-common
# xxd ships inside vim-common on Fedora
```

</details>

<details>
<summary><b>Pacman</b> — Arch Linux</summary>

```bash
sudo pacman -Syu
sudo pacman -S gcc gdb binutils file xxd ltrace strace python
```

</details>

<details>
<summary><b>XBPS</b> — Void Linux</summary>

```bash
sudo xbps-install -S
sudo xbps-install gcc gdb binutils file xxd ltrace strace python3
```

</details>

> [!NOTE]
> `binutils` alone already provides `objdump`, `readelf`, `nm`, and `strings` — the main tools
> for Phases 3 and 4. `python3` is only needed for Exercise 6 and is usually preinstalled.
> **Ghidra** is not in every package manager: grab it from the
> [official repo](https://github.com/NationalSecurityAgency/ghidra) (needs Java/JDK).
> You do not need it for Part 0.

### 7.3 Verify the install

```bash
gcc --version | head -1
gdb --version | head -1
objdump --version | head -1
command -v strings xxd file ltrace strace python3
```

Every command should print a version or a path. If something is missing, install it before
moving on.

## 8. Starter toolkit

| Tool | Purpose | Type |
| :--- | :--- | :---: |
| `strings` | Pull human-readable text straight out of binary data | Static |
| `objdump` | **Disassembler**: turns machine code back into assembly | Static |
| `ltrace` | Show **library calls** (calls to ready-made functions, e.g. `strcmp`, `printf`) while the program runs | Dynamic |
| `strace` | Show **system calls** (requests to the kernel, e.g. open/read a file) while the program runs | Dynamic |
| `gdb` | **Debugger**: pause a program, single-step it, inspect memory live | Dynamic |
| `ghidra` | Full, free disassembler + **decompiler** (guesses C-like code from assembly). Our main IDE | Static |

Helpers used in the exercises: `file` (identify a file by its contents), `xxd` (hex viewer),
`sed` and `dd` (rewrite bytes), `python3` (generate test bytes).

## 9. Practice: breaking `vault`

**Target:** `vault.c`

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

The program asks for a password, compares it with `secret` using `strcmp` (a string
comparison function: it returns `0` when the strings are equal), then prints the verdict.

Looks safe. It isn't. We will break it **three ways** — read, sniff, patch (Exercises 1–4) —
then do **three bonus labs**: magic bytes, endianness, compiler behavior (Exercises 5–7).
No assembly needed.

> [!WARNING]
> - Exercises 3, 4 and 5 **modify the `vault` file**. To start clean, recompile:
>   `gcc vault.c -o vault`, or keep a backup: `cp vault vault.bak`.
> - Only reverse programs you own or are explicitly allowed to analyze (your own code,
>   crackmes, targets with permission).

> [!NOTE]
> Outputs below were verified on x86-64 Linux. Yours may differ slightly: addresses change on
> every run (ASLR/PIE), and newer `gcc` inserts extra instructions such as `endbr64`.

Run it once before breaking it:

```bash
gcc vault.c -o vault
./vault
```

```text
Enter the vault password: tebak
ACCESS DENIED
```

### Exercise 1 — Read the secret

**Goal:** find the password without running the program (static).

```bash
gcc vault.c -o vault
strings vault | grep sup
```

```text
sup3rs3cr3t
```

`strings` prints every run of readable characters inside the binary; `grep sup` keeps only
the lines containing "sup".

> [!IMPORTANT]
> **THE AHA:** A secret compiled into a binary is not a secret. This is exactly why
> **hardcoded credentials** are a real finding in a pentest.

### Exercise 2 — Sniff it live

**Goal:** catch the password while the program runs (dynamic).

```bash
ltrace ./vault
```

Type any wrong password, e.g. `testsalah`. Trimmed output:

```text
printf("Enter the vault password: ")                 = 26
__isoc99_scanf(0x55bf3a66402b, 0x7ffe25e3a060, 0, 0)  = 1
strcmp("testsalah", "sup3rs3cr3t")                    = 1
puts("ACCESS DENIED")                                = 14
+++ exited (status 0) +++
```

How to read it:

- `strcmp("testsalah", "sup3rs3cr3t")` — your input and the secret appear **side by side as
  plaintext**. A return of `0` means equal; any non-zero value means different. (The exact
  number is the byte difference, so do not rely on it being `1`.)
- Why `puts` when the source says `printf`? The compiler is smart: `printf("...\n")` with no
  format arguments becomes the cheaper `puts`. First hint that a binary is not a mirror of
  your source (more in Exercise 7).

> [!NOTE]
> Real `ltrace` output interleaves with the program's own prints; it is cleaned up above.
> Addresses like `0x55bf3a66402b` change every run because of **ASLR** (the OS randomizes
> memory addresses).

> [!IMPORTANT]
> **THE AHA:** Even if the secret is scrambled inside the file, it must exist as plaintext in
> memory at the moment of comparison. Dynamic analysis catches it.

### Exercise 3 — Edit a binary by hand

**Goal:** change what the program prints, without recompiling.

```bash
./vault                                        # before: ACCESS DENIED
sed -i 's/ACCESS DENIED/H4CK3D BY YOU/' vault
./vault                                        # after:  H4CK3D BY YOU
```

`sed -i 's/old/new/' file` finds `old` and replaces it with `new` directly inside the file
(`-i` = in-place). Because that text sits in the binary as-is, `sed` can edit it.

Count it yourself: `ACCESS DENIED` = 13 characters, `H4CK3D BY YOU` = 13 characters.
**The lengths must match.**

> [!TIP]
> Byte-exact alternative (safer on binaries — it touches nothing else):
>
> ```bash
> gcc vault.c -o vault                       # start clean
> off=$(grep -abo 'ACCESS DENIED' vault | head -1 | cut -d: -f1)
> printf 'H4CK3D BY YOU' | dd of=vault bs=1 seek="$off" conv=notrunc
> ```
>
> `grep -abo` reports the **byte offset** of the text; `dd` then writes exactly 13 bytes there.

> [!IMPORTANT]
> **THE AHA:** A binary is just a file of bytes you can rewrite. Same length matters:
> changing the size shifts every byte after it (addresses, offsets) and can break the program.

### Exercise 4 — Bypass auth, zero assembly

**Goal:** get in with a password you choose yourself.

```bash
gcc vault.c -o vault
sed -i 's/sup3rs3cr3t/AAAAAAAAAAA/' vault
./vault
```

```text
Enter the vault password: AAAAAAAAAAA
ACCESS GRANTED
```

`sup3rs3cr3t` and `AAAAAAAAAAA` are both 11 characters. We replaced the answer key inside the
binary with an answer we already know.

> [!IMPORTANT]
> **THE AHA:** You bypassed the authentication check by editing the **DATA**, not the logic.
> You turned the correct answer into an answer you know.

### Exercise 5 — Break the magic bytes

**Goal:** see how Linux decides that a file is a program.

```bash
xxd vault | head -1
file vault
```

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
vault: ELF 64-bit LSB pie executable, x86-64, ...
```

`xxd` shows the file in **hexadecimal** (base 16; one byte = two digits). The right-hand
column shows the text version: the first four bytes `7f 45 4c 46` read as `.ELF`.

Now corrupt the first byte:

```bash
gcc vault.c -o vault                       # start clean
printf '\000' | dd of=vault bs=1 seek=0 count=1 conv=notrunc
./vault
file vault
```

```text
bash: ./vault: cannot execute binary file: Exec format error
vault: data
```

The `dd` options: `bs=1` (1-byte block), `seek=0` (start writing at position 0), `count=1`
(write one block), `conv=notrunc` (do not truncate the file).

> [!TIP]
> Why `\000` and not `\x00`? Some shells (e.g. `dash`, often used as `/bin/sh`) do not support
> `\x` escapes in `printf` and would write a literal `\`. Octal `\000` works everywhere.

> [!IMPORTANT]
> **THE AHA:** Linux recognizes executables by their **magic bytes** (`7f 45 4c 46` = `.ELF`),
> not by file extension. Notice `file` now says just "data" — it reads contents, not names.
> This is your first taste of file formats (Phase 3).

### Exercise 6 — Endianness with your own eyes

**Goal:** see the byte order x86 really uses.

```bash
python3 -c "import struct,sys; sys.stdout.buffer.write(struct.pack('<I', 0x41424344))" | xxd
```

```text
00000000: 4443 4241                                DCBA
```

We wrote the number `0x41424344` (four bytes `41 42 43 44`, the letters `A B C D`), but what
lands in the file is `44 43 42 41` (`D C B A`). The order is reversed.

> [!IMPORTANT]
> **THE AHA:** x86 stores multi-byte values with the **least-significant byte first**. This
> byte order is called **little-endian**. Later, when reading memory dumps in `gdb`, numbers
> will often "look backwards" — now you know why.

### Exercise 7 — The disappearing function

**Goal:** prove the compiler rewrites your code.

```c
static int compute(void) { return 3 * 4 * 5; }
int main(void)           { return compute(); }
```

Resulting assembly (simplified, Intel syntax):

```text
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

Try it yourself:

```bash
gcc -S -O0 -masm=intel c.c -o c_O0.s
gcc -S -O2 -masm=intel c.c -o c_O2.s
```

> [!NOTE]
> Capital **`O`** in `-O0` / `-O2`: lowercase `-o` means "output file name". Real `.s` files
> contain extra lines (`.cfi` directives, `endbr64`, etc.) that were removed above to keep
> the focus.

Two things happened:

1. **Constant folding** — the compiler computed `3*4*5 = 60` at compile time, so there is no
   multiplication instruction at all (even at `-O0`).
2. **Inlining** — at `-O2` the body of `compute()` was copied into `main()`, and `compute`
   was discarded because nothing needs it anymore.

> [!IMPORTANT]
> Machine code is **NOT** a mirror of your source. Recognizing what the compiler did
> (**optimization**) is the core skill for everything that follows.

## 10. Recap

| # | Way | Tool | Analysis | Result |
| :---: | :--- | :--- | :--- | :--- |
| 1 | Read | `strings` | Static | The secret sits in plaintext inside the file |
| 2 | Sniff | `ltrace` | Dynamic | Caught live in memory at comparison time |
| 3 | Patch | `sed` / `dd` | Data edit | We rewrote the data so an answer we know is accepted |

Exercises 5–7 are bonus labs: magic bytes, endianness, and compiler behavior.

### Self-check

You are ready for Phase 1 if you can answer:

- [ ] What is the difference between source code and a binary?
- [ ] What does the compiler throw away, and what survives?
- [ ] Name the 4 stages from `.c` to executable, with their `gcc` flags.
- [ ] When do you use static analysis, and when dynamic?
- [ ] Why is storing a password inside the code dangerous?
- [ ] Why must a replacement string have the same length when patching?

## 11. Cliffhanger: the 4th way

The 4th way needs... **assembly**.

Here is the decision inside `vault` (simplified disassembly):

```asm
    call    strcmp          ; compare input with secret → result in eax (0 = equal)
    test    eax, eax        ; is eax == 0 ?
    jne     .denied         ; if NOT equal → jump to DENIED   ◀── flip THIS
    ; ... puts("ACCESS GRANTED")
.denied:
    ; ... puts("ACCESS DENIED")
```

`jne` = *jump if not equal*.

**Patch the decision itself:** change `jne` into `je` (*jump if equal*) and the logic
inverts — wrong passwords get in. Or overwrite the jump with `NOP` ("do nothing") and **any**
password is accepted. But now you must be able to read and edit real instructions.

> [!IMPORTANT]
> That is exactly what **Phase 1** (architecture) and **Phase 2** (assembly) unlock.

## 12. Quick glossary

| Term | Plain meaning |
| :--- | :--- |
| Source code | The original code written by the programmer, readable by humans |
| Binary | The compiled program file, containing machine code |
| Compiler | Program that translates source code into a binary (e.g. `gcc`) |
| Assembly | Low-level language, almost 1:1 with CPU instructions (`mov`, `lea`, `ret`) |
| Machine code | Instructions as numbers the CPU understands directly |
| Disassembler | Tool that turns machine code into assembly (e.g. `objdump`) |
| Decompiler | Tool that guesses high-level, C-like code from a binary (e.g. Ghidra) |
| Register | Tiny, ultra-fast storage inside the CPU (`eax`, `rdi`, `rsi`) |
| Instruction | One command for the CPU (e.g. `test eax, eax`) |
| Opcode | The number encoding an instruction (`75` = `jne`, `74` = `je`) |
| Static / dynamic analysis | Analyzing without / with running the program |
| Runtime | The moment when the program is actually running |
| Library call | Calling a function from a library (a bundle of ready-made functions), e.g. `strcmp` |
| System call | A request from the program to the kernel (the OS core), e.g. opening a file |
| Debugger | Tool to pause and inspect a running program (`gdb`) |
| Hardcoded credentials | Passwords/keys written directly inside the code |
| ELF | Linux executable format (Windows equivalent: `.exe` / PE) |
| Magic bytes | The first bytes of a file that identify its type (ELF = `7f 45 4c 46`) |
| Hexadecimal (hex) | Base-16 numbers (`0-9`, `a-f`); one byte = two hex digits |
| Endianness | Byte order of multi-byte values; x86 is little-endian |
| Patch | Modifying bytes in a binary to change its behavior |
| NOP | "Do nothing" instruction (opcode `90` on x86) |
| Optimization | The compiler making code faster/smaller (`-O0` = none, `-O2` = moderate) |
| Constant folding | Compiler precomputes constant expressions (`3*4*5` → `60`) |
| Inlining | Compiler copies a function's body into its caller |
| ASLR | OS randomization of memory addresses to make them hard to predict |
| CTF / crackme | Security competition / a practice program built to be broken |
| Pentest | Penetration testing: formally testing a system's security |

---

[← README](../README.md) · [Roadmap](../INDEX.md) · **Next:** PART 1 — Architecture
Fundamentals ([ID notes](../PART1/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>