# PART 6 — Reversing Patterns

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 6 of 10**

> [!NOTE]
> **Source:** written to follow the [roadmap](../INDEX.md). The video labelled "PART 6"
> ([fgWMBp4TVdU](https://www.youtube.com/watch?v=fgWMBp4TVdU)) covers **code injection** — that
> material lives in [PART 9 / Code Injection](../PART9/CodeInjection/note.md).
> Glossary: [section 11](#11-glossary).

**Prerequisites:** [PART 2](../PART2/note.md) (instructions and conditional jumps),
[PART 4](../PART4/note.md) (Ghidra), [PART 5](../PART5/note.md) (gdb).

**After this part you can:** recognize `if`, loop, and `switch` shapes in assembly without the
source, tell `for`/`while`/`do-while` apart, find a jump table in `.rodata`, spot a stack canary,
and infer which optimizations the compiler applied.

## Contents

1. [Why patterns?](#1-why-patterns)
2. [Method: C on the left, assembly on the right](#2-method-c-on-the-left-assembly-on-the-right)
3. [Branches: if, else, and cmov](#3-branches-if-else-and-cmov)
4. [Loops: for, while, do-while](#4-loops-for-while-do-while)
5. [Switch and jump tables](#5-switch-and-jump-tables)
6. [Functions: prologue, arguments, tail calls](#6-functions-prologue-arguments-tail-calls)
7. [Compiler optimization fingerprints](#7-compiler-optimization-fingerprints)
8. [Hands-on: reading patterns](#8-hands-on-reading-patterns)
9. [Recap](#9-recap)
10. [Cliffhanger: from patterns to real targets](#10-cliffhanger-from-patterns-to-real-targets)
11. [Glossary](#11-glossary)

---

## 1. Why patterns?

PART 5 ended with the **volume** problem: finding the spot is easy, but reading 200 functions line
by line is still exhausting.

The way out is not another tool. It is this fact:

> [!IMPORTANT]
> A compiler is a **consistent machine**. The same C structure keeps producing similar assembly
> shapes. Once you know the shapes, you no longer read 40 lines — you see one **pattern** and
> know what it means.

> [!TIP]
> Analogy: reading assembly line by line is like reading handwriting letter by letter.
> Recognizing patterns is reading whole words — you grasp meaning without spelling it out.

## 2. Method: C on the left, assembly on the right

The fastest way to build a pattern library in your head:

```bash
# write a tiny C function, then look at two versions of it
gcc -S -O0 -masm=intel pola.c -o pola_O0.s
gcc -S -O2 -masm=intel pola.c -o pola_O2.s
diff -u pola_O0.s pola_O2.s
```

Or straight from a binary:

```bash
gcc -g -O0 pola.c -o pola
objdump -d -M intel --no-show-raw-insn pola | sed -n '/<function_name>:/,/ret/p'
```

> [!NOTE]
> Every listing here is a **typical gcc x86-64 shape**. Different compiler versions produce
> different orderings and labels. Memorize the **shape**, not the exact text.

## 3. Branches: if, else, and cmov

### The basic `-O0` shape

```c
int classify(int x) { return (x > 5) ? 1 : 2; }
```

```asm
classify:
    mov  DWORD PTR [rbp-0x4],edi
    cmp  DWORD PTR [rbp-0x4],0x5
    jle  .L2                  ; ◀── the condition is INVERTED
    mov  eax,0x1              ; then-branch
    jmp  .L3
.L2:
    mov  eax,0x2              ; else-branch
.L3:
    ret
```

> [!IMPORTANT]
> **The compiler inverts conditions.** `if (x > 5)` becomes `jle` (jump when ≤ 5) heading to the
> **else**. Reason: the fall-through path is reserved for the case considered more likely. If you
> read `jle` and conclude "this is if >", you misread it.

Quick mapping:

| What you see | What it means |
| :--- | :--- |
| `cmp` + forward `jcc`, then a `jmp` skipping a block | `if … else …` |
| `cmp` + forward `jcc`, no closing `jmp` | `if` without `else` |
| `cmp` + **backward** `jcc` | most likely a loop |

### The `-O2` shape: branchless with `cmov`

```c
int max2(int a, int b) { return a > b ? a : b; }
```

```asm
max2:
    cmp    edi,esi
    mov    eax,esi
    cmovge eax,edi            ; conditional move: copy ONLY if the condition holds
    ret
```

`cmov` copies a value **without jumping** — there is no branch at all.

> [!TIP]
> If Ghidra shows an `if` but the Listing has no `jcc`, look for `cmov`. Compilers pick it for
> short branches because it is cheaper than a mispredicted jump.

## 4. Loops: for, while, do-while

### `-O0`: condition at the bottom, an initial jump to it

```c
int sum_to(int n) { int t = 0; for (int i = 1; i <= n; i++) t += i; return t; }
```

```asm
    mov  DWORD PTR [rbp-0x4],0x0     ; t = 0        ← initialization
    mov  DWORD PTR [rbp-0x8],0x1     ; i = 1
    jmp  .L2                          ; ◀── jump to the condition check
.L3:
    mov  eax,DWORD PTR [rbp-0x8]
    add  DWORD PTR [rbp-0x4],eax     ; loop body
    add  DWORD PTR [rbp-0x8],0x1     ; i++          ← step
.L2:
    mov  eax,DWORD PTR [rbp-0x8]
    cmp  eax,DWORD PTR [rbp-0x14]    ; i vs n       ← bound
    jle  .L3                          ; ◀── jump BACKWARD
```

Four loop markers to hunt for:

| Marker | Role |
| :--- | :--- |
| initialization before the block | counter's starting value |
| `cmp` counter vs bound | stop condition |
| `jcc` to a **smaller** address | back to the loop top |
| `add`/`sub`/`inc` on the counter | iteration step |

### Telling the three apart

| C form | Trait at `-O0` |
| :--- | :--- |
| `for` | init + step + check, cleanly separated |
| `while` | similar, but no distinct step block at the body's end |
| `do … while` | **no initial jump**; body first, `cmp` + `jcc` at the end |

> [!NOTE]
> At `-O2`, gcc often inverts loops into a `do-while` shape with a **guard** in front (`cmp` +
> `jcc` to skip the loop when there are zero iterations). Do not infer the original C form from a
> single optimization level — source shape and final shape can differ.

## 5. Switch and jump tables

A `switch` with dense cases (0,1,2,3,4) is **not** compiled into an `if` chain. The compiler builds
a **jump table**: a table of targets in `.rodata`, then jumps by index.

```asm
    cmp    DWORD PTR [rbp-0x4],0x4
    ja     .Ldefault              ; out of range → default
    mov    eax,DWORD PTR [rbp-0x4]
    lea    rdx,[rip+0x...]        ; table address (in .rodata)
    movsxd rax,DWORD PTR [rdx+rax*4]   ; load entry i (4 bytes, signed)
    add    rax,rdx                ; entry = offset relative to the table
    jmp    rax                    ; ◀── indirect jump
```

Its fingerprints:

- `cmp` against an **upper bound** + `ja`/`jb` to `default` → range check;
- an access like `[table + index*4]` or `[table + index*8]`;
- it ends in **`jmp register`**, not `jmp label`.

> [!NOTE]
> Table formats vary by version: older gcc uses absolute 8-byte addresses
> (`jmp QWORD PTR [rax*8+0x402000]`), modern gcc uses 4-byte relative offsets as above. With
> sparse cases (1, 100, 5000) the compiler falls back to a `cmp`/`je` chain — which looks like a
> nested `if-else`.

> [!TIP]
> Jump tables live in `.rodata` (PART 3). Ghidra often recognizes them and reconstructs the
> `switch`; if it does not, define the data as an array of pointers/offsets so the decompiler
> follows suit.

## 6. Functions: prologue, arguments, tail calls

### The prologue is not always there

| Shape | When it appears |
| :--- | :--- |
| `push rbp` / `mov rbp,rsp` / `sub rsp,N` | `-O0`, or a function needing a frame |
| no `rbp`, locals via `[rsp+…]` | `-O2` with frame pointer omission |
| no prologue at all | small leaf functions (call nothing) |

### Arguments are set up before `call`

```asm
    mov  esi,edx        ; argument 2
    mov  edi,eax        ; argument 1
    call   401050 <strcmp@plt>
```

Order: `RDI, RSI, RDX, RCX, R8, R9` (PART 1). Reading the `mov` block before a `call` tells you
who is sending what.

### A pattern that confuses people: `mov eax, 0` before `printf`

```asm
    mov  eax,0x0
    call   401030 <printf@plt>
```

That is **not** a return value. For *variadic* functions (variable arguments like `printf`), `AL`
states how many vector (SSE) registers carry arguments. Zero = no floating-point arguments.

### Tail calls

```asm
    jmp    401030 <puts@plt>      ; not call + ret
```

When a function ends with `jmp` into another function, the compiler is doing a **tail call**: the
current frame is reused, so no `call` followed by `ret` is needed.

> [!IMPORTANT]
> Consequence for analysts: a gdb `backtrace` can "lose" a level, because that frame was never
> created. That is not a debugger bug.

## 7. Compiler optimization fingerprints

| Fingerprint | Example | Meaning |
| :--- | :--- | :--- |
| **Constant folding** | `mov eax,60` for `3*4*5` | computed at compile time (PART 0) |
| **Strength reduction** | `lea eax,[rdi+rdi*2]` for `x*3` | multiplication replaced by cheap math (PART 2) |
| **Shift** | `shl eax,3` for `x*8` | power-of-two multiplication |
| **Inlining** | callee body appears in the caller | the original function may vanish (PART 0) |
| **`cmov`** | branch without `jcc` | branchless |
| **Loop inversion** | `for` looks like `do-while` + guard | source shape ≠ final shape |
| **`rep stos` / SSE** | block memory fill | inlined `memset`/`memcpy` |
| **Tail call** | `jmp` at a function's tail | frame reuse |

### A security pattern: the stack canary

With protections on (compare with the Buffer Overflow lab) you will see:

```asm
    mov  rax,QWORD PTR fs:0x28        ; prologue: fetch the canary
    mov  QWORD PTR [rbp-0x8],rax
    ...
    mov  rdx,QWORD PTR [rbp-0x8]      ; epilogue: compare
    sub  rdx,QWORD PTR fs:0x28
    je   .Lok
    call __stack_chk_fail             ; canary changed → abort
```

> [!TIP]
> Seeing `fs:0x28` + `__stack_chk_fail` means this binary has a stack protector. That matters
> before you choose an analysis approach — and you read it straight from a pattern, not a guess.

## 8. Hands-on: reading patterns

### 8.1 Create the pattern file

```c
/* pola.c */
int classify(int x) { return (x > 5) ? 1 : 2; }
int max2(int a, int b) { return a > b ? a : b; }

int sum_to(int n) {
    int t = 0;
    for (int i = 1; i <= n; i++) t += i;
    return t;
}

int count_down(int n) { while (n > 0) n -= 2; return n; }

int pick(int k) {
    switch (k) {
        case 0: return 10;
        case 1: return 20;
        case 2: return 30;
        case 3: return 40;
        default: return -1;
    }
}
```

```bash
gcc -g -O0 -fno-stack-protector pola.c -o pola_O0
gcc -g -O2 -fno-stack-protector pola.c -o pola_O2
```

> [!NOTE]
> `-fno-stack-protector` is used **only to keep the pattern listings free of canary boilerplate**.
> Once you know the shapes, rebuild without it and compare (Lab 4).

### 8.2 Lab 1 — Find the inverted condition

**Goal:** read `if-else` correctly.

```bash
objdump -d -M intel --no-show-raw-insn pola_O0 | sed -n '/<classify>:/,/ret/p'
```

Mark on paper: where is the `cmp`, where is the `jcc`, which block is *then*, which is *else*.
Then answer: why `jle` when the source says `>`?

> [!IMPORTANT]
> **THE AHA:** You are not reading the source condition, you are reading the condition for
> **skipping** that block. Always follow the jump target before deciding what a branch means.

### 8.3 Lab 2 — Separate loops from ifs

**Goal:** use jump direction as the marker.

```bash
objdump -d -M intel --no-show-raw-insn pola_O0 | sed -n '/<sum_to>:/,/ret/p'
objdump -d -M intel --no-show-raw-insn pola_O0 | sed -n '/<count_down>:/,/ret/p'
```

For each, find: initialization, bound, step, and the **`jcc` target that is smaller** than the
instruction's own address.

> [!IMPORTANT]
> **THE AHA:** Forward `jcc` = branch; backward `jcc` = loop. One direction hint separates the two
> most common structures.

### 8.4 Lab 3 — Take apart a jump table

**Goal:** find the `switch` table and prove where it lives.

```bash
objdump -d -M intel pola_O0 | sed -n '/<pick>:/,/ret/p'
readelf -SW pola_O0 | grep rodata
```

Take the table address from `lea rdx,[rip+…]`, then inspect it:

```bash
xxd -s <table_offset> -l 32 pola_O0
```

> [!NOTE]
> Convert the address to a file offset with the PART 3 formula (`address - Addr + Off`) before
> using `xxd`. Those 4-byte entries are **relative offsets**, not absolute addresses — add the
> table address to get each `case` target.

> [!IMPORTANT]
> **THE AHA:** A `switch` is not an `if` chain, it is **data used as jump destinations**. Control
> flow turns out to be stored as data — and you can read that data.

### 8.5 Lab 4 — Compare optimizations and spot the canary

**Goal:** see the compiler reshape code, and recognize a protection pattern.

```bash
objdump -d -M intel --no-show-raw-insn pola_O2 | sed -n '/<max2>:/,/ret/p'
gcc -g -O0 pola.c -o pola_canary
objdump -d -M intel --no-show-raw-insn pola_canary | grep -E 'fs:0x28|stack_chk'
```

```text
max2:
    cmp    edi,esi
    mov    eax,esi
    cmovge eax,edi
    ret
    mov    rax,QWORD PTR fs:0x28
    sub    rdx,QWORD PTR fs:0x28
    call   401050 <__stack_chk_fail@plt>
```

> [!IMPORTANT]
> **THE AHA:** Two binaries from the same source can look very different. What you recognize is
> the **structural intent**, not the instruction sequence.

### 8.6 Lab 5 — Visual confirmation in Ghidra

**Goal:** use tooling to verify your manual reading.

Import `pola_O0` into Ghidra (PART 4), open `pick`, and enable **Function Graph**. The blocks and
arrows should match the mapping you drew in Lab 3.

> [!TIP]
> An efficient workflow: read patterns manually → verify in the graph → name blocks/functions by
> role. Graphs speed you up; they do not replace pattern understanding.

## 9. Recap

| C structure | Assembly marker |
| :--- | :--- |
| `if … else` | `cmp` + forward `jcc` (inverted) + closing `jmp` |
| `if` without else | `cmp` + forward `jcc`, no `jmp` |
| `for` / `while` | init, bound, step, backward `jcc` |
| `do … while` | body first, no initial jump |
| dense `switch` | range check + table access + `jmp register` |
| sparse `switch` | `cmp`/`je` chain |
| short ternary (`-O2`) | `cmov`, no `jcc` |
| tail call | `jmp` at the function tail |
| stack protector | `fs:0x28` + `__stack_chk_fail` |
| variadic call | `mov eax,0` before `printf` |

### Self-check

- [ ] Why can `if (x > 5)` appear as `jle`?
- [ ] How do you tell a loop from an `if` using only jump direction?
- [ ] What marks `do-while` at `-O0`, and why does `-O2` blur that difference?
- [ ] Which section holds a jump table, and what is inside its entries?
- [ ] What is `cmov`, and why do compilers use it?
- [ ] Why is there a `mov eax,0` before `call printf`?
- [ ] Which pattern indicates an active stack canary?
- [ ] Why can a tail call make a backtrace lose a level?

## 10. Cliffhanger: from patterns to real targets

You now have every foundation: architecture, instructions, file formats, static analysis, dynamic
analysis, and pattern recognition. What you have not done is apply them to a program **designed
to fight back**.

Crackmes are the next drill: small programs with license checks, deliberately twisted logic, and
sometimes traps for analysts.

> [!IMPORTANT]
> That is **Phase 7 — CrackMes & Challenges**: using all these patterns to build keygens and patch
> real practice targets.

## 11. Glossary

| Term | Plain meaning |
| :--- | :--- |
| Reversing pattern | a repeating assembly shape standing for a code structure |
| Basic block | a straight run of instructions with no jump in the middle |
| CFG (Control Flow Graph) | diagram of blocks and arrows describing program flow |
| Inverted condition | the compiler flips the test so fall-through is the common case |
| `jcc` | conditional jump (`je`, `jne`, `jle`, …) |
| Backward jump | `jcc` to a smaller address; a loop marker |
| Loop guard | a check before a loop to skip zero-iteration cases |
| Loop inversion | turning `for`/`while` into a `do-while` shape during optimization |
| Jump table | table of jump targets for a `switch`, usually in `.rodata` |
| Indirect jump | `jmp register` — the target is decided at runtime |
| `movsxd` | load a signed 4-byte value and widen it to 8 bytes |
| `cmov` | conditional move: copies a value without jumping |
| Branchless | code with no branch, using `cmov` |
| Strength reduction | replacing expensive operations with cheap ones (`imul` → `lea`) |
| Constant folding | computing constants at compile time |
| Inlining | inserting a function's body into its caller |
| Tail call | `jmp` to another function at the tail; frame reuse |
| Frame pointer omission | not using `RBP`; locals reached via `RSP` |
| Leaf function | a function that calls no other function |
| Variadic function | a function with variable arguments (`printf`) |
| `rep stos` / `rep movs` | block memory fill/copy instructions |
| Stack canary | stack guard value; pattern `fs:0x28` + `__stack_chk_fail` |
| Boilerplate | supporting instructions that are not the core logic |

---

[← PART 5](../PART5/note.md) · [Roadmap](../INDEX.md) · **Next:** PART 7 — CrackMes &
Challenges (*planned*)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>