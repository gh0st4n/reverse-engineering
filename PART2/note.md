# PART 2 — Assembly Language

**English** · [Bahasa Indonesia](note.id.md)

[← README](../README.md) · [Roadmap](../INDEX.md) · **Phase 2 of 10**

> [!NOTE]
> **Source:** [PART 2A on YouTube](https://www.youtube.com/watch?v=HkMetTR67XU) ·
> [PART 3 of the video series](https://www.youtube.com/watch?v=NefasNimlIk) (covers `cmp`/`test`/jump
> and patching — the relevant material is in this part)
> Technical terms stay in English, but every term is explained in plain language first.
> Glossary: [section 11](#11-glossary).

**Prerequisites:** [PART 1](../PART1/note.md) done — you understand registers, flags, stack
frames, and can use `gdb` (`break`, `si`, `info registers`, `x/`).

**After this part you can:** read x86-64 assembly without guessing, tell Intel and AT&T syntax
apart, recognize the most common instructions, explain how `cmp`/`test` becomes a decision, and
**patch a jump instruction** inside a binary you own.

## Table of contents

1. [Why assembly?](#1-why-assembly)
2. [Anatomy of an instruction](#2-anatomy-of-an-instruction)
3. [Reading the two syntaxes](#3-reading-the-two-syntaxes)
4. [Three kinds of operands](#4-three-kinds-of-operands)
5. [Moving data: MOV and LEA](#5-moving-data-mov-and-lea)
6. [Arithmetic, logic, and flags](#6-arithmetic-logic-and-flags)
7. [Control flow: conditional jumps](#7-control-flow-conditional-jumps)
8. [Hands-on: read and patch](#8-hands-on-read-and-patch)
9. [Recap](#9-recap)
10. [Cliffhanger: from instructions to the file](#10-cliffhanger-from-instructions-to-the-file)
11. [Glossary](#11-glossary)

---

## 1. Why assembly?

In PART 1 you changed **data** at runtime (`set $eax = 0`) and watched `vault` accept a wrong
password. That change vanished when the program exited. Now we change the **instruction**
itself, permanently inside the file.

The target is still these two lines:

```asm
test    eax, eax
jne     .denied
```

> [!TIP]
> Continuing PART 1's analogy: if registers are the **nouns** (things being manipulated),
> assembly instructions are the **verbs** (the actions). Reading disassembly means reading short
> sentences: "move this there", "add these", "if zero, jump".

**Assembly** is the human-readable form of **machine code** (the raw bytes the CPU executes).
One assembly line ≈ one CPU instruction. The tool that turns bytes back into that text is a
**disassembler** (`objdump`, `gdb`, Ghidra).

## 2. Anatomy of an instruction

Most x86-64 instructions share the same shape:

```text
mnemonic    destination, source
   │             │         │
   │             │         └── where the value comes from
   │             └──────────── where the result is stored
   └────────────────────────── what operation to perform

examples:
mov         rax,        5        ; rax = 5
add         rax,        rdx      ; rax = rax + rdx
```

Three terms that are easy to mix up:

| Term | What it is | Example |
| :--- | :--- | :--- |
| **Mnemonic** | the textual name of the operation, for humans | `mov`, `add`, `jne` |
| **Operand** | the value or location being operated on | `rax`, `5`, `[rbp-0x8]` |
| **Opcode** | the **number** encoding that instruction inside the file | `75` = `jne`, `74` = `je`, `90` = `nop` |

> [!IMPORTANT]
> Mnemonic ≠ opcode. `jne` is the spelling for us; the byte `75` is what actually lives in the
> binary. Patching an instruction therefore means replacing **bytes**, not text. (Same
> definition as PART 0.)

Not every instruction has two operands: `ret`, `nop`, and `leave` have none, and `inc rax` has one.

## 3. Reading the two syntaxes

The same instruction can be written in two styles. Identical meaning, different ordering:

| Feature | Intel syntax (used in these notes) | AT&T syntax (default `objdump` and `gdb`) |
| :--- | :--- | :--- |
| Operand order | `destination, source` | `source, destination` |
| Registers | `rax` | `%rax` |
| Immediate values | `5` | `$5` |
| Memory | `[rbp-0x8]` | `-0x8(%rbp)` |
| Operation size | `DWORD PTR` | letter suffix: `movl`, `movq` |
| Example | `mov rax, 5` | `mov $5, %rax` |

How to read them: **Intel = "store into the left, take from the right"**; **AT&T = the reverse**.

Configure your tools for Intel:

```gdb
set disassembly-flavor intel
```

```bash
objdump -d -M intel vault
```

> [!NOTE]
> Mind the hyphen in `disassembly-flavor`. Want it permanent? Add that gdb line to `~/.gdbinit`
> (optional).

> [!TIP]
> Do not memorize both. Learn in Intel, but **recognize** AT&T — you will meet it whenever you
> forget `-M intel` or read an older tutorial.

## 4. Three kinds of operands

| Kind | What it is | Example | Analogy |
| :--- | :--- | :--- | :--- |
| **Immediate** | a constant written into the instruction | `5`, `0x25` | a number you say out loud |
| **Register** | storage inside the CPU | `rax`, `edi`, `r8` | what is on the workbench |
| **Memory** | the contents at an address, always in `[...]` | `[rbp-0x8]`, `[rax]` | what is inside a numbered locker |

> [!IMPORTANT]
> **Golden rule:** whenever you see `[...]`, read it as "**the value stored at this address**".
> `rax` is the address (when used as a pointer), `[rax]` is the contents — same as `rsp` vs
> `[rsp]` in PART 1.

### Operand size must be explicit

When the size cannot be inferred from a register, the disassembler spells it out:

| Notation | Size | C equivalent |
| :--- | :--- | :--- |
| `BYTE PTR` | 1 byte | `char` |
| `WORD PTR` | 2 bytes | `short` |
| `DWORD PTR` | 4 bytes | `int` |
| `QWORD PTR` | 8 bytes | `long`, pointer |

```asm
mov     eax, DWORD PTR [rbp-0x4]   ; read 4 bytes (an int)
mov     rax, QWORD PTR [rbp-0x8]   ; read 8 bytes (a pointer)
mov     DWORD PTR [rbp-0x4], 0x0   ; write the 4-byte value 0
```

> [!NOTE]
> Two rules that trip up beginners:
> - `mov` **cannot** go memory → memory. One side must be a register (`mov [a], [b]` is invalid).
> - Writing a 32-bit register (`eax`) **zeroes** the upper 32 bits of `RAX` (PART 1). So
>   `mov eax, 5` sets all of `RAX` to 5, not just the low half.

## 5. Moving data: MOV and LEA

### MOV — copy a value

`mov` **copies** (it does not move): the source is unchanged. There are four patterns:

```asm
mov     rax, 5           ; immediate → register
mov     rax, rdx         ; register  → register
mov     rax, [rbp-0x8]   ; memory    → register (read memory)
mov     [rbp-0x8], rax   ; register  → memory   (write memory)
```

### LEA — take the address, not the contents

`lea` = *Load Effective Address*. It looks like `mov` with `[...]`, but it **never reads
memory**: it stores the computed address itself.

```asm
mov     rax, [rbp-0x8]   ; rax = the CONTENTS at rbp-8
lea     rax, [rbp-0x8]   ; rax = the ADDRESS rbp-8 itself ("address of")
```

> [!TIP]
> Analogy: `mov [address]` = "get what is in the locker". `lea [address]` = "write down the
> locker number".

### LEA doubles as a calculator

Because `lea` can compute `[base + index*scale + offset]` in one instruction, compilers use it
for fast arithmetic — nothing to do with addresses:

```asm
lea     eax, [rdi+rsi]        ; eax = rdi + rsi        ← the PART 0 example
lea     eax, [rdi+rdi*2]      ; eax = rdi * 3
lea     eax, [rax+rax*4]      ; eax = rax * 5
```

> [!IMPORTANT]
> When you see `lea` with an odd pattern like `[rax+rax*2]`, do not read it as a memory access.
> It is **multiplication in disguise**. Spotting tricks like this is the core of Phase 6
> (Reversing Patterns).

Important: `mov` and `lea` do **not** change flags.

## 6. Arithmetic, logic, and flags

### 6.1 Basic arithmetic

```asm
add     rax, rdx     ; rax = rax + rdx
sub     rax, rdx     ; rax = rax - rdx
inc     rax          ; rax = rax + 1
dec     rax          ; rax = rax - 1
imul    rax, rdx     ; rax = rax * rdx (signed multiply)
neg     rax          ; rax = -rax
```

### 6.2 Basic logic

```asm
and     rax, rdx     ; bitwise AND
or      rax, rdx     ; bitwise OR
xor     rax, rdx     ; bitwise XOR
not     rax          ; invert all bits
shl     rax, 1       ; shift left 1  = times 2
shr     rax, 1       ; shift right 1 = divide by 2 (unsigned)
```

### 6.3 Side effect: flags get updated

Arithmetic and logic instructions update `RFLAGS` (PART 1 §3.3) as a side effect:

| Flag | Set when |
| :---: | :--- |
| `ZF` | the result is zero |
| `SF` | the result is negative |
| `CF` | there was a carry/borrow (unsigned) |
| `OF` | the result exceeded the signed range |

> [!NOTE]
> Traps worth remembering:
> - `add`/`sub` update all of the above, but **`inc`/`dec` leave `CF` alone**.
> - `mov`/`lea` change **no** flags.
> - `lea` never reads memory, so it cannot cause a segmentation fault.

### 6.4 Compiler trick: zeroing a register with XOR

To set a register to 0, compilers usually do **not** use `mov eax, 0`. They use:

```asm
xor     eax, eax     ; eax = eax XOR eax = 0
```

Why:

| Instruction | Bytes | Size |
| :--- | :--- | :--- |
| `xor eax, eax` | `31 c0` | 2 bytes |
| `mov eax, 0` | `b8 00 00 00 00` | 5 bytes |

XOR of a value with itself is always zero. Besides being shorter, modern CPUs recognize this
*zeroing idiom* and handle it more efficiently. Side effects: `ZF` is set, `CF` and `OF` cleared.

> [!IMPORTANT]
> When reading disassembly, `xor eax, eax` is **not** a logic operation to think about. Read it
> directly as "**eax = 0**".

## 7. Control flow: conditional jumps

### 7.1 CMP and TEST: compute without storing

These two behave like `sub` and `and`, but **discard the result** — their only job is setting flags:

```asm
cmp     eax, 5       ; compute eax - 5, discard it, set flags
test    eax, eax     ; compute eax AND eax, discard it, set flags
```

| Instruction | Hidden operation | Question it answers |
| :--- | :--- | :--- |
| `cmp a, b` | `a - b` | is `a == b`? greater? smaller? |
| `test a, a` | `a & a` | is `a == 0`? |

### 7.2 Jump instructions read the flags

Once flags are set, a *conditional jump* makes the decision:

| Instruction | Jumps when | Used for | Flags read |
| :--- | :--- | :--- | :--- |
| `je` / `jz` | equal / zero | `==` | `ZF = 1` |
| `jne` / `jnz` | not equal / not zero | `!=` | `ZF = 0` |
| `ja` / `jb` | above / below | `>` `<` **unsigned** | `CF`, `ZF` |
| `jae` / `jbe` | ≥ / ≤ unsigned | `>=` `<=` **unsigned** | `CF`, `ZF` |
| `jg` / `jl` | greater / less | `>` `<` **signed** | `SF`, `OF`, `ZF` |
| `jge` / `jle` | ≥ / ≤ signed | `>=` `<=` **signed** | `SF`, `OF`, `ZF` |
| `jmp` | always | unconditional jump | — |

> [!TIP]
> A quick way to read intent: `cmp` + `jg`/`jl` → **signed** numbers (e.g. `int`).
> `cmp` + `ja`/`jb` → **unsigned** (e.g. `size_t`, a buffer length). That is a hint about types the
> compiler threw away.

### 7.3 vault's full decision chain

```asm
call    strcmp       ; result in EAX (0 = strings equal)
test    eax, eax     ; is EAX == 0 ? → sets ZF
jne     .denied      ; ZF = 0 (not equal) → jump to DENIED
```

**value → flag → decision.** Change any link and behavior changes:

| What you change | How | Result |
| :--- | :--- | :--- |
| value | `set $eax = 0` in gdb (PART 1) | temporary, gone on exit |
| flag | `set $eflags = $eflags \| 0x40` | temporary |
| instruction | patch byte `75` → `74` | **permanent in the file** ← this part |

### 7.4 Jump opcodes worth knowing for patching

A *short jump* (reach ±127 bytes) is 2 bytes: opcode + distance. A *near jump* is 6 bytes and
starts with `0f`:

| Mnemonic | Short (2 bytes) | Near (6 bytes) |
| :--- | :--- | :--- |
| `je` | `74 xx` | `0f 84 xx xx xx xx` |
| `jne` | `75 xx` | `0f 85 xx xx xx xx` |
| `jb` / `jae` | `72` / `73` | `0f 82` / `0f 83` |
| `jbe` / `ja` | `76` / `77` | `0f 86` / `0f 87` |
| `jl` / `jge` | `7c` / `7d` | `0f 8c` / `0f 8d` |
| `jle` / `jg` | `7e` / `7f` | `0f 8e` / `0f 8f` |
| `jmp` | `eb xx` | `e9 xx xx xx xx` |
| `nop` | `90` (1 byte) | `66 90` (2 bytes) |

> [!IMPORTANT]
> **Patching rule:** replace an instruction with one of the **same size**. `75` → `74` is safe
> (1 byte → 1 byte). If you find `0f 85`, its counterpart is `0f 84`, and NOP-ing it takes
> **6 bytes**, not 2. Writing fewer bytes without filling the rest shifts the meaning of every
> following byte and breaks the program.

## 8. Hands-on: read and patch

### 8.1 Build the targets

Reuse the two files you already have: `frame.c` (PART 1) and `vault.c` (PART 0).

```bash
gcc -g -O0 frame.c -o frame
gcc -g -O0 vault.c -o vault
cp vault vault.bak
```

> [!NOTE]
> A `bash` block = type it in your **terminal**. A `gdb` block = type it **inside gdb** (the
> `(gdb)` prompt is omitted so you can copy-paste). `text` = output, `asm` = disassembly.

> [!WARNING]
> Labs 4 and 5 **modify the `vault` file**. Restore anytime with `gcc -g -O0 vault.c -o vault` or
> `cp vault.bak vault`. Only patch binaries you own or are explicitly allowed to analyze.

### 8.2 Lab 1 — Read the `add` function line by line

**Goal:** prove you can read a whole function instead of guessing.

```bash
objdump -d -M intel --no-show-raw-insn frame | sed -n '/<add>:/,/ret/p'
```

```asm
add:
    push   rbp
    mov    rbp,rsp
    mov    DWORD PTR [rbp-0x14],edi
    mov    DWORD PTR [rbp-0x18],esi
    mov    edx,DWORD PTR [rbp-0x14]
    mov    eax,DWORD PTR [rbp-0x18]
    add    eax,edx
    mov    DWORD PTR [rbp-0x4],eax
    mov    eax,DWORD PTR [rbp-0x4]
    pop    rbp
    ret
```

Reading it line by line:

| Line | Read as |
| :--- | :--- |
| `push rbp` / `mov rbp,rsp` | prologue: pin the frame (PART 1) — **skip it** on a first read |
| `mov DWORD PTR [rbp-0x14],edi` | store argument 1 (`a`) into a 4-byte local |
| `mov DWORD PTR [rbp-0x18],esi` | store argument 2 (`b`) |
| `mov edx,[rbp-0x14]` / `mov eax,[rbp-0x18]` | reload both into registers to compute |
| `add eax,edx` | **the actual work**: `eax = b + a` |
| `mov [rbp-0x4],eax` | store the result into the local `sum` |
| `mov eax,[rbp-0x4]` | put `sum` in `EAX` = the return value (ABI) |
| `pop rbp` / `ret` | epilogue: tear down the frame, go home |

> [!IMPORTANT]
> **THE AHA:** Of 11 lines, only **one** does real work (`add eax,edx`). The rest is prologue,
> epilogue, and register↔stack shuffling because we compiled with `-O0`. Reading assembly means
> finding the working line inside the boilerplate.

### 8.3 Lab 2 — Same instruction, two languages

**Goal:** stay calm when you meet AT&T syntax.

```bash
objdump -d --no-show-raw-insn frame | sed -n '/<add>:/,/ret/p'
```

```asm
add:
    push   %rbp
    mov    %rsp,%rbp
    mov    %edi,-0x14(%rbp)
    mov    %esi,-0x18(%rbp)
    mov    -0x14(%rbp),%edx
    mov    -0x18(%rbp),%eax
    add    %edx,%eax
    mov    %eax,-0x4(%rbp)
    mov    -0x4(%rbp),%eax
    pop    %rbp
    ret
```

Compare just one line:

| Intel | AT&T | Meaning |
| :--- | :--- | :--- |
| `mov DWORD PTR [rbp-0x14],edi` | `mov %edi,-0x14(%rbp)` | store `edi` into memory |

> [!NOTE]
> Some binutils versions add size suffixes (`movl`, `addl`, `retq`). Same meaning; `l` = 4 bytes,
> `q` = 8 bytes.

> [!IMPORTANT]
> **THE AHA:** Same instructions, same bytes — only the **spelling** differs. Being confused by
> syntax is not being confused by logic. Set Intel once, then focus on the logic.

### 8.4 Lab 3 — Why the compiler picks `xor`

**Goal:** see the *zeroing idiom* appear in real code.

```bash
printf 'int zero(void) { return 0; }\n' > zero.c
gcc -O2 -c zero.c -o zero.o
objdump -d -M intel zero.o
```

```asm
zero:
    xor    eax,eax
    ret
```

Check the bytes yourself (the `31 c0` in the left column):

```bash
objdump -d zero.o
```

> [!IMPORTANT]
> **THE AHA:** The compiler does not write "return 0" literally. It picks the cheapest
> instruction with the same effect. If you read `xor reg, reg` as a logic operation, you will
> misunderstand it — read it as "**reg = 0**".

### 8.5 Lab 4 — Patch vault's decision: `75` into `74`

**Goal:** invert the password logic permanently inside the file, without recompiling.

**Step 1 — find the instruction and its bytes:**

```bash
objdump -d -M intel vault | grep -A3 'strcmp@plt'
```

```text
    11af:	e8 9c fe ff ff       	call   1050 <strcmp@plt>
    11b4:	85 c0                	test   eax,eax
    11b6:	75 0e                	jne    11c6 <main+0x61>
    11b8:	48 8d 3d 71 0e 00 00 	lea    rdi,[rip+0xe71]
```

Our target: `75 0e` at address `11b6`. `75` = `jne`, `0e` = the jump distance (14 bytes ahead).

**Step 2 — turn the address into a file offset:**

```bash
readelf -SW vault | grep '\.text'
```

```text
  [14] .text  PROGBITS  0000000000001060 001060 0001a2 00  AX  0   0  16
                          ↑ Addr              ↑ Off
```

Formula: `file offset = address - Addr + Off`. In a small binary like this, `Addr` and `Off` are
usually equal, so the file offset is `0x11b6`. **Do not assume — prove it:**

**Step 3 — verify before writing:**

```bash
xxd -s 0x11b6 -l 2 vault
```

```text
000011b6: 750e                                     u.
```

If it prints `750e`, you are in the right place. If not, redo Step 2.

**Step 4 — patch one byte:**

```bash
printf '\164' | dd of=vault bs=1 seek=$((0x11b6)) count=1 conv=notrunc
```

`\164` is octal for `0x74` (`je`) — same trick as `\000` in PART 0, octal works in every shell.
`$((0x11b6))` converts hex to decimal because `dd` does not accept hex.

**Step 5 — prove it:**

```bash
objdump -d -M intel vault | grep -A1 'test.*eax,eax'
./vault
```

```text
    11b6:	74 0e                	je     11c6 <main+0x61>
Enter the vault password: nope
ACCESS GRANTED
```

Now try the **correct** password (`sup3rs3cr3t`) → you get `ACCESS DENIED` instead.

> [!IMPORTANT]
> **THE AHA:** One byte, `75` → `74`, inverts the entire authentication logic. This is the "4th
> way" promised in PART 0: not editing data, not editing registers, but editing the program's
> **decision** itself.

> [!NOTE]
> If your `objdump` shows `0f 85 ...` (a 6-byte near jump), the counterpart is `0f 84` — patch the
> first **2 bytes**, not 1. Addresses also vary between gcc versions; always start from Step 1
> instead of copying `0x11b6` blindly.

### 8.6 Lab 5 — Delete the jump with NOPs

**Goal:** make **any** password accepted, rather than merely inverting the logic.

```bash
gcc -g -O0 vault.c -o vault
printf '\220\220' | dd of=vault bs=1 seek=$((0x11b6)) count=2 conv=notrunc
objdump -d -M intel vault | grep -A2 'test.*eax,eax'
./vault
```

```text
    11b6:	90                   	nop
    11b7:	90                   	nop
    11b8:	48 8d 3d 71 0e 00 00 	lea    rdi,[rip+0xe71]
Enter the vault password: anything
ACCESS GRANTED
```

`\220` is octal for `0x90` (`nop`). The two bytes `75 0e` become two `nop`s, so no jump remains:
execution always falls through into the `ACCESS GRANTED` branch.

> [!IMPORTANT]
> **THE AHA:** `je` **inverts** the decision; `nop` **deletes** it. The size must match exactly
> (2 bytes → 2 bytes) so the following instructions do not shift. That is why patching is always
> about byte accounting.

### 8.7 Runtime variant: offset breakpoint and flag flip

**Goal:** attack the same decision from the dynamic side, without touching the file.

Besides `break strcmp` + `finish` (PART 1), you can stop directly at an instruction offset
inside a function. Take the offset from `objdump`: the `jne` in PART 2 sits at `main+0x51`
(`0x11b6 - 0x1165 = 0x51`).

```gdb
set disassembly-flavor intel
break *main+0x51
run
x/i $rip
info registers eflags
set $eflags |= 0x40
continue
```

```text
Breakpoint 1, 0x00005555555551b6 in main ()
0x5555555551b6 <main+81>:  jne    0x5555555551c6 <main+97>
eflags         0x202               [ IF ]
ACCESS GRANTED
```

`0x40` is the `ZF` bit. `set $eflags |= 0x40` and `set $eflags = $eflags | 0x40` mean the same.

> [!NOTE]
> The offset `main+0x51` applies to the binary built as shown. Always recompute it from
> `objdump` (`instruction address − main address`) instead of copying it blindly.

> [!IMPORTANT]
> **THE AHA:** The same `jne` can be defeated from two sides: the **flag in memory** (runtime,
> temporary) or the **byte in the file** (permanent). The only difference is where you write.

## 9. Recap

A reading order you can use from now on:

1. **Skip the prologue** (`push rbp`, `mov rbp,rsp`, `sub rsp,N`) — frame setup.
2. **Find the `call`s** — where the program interacts (`printf`, `strcmp`, `malloc`).
3. **Find `cmp`/`test` + a conditional jump** — where decisions are made.
4. **Follow the jump direction** — which path is "success", which is "failure".
5. **Look at `eax` before `ret`** — that is the return value.

| Concept | What to remember |
| :--- | :--- |
| Mnemonic vs opcode | `jne` is text for humans, `75` is the byte in the file |
| Syntax | Intel `dst, src`; AT&T `src, dst` with `%` and `$` |
| Operands | immediate (number), register, memory `[...]` = "contents at this address" |
| `mov` vs `lea` | `mov [x]` takes contents; `lea [x]` takes the address (or hidden arithmetic) |
| Flags | `add`/`sub`/`and`/`xor` change them; `mov`/`lea` do not; `inc`/`dec` leave `CF` alone |
| `xor reg, reg` | read as "reg = 0", not as a logic operation |
| Decisions | `cmp`/`test` → flags → `jcc`; the chain value → flag → decision |
| Patching | same-size byte swaps: `75`→`74` inverts, `90 90` deletes |

### Self-check

You are ready for Phase 3 if you can answer:

- [ ] How do mnemonic, operand, and opcode differ?
- [ ] `mov rax, [rbp-8]` vs `lea rax, [rbp-8]` — which one reads memory?
- [ ] Why is `lea eax, [rdi+rdi*2]` really a multiplication?
- [ ] What does `test eax,eax` do, and which flag does it change?
- [ ] `cmp` + `jg` vs `cmp` + `ja` — what type hint does each give you?
- [ ] Why do compilers prefer `xor eax,eax` over `mov eax,0`?
- [ ] Why must a patch be the same byte length, and how many NOPs replace `0f 85`?
- [ ] How do you turn an `objdump` address into a file offset for `dd`?

## 10. Cliffhanger: from instructions to the file

Lab 4 had one step that still felt like magic: **turning address `0x11b6` into a file offset**.
We used `address - Addr + Off` from `readelf`, and the two happened to be equal.

Questions still unanswered:

- Why can a memory address and a position in the file **differ** at all?
- How does `objdump` know the function is called `add`?
- Where exactly does the string `sup3rs3cr3t` live, and why can `strings` find it?
- Why must the file start with `7f 45 4c 46` (PART 0), and what is in the 60 bytes after it?

Every answer lives in the **file header** we have been skipping.

> [!IMPORTANT]
> That is **Phase 3 — Executable Formats**: read the ELF header, sections, and segments so
> patching stops being guesswork and becomes arithmetic.

## 11. Glossary

| Term | Plain meaning |
| :--- | :--- |
| Assembly | low-level language, almost 1:1 with CPU instructions |
| Disassembly | assembly text recovered from machine code |
| Disassembler | tool that turns bytes into assembly (`objdump`, Ghidra) |
| Mnemonic | the textual instruction name for humans (`mov`, `add`, `jne`) |
| Operand | the value or location an instruction operates on |
| Opcode | the **number** encoding an instruction in the file (`75` = `jne`) |
| Destination / Source | the operand receiving the result / supplying the value |
| Intel syntax | writing style `destination, source`, no extra symbols |
| AT&T syntax | writing style `source, destination`, with `%` (register) and `$` (number) |
| Immediate | a constant written directly into the instruction (`5`, `0x25`) |
| Register operand | an operand that is a CPU register (`rax`, `edi`) |
| Memory operand | `[...]` = "the contents at this address" |
| BYTE / WORD / DWORD / QWORD PTR | operand size markers: 1 / 2 / 4 / 8 bytes |
| MOV | copies a value from source to destination (source unchanged) |
| LEA | takes the **address**, not the contents; often used for arithmetic |
| ADD / SUB | add / subtract a value |
| INC / DEC | add 1 / subtract 1 (leave `CF` untouched) |
| IMUL / NEG | signed multiply / negate |
| AND / OR / XOR / NOT | bitwise logic operations |
| SHL / SHR | shift bits left (times 2) / right (unsigned divide by 2) |
| Zeroing idiom | the `xor reg, reg` trick for setting a register to 0 |
| Flags | status bits from the last operation (`ZF`, `SF`, `CF`, `OF`) |
| CMP | computes `a - b`, discards it, only sets flags |
| TEST | computes `a & a`, discards it, only sets flags |
| Conditional jump (`jcc`) | jump only if flags meet a condition (`je`, `jne`, `jg`, ...) |
| JMP | unconditional jump |
| Short / near jump | 2-byte jump (`75 xx`) / 6-byte jump (`0f 85 ...`) |
| Signed vs unsigned jump | `jg`/`jl` for signed numbers, `ja`/`jb` for unsigned |
| NOP | "do nothing" instruction (`90`), used to erase instructions |
| Patch | modify bytes in a binary to change its behavior |
| Boilerplate | supporting instructions (prologue/epilogue), not the core logic |
| File offset | a byte position counted from the start of the file (not a memory address) |

---

[← PART 1](../PART1/note.md) · [Roadmap](../INDEX.md) · **Next:** PART 3 — Executable Formats
([ID notes](../PART3/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>