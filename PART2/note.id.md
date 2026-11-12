# PART 2 — Bahasa Assembly (Assembly Language)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 2 dari 10**

> [!NOTE]
> **Sumber:** [PART 2A di YouTube](https://www.youtube.com/watch?v=HkMetTR67XU) ·
> [video seri PART 3](https://www.youtube.com/watch?v=NefasNimlIk) (membahas `cmp`/`test`/jump
> dan patching — materinya ada di part ini)
> Istilah teknis tetap ditulis dalam bahasa aslinya, tetapi selalu dijelaskan dengan bahasa
> sederhana. Kamus istilah ada di [bagian 11](#11-kamus-istilah).

**Prasyarat:** [PART 1](../PART1/note.id.md) selesai — kamu sudah paham register, flags, stack
frame, dan bisa memakai `gdb` (`break`, `si`, `info registers`, `x/`).

**Setelah part ini kamu bisa:** membaca baris assembly x86-64 tanpa menebak, membedakan
sintaks Intel dan AT&T, mengenali instruksi yang paling sering muncul, menjelaskan bagaimana
`cmp`/`test` berubah menjadi keputusan, dan **mem-patch instruksi loncat** di dalam binary
milikmu sendiri.

## Daftar isi

1. [Kenapa assembly?](#1-kenapa-assembly)
2. [Anatomi instruksi](#2-anatomi-instruksi)
3. [Membaca dua gaya sintaks](#3-membaca-dua-gaya-sintaks)
4. [Tiga jenis operand](#4-tiga-jenis-operand)
5. [Transfer data: MOV dan LEA](#5-transfer-data-mov-dan-lea)
6. [Aritmatika, logika, dan flags](#6-aritmatika-logika-dan-flags)
7. [Kendali alur: loncat bersyarat](#7-kendali-alur-loncat-bersyarat)
8. [Praktik: baca dan patch](#8-praktik-baca-dan-patch)
9. [Rekap](#9-rekap)
10. [Cliffhanger: dari instruksi ke file](#10-cliffhanger-dari-instruksi-ke-file)
11. [Kamus istilah](#11-kamus-istilah)

---

## 1. Kenapa assembly?

Di PART 1 kamu sudah bisa mengubah **data** saat program berjalan (`set $eax = 0`) dan melihat
`vault` menerima password salah. Tapi perubahan itu hilang begitu program ditutup. Sekarang
kita ubah **instruksinya**, permanen di dalam file.

Targetnya tetap dua baris ini:

```asm
test    eax, eax
jne     .denied
```

> [!TIP]
> Analogi PART 1 dilanjutkan: kalau register adalah **kata benda** (benda yang dimanipulasi),
> maka instruksi assembly adalah **kata kerja** (tindakan yang dilakukan). Membaca disassembly
> = membaca kalimat pendek: "pindahkan ini ke sana", "tambahkan ini", "kalau nol, loncat".

**Assembly** adalah bentuk yang bisa dibaca manusia dari **machine code** (byte angka yang
dijalankan CPU). Satu baris assembly ≈ satu instruksi CPU. Yang mengubah byte menjadi teks ini
adalah **disassembler** (`objdump`, `gdb`, Ghidra).

## 2. Anatomi instruksi

Sebagian besar instruksi x86-64 punya bentuk yang sama:

```text
mnemonic    destination, source
   │             │         │
   │             │         └── dari mana nilainya
   │             └──────────── ke mana hasilnya disimpan
   └────────────────────────── perintah apa yang dijalankan

contoh:
mov         rax,        5        ; rax = 5
add         rax,        rdx      ; rax = rax + rdx
```

Tiga istilah yang sering tertukar:

| Istilah | Apa itu | Contoh |
| :--- | :--- | :--- |
| **Mnemonic** | nama teks perintahnya, untuk manusia | `mov`, `add`, `jne` |
| **Operand** | nilai/lokasi yang dioperasikan | `rax`, `5`, `[rbp-0x8]` |
| **Opcode** | **angka** yang mewakili instruksi itu di dalam file | `75` = `jne`, `74` = `je`, `90` = `nop` |

> [!IMPORTANT]
> Mnemonic ≠ opcode. `jne` adalah tulisan untuk kita; byte `75` adalah yang benar-benar ada di
> dalam binary. Karena itu mem-patch instruksi = mengganti **byte**, bukan mengganti teks.
> (Istilah ini dipakai konsisten seperti di PART 0.)

Tidak semua instruksi punya dua operand: `ret`, `nop`, `leave` tidak punya operand sama sekali,
dan `inc rax` hanya punya satu.

## 3. Membaca dua gaya sintaks

Instruksi yang sama bisa ditulis dengan dua gaya. Isinya identik, urutannya saja yang berbeda:

| Fitur | Intel syntax (dipakai di catatan ini) | AT&T syntax (default `objdump` & `gdb`) |
| :--- | :--- | :--- |
| Urutan operand | `destination, source` | `source, destination` |
| Register | `rax` | `%rax` |
| Nilai langsung | `5` | `$5` |
| Memori | `[rbp-0x8]` | `-0x8(%rbp)` |
| Ukuran operasi | `DWORD PTR` | akhiran huruf: `movl`, `movq` |
| Contoh | `mov rax, 5` | `mov $5, %rax` |

Cara membacanya: **Intel = "simpan ke kiri, ambil dari kanan"**, **AT&T = kebalikannya**.

Setel tools agar memakai Intel:

```gdb
set disassembly-flavor intel
```

```bash
objdump -d -M intel vault
```

> [!NOTE]
> Perhatikan tanda hubung pada `disassembly-flavor` (bukan `disassembly flavor`). Mau permanen?
> Tambahkan baris gdb itu ke `~/.gdbinit` (opsional).

> [!TIP]
> Jangan dihafal mati dua-duanya. Pilih Intel untuk belajar, tapi **kenali** AT&T — kamu akan
> menemuinya saat lupa memakai flag `-M intel`, atau saat membaca tutorial lama.

## 4. Tiga jenis operand

| Jenis | Apa itu | Contoh | Analogi |
| :--- | :--- | :--- | :--- |
| **Immediate** | angka konstan yang tertulis di instruksi | `5`, `0x25` | angka yang kamu sebut langsung |
| **Register** | tempat penyimpanan di dalam CPU | `rax`, `edi`, `r8` | isi meja kerja |
| **Memory** | isi memori pada suatu alamat, selalu pakai `[...]` | `[rbp-0x8]`, `[rax]` | isi loker bernomor |

> [!IMPORTANT]
> **Aturan emas:** setiap kali melihat `[...]`, bacalah "**nilai yang tersimpan di alamat ini**".
> `rax` = alamatnya (kalau dipakai sebagai pointer), `[rax]` = isinya. Sama seperti `rsp` vs
> `[rsp]` di PART 1.

### Ukuran operand harus jelas

Kalau ukurannya tidak bisa ditebak dari register, disassembler menuliskannya eksplisit:

| Penulisan | Ukuran | Padanan C |
| :--- | :--- | :--- |
| `BYTE PTR` | 1 byte | `char` |
| `WORD PTR` | 2 byte | `short` |
| `DWORD PTR` | 4 byte | `int` |
| `QWORD PTR` | 8 byte | `long`, pointer |

```asm
mov     eax, DWORD PTR [rbp-0x4]   ; baca 4 byte (sebuah int)
mov     rax, QWORD PTR [rbp-0x8]   ; baca 8 byte (sebuah pointer)
mov     DWORD PTR [rbp-0x4], 0x0   ; tulis 4 byte angka 0
```

> [!NOTE]
> Dua aturan yang sering bikin pemula salah baca:
> - `mov` **tidak bisa** memori → memori. Selalu ada register di salah satu sisi
>   (`mov [a], [b]` tidak valid; harus lewat register dulu).
> - Menulis ke register 32-bit (`eax`) **mengosongkan** 32 bit atas `RAX` (sudah dibahas di
>   PART 1). Jadi `mov eax, 5` membuat `RAX = 5`, bukan cuma separuh bawahnya.

## 5. Transfer data: MOV dan LEA

### MOV — menyalin nilai

`mov` **menyalin** (bukan memindahkan): sumbernya tidak berubah. Empat pola yang ada:

```asm
mov     rax, 5           ; immediate → register
mov     rax, rdx         ; register  → register
mov     rax, [rbp-0x8]   ; memory    → register (baca isi memori)
mov     [rbp-0x8], rax   ; register  → memory   (tulis ke memori)
```

### LEA — mengambil alamat, bukan isi

`lea` = *Load Effective Address*. Bentuknya mirip `mov` dengan `[...]`, tetapi **tidak membaca
memori**: yang disimpan adalah hasil perhitungan alamatnya.

```asm
mov     rax, [rbp-0x8]   ; rax = ISI memori di alamat rbp-8
lea     rax, [rbp-0x8]   ; rax = ALAMAT rbp-8 itu sendiri ("address of")
```

> [!TIP]
> Analogi: `mov [alamat]` = "ambil isi lokernya". `lea [alamat]` = "catat nomor lokernya".

### LEA juga dipakai sebagai kalkulator

Karena `lea` bisa menghitung `[base + index*skala + offset]` dalam satu instruksi, compiler
memakainya untuk aritmatika cepat — bukan untuk alamat sama sekali:

```asm
lea     eax, [rdi+rsi]        ; eax = rdi + rsi        ← contoh di PART 0
lea     eax, [rdi+rdi*2]      ; eax = rdi * 3
lea     eax, [rax+rax*4]      ; eax = rax * 5
```

> [!IMPORTANT]
> Kalau kamu melihat `lea` dengan pola aneh seperti `[rax+rax*2]`, jangan dibaca sebagai
> akses memori. Itu **perkalian yang disamarkan**. Mengenali trik seperti ini adalah inti
> Fase 6 (Reversing Patterns).

Catatan penting: `mov` dan `lea` **tidak mengubah flags**.

## 6. Aritmatika, logika, dan flags

### 6.1 Aritmatika dasar

```asm
add     rax, rdx     ; rax = rax + rdx
sub     rax, rdx     ; rax = rax - rdx
inc     rax          ; rax = rax + 1
dec     rax          ; rax = rax - 1
imul    rax, rdx     ; rax = rax * rdx (perkalian signed)
neg     rax          ; rax = -rax
```

### 6.2 Logika dasar

```asm
and     rax, rdx     ; AND bit per bit
or      rax, rdx     ; OR bit per bit
xor     rax, rdx     ; XOR bit per bit
not     rax          ; balik semua bit
shl     rax, 1       ; geser bit ke kiri 1  = kali 2
shr     rax, 1       ; geser bit ke kanan 1 = bagi 2 (unsigned)
```

### 6.3 Efek samping: flags ikut berubah

Instruksi aritmatika/logika memperbarui `RFLAGS` (PART 1 §3.3) sebagai efek samping:

| Flag | Menyala jika |
| :---: | :--- |
| `ZF` | hasilnya nol |
| `SF` | hasilnya negatif |
| `CF` | ada carry/borrow (unsigned) |
| `OF` | hasilnya melampaui batas signed |

> [!NOTE]
> Jebakan yang perlu diingat:
> - `add`/`sub` mengubah semua flag di atas, tetapi **`inc`/`dec` tidak mengubah `CF`**.
> - `mov`/`lea` **tidak** mengubah flag apa pun.
> - `lea` tidak membaca memori, jadi tidak bisa menyebabkan *segmentation fault*.

### 6.4 Trik compiler: mengosongkan register dengan XOR

Untuk membuat register bernilai 0, compiler biasanya **tidak** memakai `mov eax, 0`, melainkan:

```asm
xor     eax, eax     ; eax = eax XOR eax = 0
```

Alasannya:

| Instruksi | Byte | Ukuran |
| :--- | :--- | :--- |
| `xor eax, eax` | `31 c0` | 2 byte |
| `mov eax, 0` | `b8 00 00 00 00` | 5 byte |

XOR terhadap nilai yang sama selalu nol. Selain lebih pendek, CPU modern mengenali pola ini
sebagai *zeroing idiom* dan memprosesnya lebih efisien. Efek sampingnya: `ZF` menyala, `CF` dan
`OF` bersih.

> [!IMPORTANT]
> Saat membaca disassembly, `xor eax, eax` **bukan** operasi logika yang perlu kamu pikirkan.
> Bacalah langsung sebagai "**eax = 0**".

## 7. Kendali alur: loncat bersyarat

### 7.1 CMP dan TEST: menghitung tanpa menyimpan

Dua instruksi ini bekerja seperti `sub` dan `and`, tetapi **hasilnya dibuang** — tujuannya hanya
menyalakan flag:

```asm
cmp     eax, 5       ; hitung eax - 5, buang hasilnya, set flags
test    eax, eax     ; hitung eax AND eax, buang hasilnya, set flags
```

| Instruksi | Operasi tersembunyi | Pertanyaan yang dijawab |
| :--- | :--- | :--- |
| `cmp a, b` | `a - b` | apakah `a == b`? lebih besar? lebih kecil? |
| `test a, a` | `a & a` | apakah `a == 0`? |

### 7.2 Instruksi loncat membaca flag

Setelah flag terpasang, instruksi loncat (*conditional jump*) mengambil keputusan:

| Instruksi | Loncat jika | Dipakai untuk | Flag yang dibaca |
| :--- | :--- | :--- | :--- |
| `je` / `jz` | sama / nol | `==` | `ZF = 1` |
| `jne` / `jnz` | tidak sama / bukan nol | `!=` | `ZF = 0` |
| `ja` / `jb` | di atas / di bawah | `>` `<` **unsigned** | `CF`, `ZF` |
| `jae` / `jbe` | ≥ / ≤ unsigned | `>=` `<=` **unsigned** | `CF`, `ZF` |
| `jg` / `jl` | lebih besar / kecil | `>` `<` **signed** | `SF`, `OF`, `ZF` |
| `jge` / `jle` | ≥ / ≤ signed | `>=` `<=` **signed** | `SF`, `OF`, `ZF` |
| `jmp` | selalu | loncat tanpa syarat | — |

> [!TIP]
> Cara cepat mengenali niat kode: `cmp` + `jg`/`jl` → angka **signed** (mis. `int`).
> `cmp` + `ja`/`jb` → angka **unsigned** (mis. `size_t`, panjang buffer). Ini petunjuk tipe data
> yang sudah dibuang compiler.

### 7.3 Rantai lengkap keputusan vault

```asm
call    strcmp       ; hasilnya di EAX (0 = string sama)
test    eax, eax     ; EAX == 0 ? → menyalakan ZF
jne     .denied      ; ZF = 0 (tidak sama) → loncat ke DENIED
```

**nilai → flag → keputusan.** Ubah salah satu mata rantai, perilaku program berubah:

| Yang diubah | Cara | Hasil |
| :--- | :--- | :--- |
| nilai | `set $eax = 0` di gdb (PART 1) | sementara, hilang saat program tutup |
| flag | `set $eflags = $eflags \| 0x40` | sementara |
| instruksi | patch byte `75` → `74` | **permanen di file** ← part ini |

### 7.4 Opcode loncat: yang perlu dikenali saat patching

Loncat pendek (*short jump*, jarak ±127 byte) berumur 2 byte: opcode + jarak. Loncat jauh
(*near jump*) berumur 6 byte dan diawali `0f`:

| Mnemonic | Short (2 byte) | Near (6 byte) |
| :--- | :--- | :--- |
| `je` | `74 xx` | `0f 84 xx xx xx xx` |
| `jne` | `75 xx` | `0f 85 xx xx xx xx` |
| `jb` / `jae` | `72` / `73` | `0f 82` / `0f 83` |
| `jbe` / `ja` | `76` / `77` | `0f 86` / `0f 87` |
| `jl` / `jge` | `7c` / `7d` | `0f 8c` / `0f 8d` |
| `jle` / `jg` | `7e` / `7f` | `0f 8e` / `0f 8f` |
| `jmp` | `eb xx` | `e9 xx xx xx xx` |
| `nop` | `90` (1 byte) | `66 90` (2 byte) |

> [!IMPORTANT]
> **Aturan patching:** ganti instruksi dengan instruksi **berukuran sama**. `75` → `74` aman
> (1 byte → 1 byte). Kalau kamu menemukan `0f 85`, pasangannya `0f 84`, dan NOP-nya harus
> **6 byte**, bukan 2. Menulis lebih pendek tanpa mengisi sisanya akan menggeser arti byte
> berikutnya dan merusak program.

## 8. Praktik: baca dan patch

### 8.1 Siapkan target

Pakai dua file yang sudah kamu punya: `frame.c` (PART 1) dan `vault.c` (PART 0).

```bash
gcc -g -O0 frame.c -o frame
gcc -g -O0 vault.c -o vault
cp vault vault.bak
```

> [!NOTE]
> Blok `bash` = ketik di **terminal**. Blok `gdb` = ketik **di dalam gdb** (prompt `(gdb)` tidak
> ditulis supaya bisa di-copy-paste). Blok `text` = output, `asm` = disassembly.

> [!WARNING]
> Lab 4 dan 5 **mengubah file `vault`**. Kembalikan kapan saja dengan `gcc -g -O0 vault.c -o vault`
> atau `cp vault.bak vault`. Hanya patch binary milikmu sendiri atau yang kamu punya izin untuk
> dianalisis.

### 8.2 Lab 1 — Baca fungsi `add` baris demi baris

**Tujuan:** membuktikan bahwa kamu bisa membaca fungsi utuh, bukan menebak.

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

Cara membacanya baris per baris:

| Baris | Dibaca sebagai |
| :--- | :--- |
| `push rbp` / `mov rbp,rsp` | prologue: pasang patok frame (PART 1) — **boleh dilewati** saat pertama membaca |
| `mov DWORD PTR [rbp-0x14],edi` | simpan argumen ke-1 (`a`) ke variabel lokal 4 byte |
| `mov DWORD PTR [rbp-0x18],esi` | simpan argumen ke-2 (`b`) |
| `mov edx,[rbp-0x14]` / `mov eax,[rbp-0x18]` | muat ulang keduanya ke register untuk dihitung |
| `add eax,edx` | **ini inti fungsinya**: `eax = b + a` |
| `mov [rbp-0x4],eax` | simpan hasilnya ke variabel `sum` |
| `mov eax,[rbp-0x4]` | taruh `sum` di `EAX` = nilai kembalian (ABI) |
| `pop rbp` / `ret` | epilogue: bereskan frame, pulang |

> [!IMPORTANT]
> **THE AHA:** Dari 11 baris, hanya **1 baris** yang benar-benar "bekerja" (`add eax,edx`).
> Sisanya prologue, epilogue, dan bolak-balik register↔stack karena kita memakai `-O0`.
> Membaca assembly = menemukan baris inti di antara boilerplate.

### 8.3 Lab 2 — Instruksi yang sama, dua bahasa

**Tujuan:** tidak panik saat gặp sintaks AT&T.

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

Bandingkan satu baris saja:

| Intel | AT&T | Artinya |
| :--- | :--- | :--- |
| `mov DWORD PTR [rbp-0x14],edi` | `mov %edi,-0x14(%rbp)` | simpan `edi` ke memori |

> [!NOTE]
> Sebagian versi binutils menambahkan akhiran ukuran (`movl`, `addl`, `retq`). Artinya sama;
> huruf `l` = 4 byte, `q` = 8 byte.

> [!IMPORTANT]
> **THE AHA:** Instruksinya identik, byte-nya identik, hanya **cara menuliskannya** yang berbeda.
> Bingung sintaks ≠ bingung logika. Setel Intel sekali, lalu fokus ke logikanya.

### 8.4 Lab 3 — Kenapa compiler memilih `xor`

**Tujuan:** melihat *zeroing idiom* muncul di kode nyata.

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

Cek byte-nya sendiri (bagian `31 c0` di kolom kiri):

```bash
objdump -d zero.o
```

> [!IMPORTANT]
> **THE AHA:** Compiler tidak menulis "return 0" apa adanya. Ia memilih instruksi paling murah
> yang menghasilkan efek sama. Kalau kamu membaca `xor reg, reg` sebagai operasi logika, kamu
> akan salah paham — bacalah "**reg = 0**".

### 8.5 Lab 4 — Patch keputusan `vault`: `75` menjadi `74`

**Tujuan:** membalik logika password secara permanen di dalam file, tanpa kompilasi ulang.

**Langkah 1 — temukan instruksi dan byte-nya:**

```bash
objdump -d -M intel vault | grep -A3 'strcmp@plt'
```

```text
    11af:	e8 9c fe ff ff       	call   1050 <strcmp@plt>
    11b4:	85 c0                	test   eax,eax
    11b6:	75 0e                	jne    11c6 <main+0x61>
    11b8:	48 8d 3d 71 0e 00 00 	lea    rdi,[rip+0xe71]
```

Yang kita incar: `75 0e` di alamat `11b6`. `75` = `jne`, `0e` = jarak loncat (14 byte ke depan).

**Langkah 2 — ubah alamat menjadi offset file:**

```bash
readelf -SW vault | grep '\.text'
```

```text
  [14] .text  PROGBITS  0000000000001060 001060 0001a2 00  AX  0   0  16
                          ↑ Addr              ↑ Off
```

Rumusnya: `offset file = alamat - Addr + Off`. Pada binary kecil seperti ini `Addr` dan `Off`
biasanya sama, jadi offset file = `0x11b6`. **Jangan berasumsi — buktikan:**

**Langkah 3 — verifikasi sebelum menulis:**

```bash
xxd -s 0x11b6 -l 2 vault
```

```text
000011b6: 750e                                     u.
```

Kalau outputnya `750e`, posisi kita benar. Kalau bukan, hitung ulang Langkah 2.

**Langkah 4 — patch 1 byte:**

```bash
printf '\164' | dd of=vault bs=1 seek=$((0x11b6)) count=1 conv=notrunc
```

`\164` adalah oktal untuk `0x74` (`je`) — sama seperti trik `\000` di PART 0, oktal dipakai agar
jalan di semua shell. `$((0x11b6))` mengubah hex menjadi desimal karena `dd` tidak menerima hex.

**Langkah 5 — buktikan:**

```bash
objdump -d -M intel vault | grep -A1 'test.*eax,eax'
./vault
```

```text
    11b6:	74 0e                	je     11c6 <main+0x61>
Enter the vault password: tebak
ACCESS GRANTED
```

Coba juga password yang **benar** (`sup3rs3cr3t`) → sekarang malah `ACCESS DENIED`.

> [!IMPORTANT]
> **THE AHA:** Satu byte, `75` → `74`, membalik seluruh logika autentikasi. Ini "cara ke-4" yang
> dijanjikan PART 0: bukan mengubah data, bukan mengubah register, tapi mengubah **keputusan
> program itu sendiri**.

> [!NOTE]
> Kalau `objdump` di mesinmu menampilkan `0f 85 ...` (near jump, 6 byte), pasangannya `0f 84`
> — patch **2 byte** pertama, bukan 1. Alamat juga bisa berbeda antar versi gcc; selalu mulai
> dari Langkah 1, jangan menyalin angka `0x11b6` mentah-mentah.

### 8.6 Lab 5 — Hapus loncatan dengan NOP

**Tujuan:** membuat password **apa pun** diterima, bukan sekadar membalik logika.

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
Enter the vault password: apa-saja
ACCESS GRANTED
```

`\220` = oktal untuk `0x90` (`nop`). Dua byte `75 0e` ditimpa dua `nop`, jadi tidak ada loncatan
yang tersisa: eksekusi selalu jatuh ke cabang `ACCESS GRANTED`.

> [!IMPORTANT]
> **THE AHA:** `je` **membalik** keputusan; `nop` **menghapus** keputusan. Ukurannya harus sama
> persis (2 byte → 2 byte) supaya instruksi setelahnya tidak bergeser.这就是 alasan patching
> selalu soal hitungan byte.

### 8.7 Variasi runtime: breakpoint di offset dan flip flag

**Tujuan:** melihat keputusan yang sama dari sisi dinamis, tanpa menyentuh file.

Selain `break strcmp` + `finish` (PART 1), kamu bisa berhenti langsung di offset instruksi
di dalam fungsi. Ambil offsetnya dari `objdump`: `jne` di PART 2 berada di `main+0x51`
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

`0x40` adalah bit `ZF`. Sintaks `set $eflags |= 0x40` dan `set $eflags = $eflags | 0x40`
sama artinya.

> [!NOTE]
> Offset `main+0x51` berlaku untuk binary yang dikompilasi seperti contoh. Selalu hitung ulang
> dari `objdump` (`alamat instruksi − alamat main`), jangan disalin mentah.

> [!IMPORTANT]
> **THE AHA:** Instruksi `jne` yang sama bisa dikalahkan dari dua sisi: **flag di memori**
> (runtime, sementara) atau **byte di file** (permanen). Bedanya cuma di mana kamu menulis.

## 9. Rekap

Urutan membaca disassembly yang bisa kamu pakai mulai sekarang:

1. **Lewati prologue** (`push rbp`, `mov rbp,rsp`, `sub rsp,N`) — itu persiapan frame.
2. **Cari `call`** — di situlah program berinteraksi (`printf`, `strcmp`, `malloc`).
3. **Cari `cmp`/`test` + loncat bersyarat** — di situlah keputusan dibuat.
4. **Ikuti arah loncat** — mana jalur "berhasil", mana jalur "gagal".
5. **Lihat `eax` sebelum `ret`** — itu nilai kembaliannya.

| Konsep | Inti yang harus kamu ingat |
| :--- | :--- |
| Mnemonic vs opcode | `jne` = teks untuk manusia, `75` = byte di file |
| Sintaks | Intel `dst, src`; AT&T `src, dst` dengan `%` dan `$` |
| Operand | immediate (angka), register, memori `[...]` = "isi alamat ini" |
| `mov` vs `lea` | `mov [x]` ambil isi; `lea [x]` ambil alamat (atau aritmatika terselubung) |
| Flags | `add`/`sub`/`and`/`xor` mengubahnya; `mov`/`lea` tidak; `inc`/`dec` tidak menyentuh `CF` |
| `xor reg, reg` | baca sebagai "reg = 0", bukan operasi logika |
| Keputusan | `cmp`/`test` → flags → `jcc`; rantai nilai → flag → keputusan |
| Patching | ganti byte berukuran sama: `75`→`74` membalik, `90 90` menghapus |

### Cek dirimu

Kamu siap lanjut ke Fase 3 jika bisa menjawab:

- [ ] Apa beda mnemonic, operand, dan opcode?
- [ ] `mov rax, [rbp-8]` vs `lea rax, [rbp-8]` — mana yang membaca memori?
- [ ] Kenapa `lea eax, [rdi+rdi*2]` sebenarnya adalah perkalian?
- [ ] Apa yang dilakukan `test eax,eax`, dan flag apa yang diubahnya?
- [ ] `cmp` + `jg` vs `cmp` + `ja` — apa petunjuk tipe data yang bisa kamu ambil?
- [ ] Kenapa `xor eax,eax` lebih disukai compiler daripada `mov eax,0`?
- [ ] Kenapa patch harus berukuran byte sama, dan berapa byte NOP untuk `0f 85`?
- [ ] Bagaimana kamu mengubah alamat dari `objdump` menjadi offset file untuk `dd`?

## 10. Cliffhanger: dari instruksi ke file

Lab 4 punya satu langkah yang masih terasa seperti sulap: **mengubah alamat `0x11b6` menjadi
offset file**. Kita memakai rumus `alamat - Addr + Off` dari `readelf`, dan kebetulan keduanya
sama.

Pertanyaan yang belum terjawab:

- Kenapa alamat memori dan posisi di file **bisa berbeda**?
- Dari mana `objdump` tahu bahwa fungsi itu bernama `add`?
- Di mana sebenarnya string `sup3rs3cr3t` disimpan, dan kenapa `strings` bisa menemukannya?
- Kenapa byte pertama file harus `7f 45 4c 46` (PART 0), dan apa isi 60 byte sesudahnya?

Semua jawabannya ada di **kepala file** yang selama ini kita lewati.

> [!IMPORTANT]
> Itulah **Fase 3 — Executable Formats**: membaca header ELF, section, dan segment, supaya
> patching berhenti menjadi tebakan dan menjadi perhitungan.

## 11. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| Assembly | bahasa tingkat rendah yang hampir 1:1 dengan instruksi CPU |
| Disassembly | teks assembly hasil bongkaran machine code |
| Disassembler | tool yang membongkar byte menjadi assembly (`objdump`, Ghidra) |
| Mnemonic | nama teks instruksi untuk manusia (`mov`, `add`, `jne`) |
| Operand | nilai atau lokasi yang dioperasikan instruksi |
| Opcode | **angka** yang mewakili instruksi di dalam file (`75` = `jne`) |
| Destination / Source | operand tujuan hasil / operand asal nilai |
| Intel syntax | gaya penulisan `tujuan, sumber`, tanpa simbol tambahan |
| AT&T syntax | gaya penulisan `sumber, tujuan`, dengan `%` (register) dan `$` (angka) |
| Immediate | angka konstan yang tertulis langsung di instruksi (`5`, `0x25`) |
| Register operand | operand yang berupa register CPU (`rax`, `edi`) |
| Memory operand | operand `[...]` = "isi memori pada alamat ini" |
| BYTE / WORD / DWORD / QWORD PTR | penanda ukuran operand: 1 / 2 / 4 / 8 byte |
| MOV | menyalin nilai dari sumber ke tujuan (sumber tidak berubah) |
| LEA | mengambil **alamat**-nya, bukan isinya; sering dipakai untuk aritmatika |
| ADD / SUB | menambah / mengurangi nilai |
| INC / DEC | menambah 1 / mengurangi 1 (tidak mengubah `CF`) |
| IMUL / NEG | perkalian signed / mengubah tanda menjadi negatif |
| AND / OR / XOR / NOT | operasi logika bit per bit |
| SHL / SHR | geser bit kiri (kali 2) / kanan (bagi 2 unsigned) |
| Zeroing idiom | trik `xor reg, reg` untuk membuat register bernilai 0 |
| Flags | bit status hasil operasi terakhir (`ZF`, `SF`, `CF`, `OF`) |
| CMP | menghitung `a - b` lalu membuang hasilnya, hanya untuk set flags |
| TEST | menghitung `a & a` lalu membuang hasilnya, hanya untuk set flags |
| Jump bersyarat (`jcc`) | loncat hanya jika flag memenuhi syarat (`je`, `jne`, `jg`, ...) |
| JMP | loncat tanpa syarat |
| Short / near jump | loncat 2 byte (`75 xx`) / 6 byte (`0f 85 ...`) |
| Signed vs unsigned jump | `jg`/`jl` untuk bilangan bertanda, `ja`/`jb` untuk tanpa tanda |
| NOP | instruksi "tidak melakukan apa-apa" (`90`), untuk menghapus instruksi |
| Patch | mengubah byte di dalam binary untuk mengubah perilakunya |
| Boilerplate | instruksi pendukung (prologue/epilogue) yang bukan inti logika |
| Offset file | posisi byte dihitung dari awal file (beda dengan alamat memori) |

---

[← PART 1](../PART1/note.id.md) · [Peta belajar](../INDEX.id.md) · **Selanjutnya:** PART 3 —
Format Executable ([catatan](../PART3/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>