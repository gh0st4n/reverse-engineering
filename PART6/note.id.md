# PART 6 — Pola Reversing (Reversing Patterns)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 6 dari 10**

> [!NOTE]
> **Sumber:** catatan mandiri mengikuti [peta belajar](../INDEX.id.md). Video berlabel "PART 6"
> ([fgWMBp4TVdU](https://www.youtube.com/watch?v=fgWMBp4TVdU)) membahas **code injection** —
> materi itu ada di [PART 9 / Code Injection](../PART9/CodeInjection/note.id.md).
> Kamus istilah: [bagian 11](#11-kamus-istilah).

**Prasyarat:** [PART 2](../PART2/note.id.md) (instruksi & loncat bersyarat),
[PART 4](../PART4/note.id.md) (Ghidra), [PART 5](../PART5/note.id.md) (gdb).

**Setelah part ini kamu bisa:** mengenali bentuk `if`, loop, dan `switch` di assembly tanpa
melihat source-nya, membedakan `for`/`while`/`do-while`, menemukan jump table di `.rodata`,
mengenali stack canary, dan menebak optimasi apa yang dilakukan compiler.

## Daftar isi

1. [Kenapa pola?](#1-kenapa-pola)
2. [Metode: C di kiri, assembly di kanan](#2-metode-c-di-kiri-assembly-di-kanan)
3. [Percabangan: if, else, dan cmov](#3-percabangan-if-else-dan-cmov)
4. [Perulangan: for, while, do-while](#4-perulangan-for-while-do-while)
5. [Switch dan jump table](#5-switch-dan-jump-table)
6. [Fungsi: prologue, argumen, tail call](#6-fungsi-prologue-argumen-tail-call)
7. [Jejak optimasi compiler](#7-jejak-optimasi-compiler)
8. [Praktik: membaca pola](#8-praktik-membaca-pola)
9. [Rekap](#9-rekap)
10. [Cliffhanger: dari pola ke target nyata](#10-cliffhanger-dari-pola-ke-target-nyata)
11. [Kamus istilah](#11-kamus-istilah)

---

## 1. Kenapa pola?

PART 5 ditutup dengan masalah **volume**: menemukan tempat itu mudah, tapi membaca 200 fungsi
baris demi baris tetap melelahkan.

Jalan keluarnya bukan tool baru, tapi fakta ini:

> [!IMPORTANT]
> Compiler itu **mesin yang konsisten**. Struktur C yang sama selalu menghasilkan bentuk
> assembly yang mirip. Kalau kamu hafal bentuknya, kamu tidak lagi membaca 40 baris — kamu
> melihat satu **pola** dan langsung tahu maksudnya.

> [!TIP]
> Analogi: membaca assembly baris demi baris itu seperti membaca tulisan tangan huruf per
> huruf. Mengenali pola itu seperti membaca kata — kamu langsung menangkap maknanya tanpa
> mengeja.

## 2. Metode: C di kiri, assembly di kanan

Cara tercepat membangun pustaka pola di kepalamu:

```bash
# tulis fungsi C kecil, lalu lihat dua versinya
gcc -S -O0 -masm=intel pola.c -o pola_O0.s
gcc -S -O2 -masm=intel pola.c -o pola_O2.s
diff -u pola_O0.s pola_O2.s
```

Atau langsung dari binary:

```bash
gcc -g -O0 pola.c -o pola
objdump -d -M intel --no-show-raw-insn pola | sed -n '/<nama_fungsi>:/,/ret/p'
```

> [!NOTE]
> Semua listing di catatan ini adalah **bentuk umum gcc x86-64**. Versi compiler berbeda bisa
> menghasilkan urutan/label berbeda. Yang kamu hafal adalah **bentuknya**, bukan teks persisnya.

## 3. Percabangan: if, else, dan cmov

### Bentuk dasar `-O0`

```c
int classify(int x) { return (x > 5) ? 1 : 2; }
```

```asm
classify:
    mov  DWORD PTR [rbp-0x4],edi
    cmp  DWORD PTR [rbp-0x4],0x5
    jle  .L2                  ; ◀── kondisi DIBALIK
    mov  eax,0x1              ; then-branch
    jmp  .L3
.L2:
    mov  eax,0x2              ; else-branch
.L3:
    ret
```

> [!IMPORTANT]
> **Compiler membalik kondisi.** `if (x > 5)` menjadi `jle` (loncat jika ≤ 5) menuju **else**.
> Alasannya: jalur "lurus" (tanpa loncat) dibuat untuk kasus yang dianggap lebih mungkin.
> Kalau kamu membaca `jle` lalu menyimpulkan "ini if >", kamu salah baca.

Cara cepat memetakannya:

| Yang kamu lihat | Artinya |
| :--- | :--- |
| `cmp` + `jcc` ke depan, lalu `jmp` melewati blok | `if … else …` |
| `cmp` + `jcc` ke depan, tanpa `jmp` penutup | `if` tanpa `else` |
| `cmp` + `jcc` ke **belakang** | kemungkinan besar loop |

### Bentuk `-O2`: branchless dengan `cmov`

```c
int max2(int a, int b) { return a > b ? a : b; }
```

```asm
max2:
    cmp    edi,esi
    mov    eax,esi
    cmovge eax,edi            ; conditional move: pindah HANYA jika syarat terpenuhi
    ret
```

`cmov` menyalin nilai **tanpa melompat** — tidak ada percabangan sama sekali.

> [!TIP]
> Kalau Ghidra menampilkan `if` tapi Listing tidak punya `jcc`, cari `cmov`. Compiler memilihnya
> untuk percabangan pendek karena lebih murah daripada salah prediksi loncatan.

## 4. Perulangan: for, while, do-while

### `-O0`: kondisi di bawah, lompatan awal ke sana

```c
int sum_to(int n) { int t = 0; for (int i = 1; i <= n; i++) t += i; return t; }
```

```asm
    mov  DWORD PTR [rbp-0x4],0x0     ; t = 0        ← inisialisasi
    mov  DWORD PTR [rbp-0x8],0x1     ; i = 1
    jmp  .L2                          ; ◀── lompat ke cek kondisi
.L3:
    mov  eax,DWORD PTR [rbp-0x8]
    add  DWORD PTR [rbp-0x4],eax     ; badan loop
    add  DWORD PTR [rbp-0x8],0x1     ; i++          ← langkah
.L2:
    mov  eax,DWORD PTR [rbp-0x8]
    cmp  eax,DWORD PTR [rbp-0x14]    ; i vs n       ← batas
    jle  .L3                          ; ◀── loncat KE BELAKANG
```

Empat penanda loop yang harus kamu cari:

| Penanda | Peran |
| :--- | :--- |
| inisialisasi sebelum blok | nilai awal counter |
| `cmp` counter vs batas | kondisi berhenti |
| `jcc` ke alamat **lebih kecil** | kembali ke awal loop |
| `add`/`sub`/`inc` pada counter | langkah iterasi |

### Membedakan ketiganya

| Bentuk C | Ciri di `-O0` |
| :--- | :--- |
| `for` | inisialisasi + langkah + cek, semuanya terpisah rapi |
| `while` | mirip `for`, tapi tanpa blok langkah yang jelas di akhir badan |
| `do … while` | **tidak ada lompatan awal**; badan dulu, `cmp` + `jcc` di akhir |

> [!NOTE]
> Di `-O2`, gcc sering membalik loop menjadi bentuk `do-while` dengan **guard** di depan
> (`cmp` + `jcc` untuk melewati loop kalau iterasi nol). Jadi jangan menyimpulkan bentuk C asli
> hanya dari satu versi optimasi — bentuk source bisa berbeda dari bentuk akhir.

## 5. Switch dan jump table

`switch` dengan kasus rapat (0,1,2,3,4) **tidak**compiled jadi rantai `if`. Compiler membuat
**jump table**: tabel alamat di `.rodata`, lalu melompat lewat indeks.

```asm
    cmp    DWORD PTR [rbp-0x4],0x4
    ja     .Ldefault              ; di luar rentang → default
    mov    eax,DWORD PTR [rbp-0x4]
    lea    rdx,[rip+0x...]        ; alamat tabel (di .rodata)
    movsxd rax,DWORD PTR [rdx+rax*4]   ; ambil entri ke-i (4 byte, bertanda)
    add    rax,rdx                ; entri = offset relatif terhadap tabel
    jmp    rax                    ; ◀── loncat tak langsung
```

Ciri khasnya:

- `cmp` dengan **batas atas** + `ja`/`jb` ke `default` → pemeriksaan rentang;
- akses `[tabel + indeks*4]` atau `[tabel + indeks*8]`;
- diakhiri **`jmp register`**, bukan `jmp label`.

> [!NOTE]
> Bentuk tabel berbeda antar versi: gcc lama memakai alamat absolut 8 byte
> (`jmp QWORD PTR [rax*8+0x402000]`), gcc modern memakai offset relatif 4 byte seperti di atas.
> Kalau kasus `switch` jarang/berjauhan (1, 100, 5000), compiler kembali memakai rantai
> `cmp`/`je` — terlihat seperti `if-else` bertingkat.

> [!TIP]
> Jump table tinggal di `.rodata` (PART 3). Di Ghidra, tabel seperti ini sering dikenali otomatis
> dan `switch`-nya direkonstruksi — kalau tidak, tandai datanya sebagai array pointer/offset
> agar decompiler ikut memperbaikinya.

## 6. Fungsi: prologue, argumen, tail call

### Prologue tidak selalu ada

| Bentuk | Kapan muncul |
| :--- | :--- |
| `push rbp` / `mov rbp,rsp` / `sub rsp,N` | `-O0`, atau fungsi butuh frame |
| tanpa `rbp`, lokal lewat `[rsp+…]` | `-O2` dengan *frame pointer omission* |
| tanpa prologue sama sekali | fungsi daun kecil (tidak memanggil siapa pun) |

### Argumen disiapkan sebelum `call`

```asm
    mov  esi,edx        ; argumen ke-2
    mov  edi,eax        ; argumen ke-1
    call   401050 <strcmp@plt>
```

Urutan `RDI, RSI, RDX, RCX, R8, R9` (PART 1). Membaca blok `mov` sebelum `call` = membaca
"siapa mengirim apa".

### Pola yang sering bikin bingung: `mov eax, 0` sebelum `printf`

```asm
    mov  eax,0x0
    call   401030 <printf@plt>
```

Itu **bukan** nilai kembalian. Untuk fungsi *variadic* (argumen berubah-ubah seperti `printf`),
`AL` memberi tahu berapa banyak register vektor (SSE) yang dipakai untuk argumen. Nol = tidak
ada argumen floating-point.

### Tail call

```asm
    jmp    401030 <puts@plt>      ; bukan call + ret
```

Kalau fungsi berakhir dengan `jmp` ke fungsi lain, compiler sedang melakukan **tail call**:
frame fungsi saat ini dipakai ulang, jadi tidak perlu `call` lalu `ret`.

> [!IMPORTANT]
> Efeknya untuk analis: `backtrace` di gdb bisa "kehilangan" satu tingkat, karena frame-nya
> memang tidak pernah dibuat. Bukan bug debugger.

## 7. Jejak optimasi compiler

| Jejak | Contoh | Artinya |
| :--- | :--- | :--- |
| **Constant folding** | `mov eax,60` untuk `3*4*5` | dihitung saat kompilasi (PART 0) |
| **Strength reduction** | `lea eax,[rdi+rdi*2]` untuk `x*3` | perkalian diganti perhitungan murah (PART 2) |
| **Shift** | `shl eax,3` untuk `x*8` | perkalian pangkat dua |
| **Inlining** | isi fungsi muncul di pemanggil | fungsi asli bisa hilang (PART 0) |
| **`cmov`** | percabangan tanpa `jcc` | branchless |
| **Loop inversion** | `for` terlihat seperti `do-while` + guard | bentuk source ≠ bentuk akhir |
| **`rep stos` / SSE** | pengisian blok memori | `memset`/`memcpy` yang di-inline |
| **Tail call** | `jmp` di ekor fungsi | frame dipakai ulang |

### Pola keamanan: stack canary

Kalau proteksi aktif (bandingkan dengan lab Buffer Overflow), kamu akan melihat:

```asm
    mov  rax,QWORD PTR fs:0x28        ; prologue: ambil canary
    mov  QWORD PTR [rbp-0x8],rax
    ...
    mov  rdx,QWORD PTR [rbp-0x8]      ; epilogue: bandingkan
    sub  rdx,QWORD PTR fs:0x28
    je   .Lok
    call __stack_chk_fail             ; canary berubah → abort
```

> [!TIP]
> Melihat pola `fs:0x28` + `__stack_chk_fail` = binary ini punya stack protector. Itu informasi
> penting sebelum kamu memilih pendekatan analisis, dan langsung terbaca dari pola — bukan dari
> menebak.

## 8. Praktik: membaca pola

### 8.1 Siapkan berkas pola

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
> `-fno-stack-protector` dipakai **hanya agar listing pola bersih** dari boilerplate canary.
> Setelah paham polanya, kompilasi ulang tanpa flag itu dan lihat perbedaannya (Lab 4).

### 8.2 Lab 1 — Temukan kondisi yang dibalik

**Tujuan:** membaca `if-else` dengan benar.

```bash
objdump -d -M intel --no-show-raw-insn pola_O0 | sed -n '/<classify>:/,/ret/p'
```

Tandai di kertas: mana `cmp`, mana `jcc`, blok mana yang jadi *then*, mana *else*. Lalu jawab:
kenapa `jle` padahal source-nya `>`?

> [!IMPORTANT]
> **THE AHA:** Yang kamu baca bukan kondisi source, tapi **kondisi untuk melompati** blok itu.
> Selalu telusuri target loncatannya sebelum menyimpulkan makna percabangan.

### 8.3 Lab 2 — Bedakan loop dari if

**Tujuan:** memakai arah loncatan sebagai penanda.

```bash
objdump -d -M intel --no-show-raw-insn pola_O0 | sed -n '/<sum_to>:/,/ret/p'
objdump -d -M intel --no-show-raw-insn pola_O0 | sed -n '/<count_down>:/,/ret/p'
```

Untuk masing-masing, temukan: inisialisasi, batas, langkah, dan **alamat tujuan `jcc` yang lebih
kecil** dari alamat instruksinya.

> [!IMPORTANT]
> **THE AHA:** `jcc` ke depan = percabangan; `jcc` ke belakang = perulangan. Satu petunjuk arah
> sudah cukup untuk memisahkan dua struktur yang paling umum.

### 8.4 Lab 3 — Bongkar jump table

**Tujuan:** menemukan tabel `switch` dan membuktikan lokasinya di `.rodata`.

```bash
objdump -d -M intel pola_O0 | sed -n '/<pick>:/,/ret/p'
readelf -SW pola_O0 | grep rodata
```

Ambil alamat tabel dari `lea rdx,[rip+…]`, lalu lihat isinya:

```bash
xxd -s <offset_tabel> -l 32 pola_O0
```

> [!NOTE]
> Ubah alamat menjadi offset file dengan rumus PART 3 (`alamat - Addr + Off`) sebelum memakai
> `xxd`. Entri 4 byte di sana adalah **offset relatif**, bukan alamat langsung — jumlahkan dengan
> alamat tabel untuk mendapatkan tujuan tiap `case`.

> [!IMPORTANT]
> **THE AHA:** `switch` bukan rantai `if`, tapi **tabel data yang dipakai sebagai tujuan
> loncatan**. Struktur kontrol ternyata disimpan sebagai data — dan data itu bisa kamu baca.

### 8.5 Lab 4 — Bandingkan optimasi dan lihat canary

**Tujuan:** melihat compiler mengubah bentuk, dan mengenali pola proteksi.

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
> **THE AHA:** Dua binary dari source yang sama bisa terlihat sangat berbeda. Yang kamu kenali
> adalah **niat strukturnya**, bukan urutan instruksinya.

### 8.6 Lab 5 — Konfirmasi visual di Ghidra

**Tujuan:** memakai alat untuk memverifikasi pembacaan manualmu.

Impor `pola_O0` ke Ghidra (PART 4), buka `pick`, lalu nyalakan **Function Graph**. Blok-blok dan
panah yang muncul harus cocok dengan pemetaan manualmu dari Lab 3.

> [!TIP]
> Alur kerja yang efisien: baca pola manual dulu → verifikasi di graph → beri nama blok/fungsi
> sesuai perannya. Graph mempercepat, tapi tidak menggantikan pemahaman pola.

## 9. Rekap

| Struktur C | Penanda assembly |
| :--- | :--- |
| `if … else` | `cmp` + `jcc` ke depan (kondisi dibalik) + `jmp` penutup |
| `if` tanpa else | `cmp` + `jcc` ke depan, tanpa `jmp` |
| `for` / `while` | inisialisasi, batas, langkah, `jcc` ke belakang |
| `do … while` | badan dulu, tanpa lompatan awal |
| `switch` rapat | cek rentang + akses tabel + `jmp register` |
| `switch` jarang | rantai `cmp`/`je` |
| ternary pendek (`-O2`) | `cmov`, tanpa `jcc` |
| tail call | `jmp` di ekor fungsi |
| stack protector | `fs:0x28` + `__stack_chk_fail` |
| variadic call | `mov eax,0` sebelum `printf` |

### Cek dirimu

- [ ] Kenapa `if (x > 5)` bisa muncul sebagai `jle`?
- [ ] Bagaimana membedakan loop dari `if` hanya dari arah loncatan?
- [ ] Apa ciri `do-while` di `-O0`, dan kenapa `-O2` mengaburkan perbedaan itu?
- [ ] Di section mana jump table tinggal, dan apa isi entrinya?
- [ ] Apa itu `cmov`, dan kenapa compiler memakainya?
- [ ] Kenapa ada `mov eax,0` sebelum `call printf`?
- [ ] Pola apa yang menandakan stack canary aktif?
- [ ] Kenapa tail call bisa membuat backtrace kehilangan satu tingkat?

## 10. Cliffhanger: dari pola ke target nyata

Sekarang kamu punya semua fondasi: arsitektur, instruksi, format file, analisis statis, analisis
dinamis, dan pengenalan pola. Belum pernah menerapkannya pada program yang **dirancang untuk
melawan**.

Crackme adalah latihan berikutnya: program kecil dengan pemeriksaan lisensi, logika yang
sengaja dibelokkan, dan kadang jebakan untuk analis.

> [!IMPORTANT]
> Itulah **Fase 7 — CrackMes & Challenges**: memakai semua pola ini untuk membuat keygen dan
> mem-patch target latihan yang nyata.

## 11. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| Reversing pattern | bentuk assembly berulang yang mewakili struktur kode tertentu |
| Basic block | rangkaian instruksi lurus tanpa loncatan di tengahnya |
| CFG (Control Flow Graph) | diagram blok + panah yang menggambarkan alur program |
| Kondisi terbalik | compiler membalik syarat agar jalur lurus jadi kasus umum |
| `jcc` | loncat bersyarat (`je`, `jne`, `jle`, …) |
| Loncat ke belakang | `jcc` menuju alamat lebih kecil; penanda loop |
| Loop guard | pemeriksaan di depan loop untuk melewati iterasi nol |
| Loop inversion | mengubah `for`/`while` menjadi bentuk `do-while` saat optimasi |
| Jump table | tabel tujuan loncatan untuk `switch`, biasanya di `.rodata` |
| Indirect jump | `jmp register` — tujuan ditentukan saat runtime |
| `movsxd` | memuat 4 byte bertanda lalu memperlebar ke 8 byte |
| `cmov` | conditional move: menyalin nilai tanpa melompat |
| Branchless | kode tanpa percabangan, memakai `cmov` |
| Strength reduction | mengganti operasi mahal dengan yang murah (`imul` → `lea`) |
| Constant folding | menghitung konstanta saat kompilasi |
| Inlining | menyisipkan isi fungsi ke pemanggilnya |
| Tail call | `jmp` ke fungsi lain di ekor fungsi; frame dipakai ulang |
| Frame pointer omission | tidak memakai `RBP`; lokal diakses lewat `RSP` |
| Leaf function | fungsi yang tidak memanggil fungsi lain |
| Variadic function | fungsi dengan argumen berubah-ubah (`printf`) |
| `rep stos` / `rep movs` | instruksi pengisian/penyalinan blok memori |
| Stack canary | nilai penjaga stack; pola `fs:0x28` + `__stack_chk_fail` |
| Boilerplate | instruksi pendukung yang bukan inti logika |

---

[← PART 5](../PART5/note.id.md) · [Peta belajar](../INDEX.id.md) · **Selanjutnya:** PART 7 —
CrackMe & Tantangan (*rencana*)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>