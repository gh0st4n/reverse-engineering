# PART 1 — Architecture Fundamentals

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 1 of 10**

> [!NOTE]
> **Source:** [PART 1 on YouTube](https://www.youtube.com/watch?v=eEcXSvd8PCM)
> Technical terms stay in English, but every term is explained in plain language first.
> Glossary: [section 9](#9-glossary).

**Prerequisites:** [PART 0](../PART0/note.md) done — lab installed (`gcc`, `gdb`, `binutils`)
and you have used `strings` and `ltrace` at least once.

**After this part you can:** explain what the CPU does on every instruction, read register
values, point out where code/globals/stack live in memory, take apart a stack frame, and use
`gdb` to inspect function arguments, the return address, and to change registers while the
program runs.

## Table of contents

1. [Why architecture first?](#1-why-architecture-first)
2. [The CPU cycle: fetch-decode-execute](#2-the-cpu-cycle-fetch-decode-execute)
3. [Registers: the CPU workbench](#3-registers-the-cpu-workbench)
4. [Process memory layout](#4-process-memory-layout)
5. [The stack and stack frames](#5-the-stack-and-stack-frames)
6. [Hands-on: GDB](#6-hands-on-gdb)
7. [Recap](#7-recap)
8. [Cliffhanger: from registers to instructions](#8-cliffhanger-from-registers-to-instructions)
9. [Glossary](#9-glossary)

---

## 1. Why architecture first?

In PART 0 you broke `vault` **without** reading assembly: you read the secret from the file,
sniffed it in memory, then patched the data. The 4th way — changing the **decision** — was
blocked by these two lines:

```asm
test    eax, eax
jne     .denied
```

To understand them you need three things: what `eax` is (a register), what `test` actually
tests (flags), and where a function gets its arguments (stack + calling convention). That is
exactly this part.

> [!TIP]
> Analogy: if a binary is a factory, the **CPU** is the worker, **registers** are its workbench
> (tiny but fastest to reach), **RAM** is the warehouse (big but farther away), and the
> **stack** is the pile of trays holding work in progress.

Binary-level reverse engineering is essentially tracking three things: which instruction the
CPU runs, what data sits in registers, and how the stack organizes function calls.

## 2. The CPU cycle: fetch-decode-execute

The CPU (Central Processing Unit) repeats one cycle, instruction after instruction:

| Stage | What happens |
| :--- | :--- |
| **Fetch** | The CPU reads the next instruction from memory, at the address held in `RIP` |
| **Decode** | The CPU translates the instruction number (*opcode*) into internal control signals |
| **Execute** | The CPU performs the operation (compute, move data, or jump) and updates registers/memory |

```text
[ RAM / Text Segment ] ──fetch (address from RIP)──▶ [ CPU Decoder ] ──▶ Execute
                                                                          │
                       ┌──────────────────────────────────────────────────┘
                       ▼
              Update registers & memory ──▶ RIP advances ──▶ fetch again
```

> [!IMPORTANT]
> `RIP` is the CPU's "finger on the current line". Whoever controls `RIP` controls the program.
> That is why every jump instruction (`jmp`, `jne`, `call`, `ret`) matters so much in RE.

## 3. Registers: the CPU workbench

A **register** is ultra-fast storage **inside** the CPU. On x86-64 it is 64 bits (8 bytes)
wide. There are only a few of them, so the compiler must use them carefully.

### 3.1 General Purpose Registers (GPR)

| 64-bit | 32-bit | 16-bit | 8-bit | Main role |
| :--- | :--- | :--- | :--- | :--- |
| `RAX` | `EAX` | `AX` | `AL` | *Accumulator*: arithmetic results and the function's **return value** |
| `RBX` | `EBX` | `BX` | `BL` | general data storage / base pointer |
| `RCX` | `ECX` | `CX` | `CL` | *counter*: loops and string operations |
| `RDX` | `EDX` | `DX` | `DL` | extra data for multiply/divide, I/O |
| `RSI` | `ESI` | `SI` | `SIL` | *source index*: source pointer when copying memory/strings |
| `RDI` | `EDI` | `DI` | `DIL` | *destination index*: destination pointer when copying |
| `R8`–`R15` | `R8D`–`R15D` | `R8W`–`R15W` | `R8B`–`R15B` | extra general-purpose registers added in x86-64 |

These are not separate registers — they are **slices of the same one**: `RAX` (64 bits) →
`EAX` (low 32) → `AX` (low 16) → `AL` (low 8).

> [!NOTE]
> A trap you will meet while reading disassembly: writing to `EAX` **zeroes** the upper 32 bits
> of `RAX`, while writing to `AX`/`AL` leaves the upper bits untouched. So `mov eax, 0` really
> makes `RAX = 0`, but `mov al, 0` does not.

### 3.2 Special pointer registers

| Register | Name | Purpose | Analogy |
| :--- | :--- | :--- | :--- |
| `RIP` | Instruction Pointer | address of the **next** instruction to run | finger on the line being read |
| `RSP` | Stack Pointer | address of the current **top of stack** | the topmost tray |
| `RBP` | Base/Frame Pointer | **base of the active stack frame** | the marker pinning one function's workspace |

`RSP` **decreases** on `PUSH` and **increases** on `POP` (the stack grows toward lower
addresses — see section 5).

### 3.3 RFLAGS: indicator lights for the last operation

`RFLAGS` holds status bits (*flags*) rather than data. They record the outcome of the last
arithmetic/logic operation:

| Flag | Name | Set (=1) when |
| :---: | :--- | :--- |
| `ZF` | Zero Flag | the result is **zero** |
| `SF` | Sign Flag | the result is **negative** (top bit = 1) |
| `CF` | Carry Flag | there was a carry/borrow on **unsigned** numbers |
| `OF` | Overflow Flag | the result exceeded the **signed** range |

`cmp` and `test` compute a result and then **throw it away** — their only job is to set flags.
Conditional jumps then read those flags:

```asm
test    eax, eax     ; compute eax AND eax → zero only if eax == 0 → sets ZF
jne     .denied      ; jump if not equal = jump when ZF = 0
```

> [!IMPORTANT]
> This is the answer to PART 0's cliffhanger: `strcmp` puts its result in `EAX`, `test` turns it
> into `ZF`, and `jne` decides based on `ZF`. The chain is **value → flag → decision**. Change
> any link and the program behaves differently.

Common jump pairs: `je`/`jne` read `ZF`; `jb`/`ja` read `CF` (unsigned); `jl`/`jg` read
`SF`+`OF`+`ZF` (signed).

### 3.4 Calling convention (System V ABI)

An **ABI** (Application Binary Interface) is the agreed rulebook for how functions receive
arguments and return values. On Linux x86-64 (System V ABI):

| Argument | 1 | 2 | 3 | 4 | 5 | 6 | 7 and beyond |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Passed in | `RDI` | `RSI` | `RDX` | `RCX` | `R8` | `R9` | stack |

- The return value goes in `RAX`.
- `RBX`, `RBP`, `R12`–`R15` are *callee-saved* (the called function must restore them); the rest
  are *caller-saved* (the called function may clobber them).

> [!TIP]
> In PART 0, `strcmp(input, secret)` means `RDI` = your input and `RSI` = the secret. That is
> why `x/s $rdi` prints the password you typed.

## 4. Process memory layout

When a program runs, the OS gives it a **virtual address space** (an address range that looks
private to the program, mapped to physical RAM by the OS). It is split into regions:

```text
+-----------------------------------+  high addresses (0x7fff...)
|           Kernel space            |  OS territory, off limits
+-----------------------------------+
|              Stack                |  ▼ grows toward lower addresses
|               ...                 |
|   (large gap: mmap / libraries)   |  ← libc and mapped files live here
|               ...                 |
|              Heap                 |  ▲ grows toward higher addresses
+-----------------------------------+
|      BSS  (globals, no value)     |
+-----------------------------------+
|      Data (globals with values)   |
+-----------------------------------+
|      Text (program code) r-x      |
+-----------------------------------+  low addresses
```

| Region | Contents | Permissions | Analogy |
| :--- | :--- | :--- | :--- |
| **Text** (`.text`) | program instructions | read + execute, **not writable** | the recipe book (no scribbling) |
| **Data** (`.data`) | global/static variables with initial values | read + write | a whiteboard already filled in |
| **BSS** (`.bss`) | global/static variables without initial values (zero-filled) | read + write | a blank whiteboard |
| **Heap** | dynamic allocation: `malloc`, `calloc`, `new` | read + write, grows up | warehouse space rented on demand |
| **Stack** | arguments, locals, return addresses | read + write, grows down | pile of work trays |

> [!NOTE]
> - Text is *read-only* so code cannot be rewritten at runtime. That is why PART 0's patches
>   targeted the **file**, not live memory.
> - The addresses above are schematic. Because of **ASLR** (and **PIE** for modern binaries),
>   real addresses change on every run.
> - `gdb` disables randomization by default (`set disable-randomization on`), so addresses look
>   stable **inside** gdb even though they differ outside it.

## 5. The stack and stack frames

### 5.1 LIFO in practice

The **stack** is a **LIFO** structure (Last-In, First-Out): the last item pushed is the first
popped — like a pile of plates. `RSP` always points at the top. On x86-64 one slot = 8 bytes.

| Operation | Steps | Effect on `RSP` |
| :--- | :--- | :--- |
| `PUSH <value>` | `RSP = RSP - 8`, then write the value to `[RSP]` | decreases by 8 |
| `POP <register>` | read `[RSP]` into the register, then `RSP = RSP + 8` | increases by 8 |

Brackets `[...]` mean **dereference**: "the memory contents at that address", not the address
itself. So `rsp` is the address, `[rsp]` is what is stored there.

### 5.2 What CALL and RET do

`CALL <function>` does two things at once:

1. Pushes the **return address** (the address of the instruction right after the `CALL`).
2. Jumps to the target (`RIP = function address`).

`RET` does the reverse: pops the return address into `RIP`, so execution resumes in the caller.

> [!IMPORTANT]
> The return address lives **inside the stack** — writable data. If a program can be forced to
> write past a buffer's boundary and overwrite that address, control flow can be hijacked.
> That is the root of **buffer overflow** (lab in Phase 5).

### 5.3 Prologue and epilogue

Every function builds its own workspace (*stack frame*) with a standard pattern:

```asm
; ---- prologue ----
push    rbp          ; save the caller's RBP
mov     rbp, rsp     ; pin the new frame here
sub     rsp, 0x20    ; reserve 32 bytes for locals

; ---- body: locals are reached via RBP offsets ----
mov     DWORD PTR [rbp-0x4], eax    ; a local variable

; ---- epilogue ----
mov     rsp, rbp     ; discard locals
pop     rbp          ; restore the caller's RBP
ret                  ; return to the return address
```

> [!NOTE]
> - The two epilogue instructions `mov rsp, rbp` + `pop rbp` are often fused into the single
>   instruction **`leave`** — the one you saw in `vault`'s disassembly in PART 0.
> - `sub rsp, N` is usually a multiple of 16: the ABI requires the stack to be **16-byte
>   aligned** at a `call`. Otherwise some library functions (e.g. ones using SSE) can crash.
> - With frame-pointer omission (typical at `-O2`), `RBP` becomes a normal register and locals
>   are reached via `RSP`. Do not be surprised when the pattern disappears.

### 5.4 Anatomy of a stack frame

Real example from the lab in section 6 (`add(7, 5)`):

```text
        high addresses
        +---------------------------+
        |   ... main's frame ...    |
        +---------------------------+
        |   return address          |  ← [rbp+8]
        +---------------------------+
        |   saved rbp               |  ← rbp   (add's frame marker)
        +---------------------------+
        |   sum  (local)            |  ← [rbp-0x4]
        +---------------------------+
        |   a    (spilled argument) |  ← [rbp-0x14]
        +---------------------------+
        |   b    (spilled argument) |  ← [rbp-0x18]  ← rsp
        +---------------------------+
        low addresses (the stack grows this way)
```

Full flow: arguments arrive in registers → `call` pushes the return address → the prologue
pins `RBP` → locals are accessed as `[rbp-offset]` → the epilogue tears the frame down →
`ret` goes home.

## 6. Hands-on: GDB

### 6.1 Build the target

Create `frame.c`:

```c
#include <stdio.h>

int add(int a, int b) {
    int sum = a + b;
    return sum;
}

int main(void) {
    int x = 7, y = 5;
    int r = add(x, y);
    printf("r = %d\n", r);
    return 0;
}
```

```bash
gcc -g -O0 frame.c -o frame
gdb -q ./frame
```

`-g` includes **debug symbols** (function names, variable names, line numbers) so gdb can refer
to `add` and `sum` by name. `-O0` disables optimization so nothing gets folded away as in
PART 0 Exercise 7. `-q` just hides gdb's banner.

### 6.2 Command block conventions

> [!NOTE]
> A `bash` block = type it in your **terminal**. A `gdb` block = type it **inside gdb** (the
> `(gdb)` prompt is omitted so you can copy-paste directly). A `text` block = output.

Do this first, always:

```gdb
set disassembly-flavor intel
```

Without it gdb shows **AT&T** syntax (`mov %rsp,%rbp`), whose operand order is reversed
compared to the **Intel** syntax (`mov rbp,rsp`) used since PART 0. Want it permanent? Add that
line to `~/.gdbinit` (optional).

### 6.3 Essential GDB commands

| Command | Short | What it does |
| :--- | :--- | :--- |
| `disassemble main` | `disas main` | show `main`'s assembly |
| `disassemble /r main` | — | assembly **plus raw bytes** (for patching later) |
| `break *add` | `b *add` | stop at the **very first instruction** of `add` |
| `break add` | `b add` | stop at the function body (prologue usually skipped) |
| `break frame.c:5` | — | stop at a source line (needs `-g`) |
| `run` | `r` | start the program |
| `starti` | — | stop at the program's very first instruction |
| `info registers` | `i r` | show all registers |
| `info registers eax eflags` | — | show only certain registers |
| `stepi` | `si` | advance **one instruction**, stepping into calls |
| `nexti` | `ni` | advance one instruction, stepping over calls |
| `finish` | — | run until the current function returns, print its return value |
| `continue` | `c` | resume until the next breakpoint |
| `backtrace` | `bt` | show the chain of function calls |
| `info frame` | — | details of the active frame (saved rbp, return address) |
| `p/x $rax` | — | print a register in hex |
| `set $rax = 0` | — | **change** a register while the program runs |
| `layout regs` | — | TUI view: registers and assembly side by side |
| `quit` | `q` | exit |

### 6.4 Reading memory with `x/`

Format: `x/<count><format><size> <address>`

- **count** = how many elements to show.
- **format** = `x` hex, `d` decimal, `s` string, `i` instruction.
- **size** = `b` 1 byte, `h` 2 bytes, `w` 4 bytes, `g` 8 bytes.

| Command | Meaning |
| :--- | :--- |
| `x/16gx $rsp` | 16 **8-byte** slots from the top of stack, in hex (best fit for x86-64 stacks) |
| `x/20wx $rsp` | 20 4-byte slots from the top of stack (handy for `int` arrays) |
| `x/10i $rip` | 10 instructions starting at the current position |
| `x/s $rdi` | the string pointed to by `RDI` |
| `x/gx $rbp` | the 8 bytes at `RBP` (= saved rbp) |
| `x/wx $rbp-0x4` | a 4-byte local (`int`) at `[rbp-0x4]` |

### 6.5 Lab 1 — See arguments, locals, and the return address

**Goal:** prove that arguments arrive in registers, locals live on the stack, and the return
address is stored there too.

```gdb
set disassembly-flavor intel
break *add
run
info registers rdi rsi rip rsp
x/gx $rsp
```

Output (trimmed; your addresses will differ):

```text
Breakpoint 1, 0x0000555555555135 in add ()
rdi            0x7                 7
rsi            0x5                 5
rip            0x555555555135      0x555555555135 <add>
rsp            0x7fffffffe098      0x7fffffffe098
0x7fffffffe098: 0x0000555555555174
```

Read it: `rdi = 7` and `rsi = 5` are `x` and `y` sent by `main`, exactly as the ABI promises.
The value at `[rsp]` is the **return address**: the instruction in `main` right after
`call add`.

Now run the prologue one instruction at a time:

```gdb
si
si
info registers rbp rsp
x/2gx $rsp
```

```text
rbp            0x7fffffffe090      0x7fffffffe090
rsp            0x7fffffffe090      0x7fffffffe090
0x7fffffffe090: 0x00007fffffffe0b0   0x0000555555555174
```

`RSP` dropped 8 bytes (`...e098` → `...e090`) because of `push rbp`. After `mov rbp, rsp`, `RBP`
and `RSP` match. The top two stack slots are now **saved rbp** then **return address** — exactly
the picture in section 5.4.

Move into the body to see the locals:

```gdb
break frame.c:5
continue
info locals
x/wx $rbp-0x4
x/wx $rbp-0x14
x/wx $rbp-0x18
finish
```

```text
sum = 12
0x7fffffffe08c: 0x0000000c
0x7fffffffe07c: 0x00000007
0x7fffffffe078: 0x00000005
Run till exit from #0  add (a=7, b=5) at frame.c:5
main () at frame.c:11
11          printf("r = %d\n", r);
Value returned is $1 = 12
```

Compare with `add`'s disassembly (`objdump -d -M intel frame`, verified):

```asm
add:
    push   rbp
    mov    rbp,rsp
    mov    DWORD PTR [rbp-0x14],edi    ; a copied from RDI onto the stack
    mov    DWORD PTR [rbp-0x18],esi    ; b copied from RSI onto the stack
    mov    edx,DWORD PTR [rbp-0x14]
    mov    eax,DWORD PTR [rbp-0x18]
    add    eax,edx                     ; the computation happens in EAX
    mov    DWORD PTR [rbp-0x4],eax     ; sum = local variable
    mov    eax,DWORD PTR [rbp-0x4]     ; return value goes in EAX
    pop    rbp
    ret
```

> [!IMPORTANT]
> **THE AHA:** The names `a`, `b`, and `sum` were thrown away by the compiler (PART 0), but
> their **locations survive**: `[rbp-0x14]`, `[rbp-0x18]`, `[rbp-0x4]`. A reverser recognizes
> variables by offset patterns, not by names.

### 6.6 Lab 2 — Flip vault's decision through a register

**Goal:** pay off PART 0's cliffhanger — get into `vault` **without changing a single byte** of
the file, by editing a register instead.

```bash
gcc -g -O0 vault.c -o vault
gdb -q ./vault
```

```gdb
set disassembly-flavor intel
break strcmp
run
```

Type any wrong password, e.g. `nope`, and press Enter. The program stops inside `strcmp`.
Return to `main` and look at the instructions waiting there:

```gdb
finish
x/2i $rip
info registers eax
```

```text
Run till exit from #0  0x00007ffff7e0a2c0 in strcmp () from /lib/x86_64-linux-gnu/libc.so.6
main () at vault.c:9
9           if (strcmp(input, secret) == 0) {
0x5555555551b4 <main+79>:  test   eax,eax
0x5555555551b6 <main+81>:  jne    0x5555555551c6 <main+97>
eax            0x1                 1
```

`EAX = 1` (non-zero) means the strings differ. Change it **before** `test` runs, then watch the
flags:

```gdb
set $eax = 0
si
info registers eflags
continue
```

```text
eflags         0x246               [ PF ZF IF ]
ACCESS GRANTED
```

The chain: `EAX = 0` → `test eax,eax` yields zero → **`ZF` turns on** → `jne` (jump when
`ZF = 0`) does **not** jump → execution falls into the `ACCESS GRANTED` branch.

> [!TIP]
> You can also set the flag directly: `set $eflags = $eflags | 0x40` (`0x40` is the `ZF` bit).
> To see the "before" state, run `info registers eflags` prior to `si`: without `ZF` it reads
> `0x202 [ IF ]`.

> [!NOTE]
> `break strcmp` works because `vault` links dynamically against libc. For a static or
> *stripped* binary you would break at an address instead — covered in Phases 3 and 5.

> [!IMPORTANT]
> **THE AHA:** PART 0 changed **data in the file** (permanent). This lab changed **data in
> memory at runtime** (gone when the program exits — restart gdb and you get `ACCESS DENIED`
> again). Both bend the decision without touching the logic. Next step: change the
> **instruction** itself.

### 6.7 Watch it change in real time

While stepping with `si`, keep an eye on three things:

- [ ] `RIP` advances by each instruction's length, and **jumps** on `call`/`jne`/`ret`.
- [ ] `RSP` drops 8 on `push`, rises 8 on `pop`.
- [ ] The stack slot at `[rsp]` becomes a return address the moment `call` executes.

To avoid retyping commands constantly:

```gdb
display/i $rip
layout regs
```

`display/i $rip` prints the next instruction automatically at every stop; `layout regs` opens
the TUI (leave it with `Ctrl+X` then `A`, or `tui disable`).

## 7. Recap

| Concept | What to remember |
| :--- | :--- |
| CPU & `RIP` | fetch-decode-execute cycle; `RIP` selects the next instruction |
| Registers | fastest storage; `RAX` return value, `RDI`/`RSI` first/second argument |
| `RFLAGS` | `cmp`/`test` set flags, conditional jumps read them (`ZF` → `je`/`jne`) |
| Memory | code in Text (read-only), globals in Data/BSS, dynamic in Heap, calls in Stack |
| Stack | LIFO, grows toward low addresses; `PUSH`/`POP` move `RSP` by 8 |
| Stack frame | `RBP` is the marker; locals = `[rbp-offset]`; return address = `[rbp+8]` |
| GDB | `break *func`, `si`/`ni`, `info registers`, `x/`, `set $reg` |

### Self-check

You are ready for Phase 2 if you can answer:

- [ ] What are the three CPU stages, and what does `RIP` do in them?
- [ ] Where do arguments 1 through 6 live on Linux x86-64?
- [ ] How do `RSP` and `RBP` differ, and why does `RSP` decrease on `push`?
- [ ] What does `CALL` store on the stack, and what does `RET` do with it?
- [ ] Why does `test eax,eax` decide whether `jne` jumps?
- [ ] Why can't code in the Text segment be patched while the program runs?
- [ ] How do you make `vault` accept a wrong password **without** editing the file?

## 8. Cliffhanger: from registers to instructions

You can now read the machine's **state** (registers, flags, stack) and change its **data**.
What you cannot do yet: read and write its **instructions**.

Look at the raw bytes behind `vault`'s decision:

```gdb
disassemble /r main
```

You will see something like `75 0e` next to `jne`. `75` is the *opcode* for `jne`, and `0e` is
the jump distance. Change `75` into `74` (`je`) and the logic is permanently inverted inside the
file — exactly the threat mentioned in PART 0.

To do that safely, though, you must be able to read *any* instruction, not just guess one byte.

> [!IMPORTANT]
> That is **Phase 2 — Assembly Language**: read x86-64 disassembly fluently, then decide which
> instructions are safe to change.

---

[← PART 0](../PART0/note.md) · [Roadmap](../INDEX.md) · **Next:** PART 2 — Assembly Language
([ID notes](../PART2/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>