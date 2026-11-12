# PART 1 — Dasar Arsitektur (Architecture Fundamentals)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 1 dari 10**

> [!NOTE]
> **Sumber:** [PART 1 di YouTube](https://www.youtube.com/watch?v=eEcXSvd8PCM)
> Istilah teknis tetap ditulis dalam bahasa aslinya, tetapi selalu dijelaskan dengan bahasa
> sederhana. Kamus istilah ada di [bagian 9](#9-kamus-istilah).

**Prasyarat:** [PART 0](../PART0/note.id.md) selesai — lab sudah terpasang (`gcc`, `gdb`,
`binutils`) dan kamu sudah pernah memakai `strings` serta `ltrace`.

**Setelah part ini kamu bisa:** menjelaskan apa yang dilakukan CPU pada setiap instruksi,
membaca isi register, menunjukkan di mana kode/variabel/stack tinggal di memori, membongkar
anatomi stack frame, dan memakai `gdb` untuk melihat argumen fungsi, return address, serta
mengubah nilai register saat program berjalan.

## Daftar isi

1. [Kenapa harus paham arsitektur?](#1-kenapa-harus-paham-arsitektur)
2. [Siklus kerja CPU: fetch-decode-execute](#2-siklus-kerja-cpu-fetch-decode-execute)
3. [Register: meja kerja CPU](#3-register-meja-kerja-cpu)
4. [Peta memori proses](#4-peta-memori-proses)
5. [Stack dan stack frame](#5-stack-dan-stack-frame)
6. [Praktik: GDB](#6-praktik-gdb)
7. [Rekap](#7-rekap)
8. [Cliffhanger: dari register ke instruksi](#8-cliffhanger-dari-register-ke-instruksi)
9. [Kamus istilah](#9-kamus-istilah)

---

## 1. Kenapa harus paham arsitektur?

Di PART 0 kamu membongkar `vault` **tanpa** membaca assembly: rahasianya dibaca dari file,
disadap dari memori, lalu datanya ditambal. Cara ke-4 — mengubah **keputusannya** — tertahan
di satu baris ini:

```asm
test    eax, eax
jne     .denied
```

Untuk memahaminya, kamu perlu tahu tiga hal: apa itu `eax` (register), apa yang diuji `test`
(flags), dan dari mana fungsi mengambil argumennya (stack + aturan pemanggilan). Itulah isi
part ini.

> [!TIP]
> Analogi: kalau binary adalah sebuah pabrik, **CPU** adalah pekerjanya, **register** adalah
> meja kerjanya (kecil tapi paling cepat dijangkau), **RAM** adalah gudangnya (besar tapi
> lebih jauh), dan **stack** adalah tumpukan nampan pekerjaan yang sedang berjalan.

Reverse engineering di tingkat biner pada dasarnya adalah melacak tiga hal: instruksi mana
yang dijalankan CPU, data apa yang ada di register, dan bagaimana stack mengatur alur fungsi.

## 2. Siklus kerja CPU: fetch-decode-execute

CPU (Central Processing Unit) bekerja dalam satu siklus yang berulang terus, satu instruksi
demi satu instruksi:

| Tahap | Arti | Yang terjadi |
| :--- | :--- | :--- |
| **Fetch** | ambil | CPU mengambil instruksi berikutnya dari memori, dari alamat yang ditunjuk `RIP` |
| **Decode** | terjemahkan | CPU menerjemahkan angka instruksi (*opcode*) menjadi sinyal kontrol internal |
| **Execute** | jalankan | CPU melakukan operasinya (hitung, pindahkan data, atau loncat), lalu memperbarui register/memori |

```text
[ RAM / Text Segment ] ──fetch (alamat dari RIP)──▶ [ CPU Decoder ] ──▶ Execute
                                                                          │
                       ┌──────────────────────────────────────────────────┘
                       ▼
              Update register & memori  ──▶ RIP maju ──▶ fetch lagi (ulang)
```

> [!IMPORTANT]
> `RIP` adalah "jari penunjuk baris" CPU. Siapa yang mengendalikan `RIP`, dia mengendalikan
> program. Inilah sebabnya semua instruksi loncat (`jmp`, `jne`, `call`, `ret`) begitu penting
> dalam reverse engineering.

## 3. Register: meja kerja CPU

**Register** adalah tempat penyimpanan super cepat **di dalam** CPU. Di x86-64 lebarnya 64 bit
(8 byte). Jumlahnya sedikit, jadi compiler harus pintar memakainya.

### 3.1 General Purpose Registers (GPR)

Register serbaguna untuk menghitung dan memindahkan data:

| 64-bit | 32-bit | 16-bit | 8-bit | Peran utama |
| :--- | :--- | :--- | :--- | :--- |
| `RAX` | `EAX` | `AX` | `AL` | *Accumulator*: hasil perhitungan dan **nilai kembalian fungsi** |
| `RBX` | `EBX` | `BX` | `BL` | penyimpan data umum / pointer basis |
| `RCX` | `ECX` | `CX` | `CL` | *counter*: penghitung loop dan operasi string |
| `RDX` | `EDX` | `DX` | `DL` | data tambahan untuk perkalian/pembagian, I/O |
| `RSI` | `ESI` | `SI` | `SIL` | *source index*: pointer sumber saat menyalin memori/string |
| `RDI` | `EDI` | `DI` | `DIL` | *destination index*: pointer tujuan saat menyalin memori/string |
| `R8`–`R15` | `R8D`–`R15D` | `R8W`–`R15W` | `R8B`–`R15B` | register tambahan di x86-64 untuk kebutuhan umum |

Nama-nama itu bukan register berbeda, melainkan **potongan dari register yang sama**: `RAX`
(64 bit) → `EAX` (32 bit bawah) → `AX` (16 bit bawah) → `AL` (8 bit bawah).

> [!NOTE]
> Jebakan yang sering muncul saat membaca disassembly: menulis ke `EAX` **mengosongkan**
> 32 bit atas `RAX`, sedangkan menulis ke `AX`/`AL` **tidak** mengubah bit di atasnya. Jadi
> `mov eax, 0` benar-benar membuat `RAX = 0`, tetapi `mov al, 0` belum tentu.

### 3.2 Register khusus penunjuk memori

| Register | Nama | Fungsi | Analogi |
| :--- | :--- | :--- | :--- |
| `RIP` | Instruction Pointer | alamat instruksi **berikutnya** yang akan dijalankan | jari penunjuk baris yang sedang dibaca |
| `RSP` | Stack Pointer | alamat **puncak stack** saat ini | nampan paling atas |
| `RBP` | Base/Frame Pointer | alamat **dasar stack frame** fungsi yang aktif | patok/pembatas ruang kerja satu fungsi |

`RSP` **berkurang** saat `PUSH` dan **bertambah** saat `POP` (karena stack tumbuh ke alamat
rendah — lihat bagian 5).

### 3.3 RFLAGS: lampu indikator hasil operasi

`RFLAGS` bukan tempat menyimpan data, melainkan kumpulan bit status (*flag*) yang merekam
hasil operasi terakhir:

| Flag | Nama | Menyala (=1) jika |
| :---: | :--- | :--- |
| `ZF` | Zero Flag | hasilnya **nol** |
| `SF` | Sign Flag | hasilnya **negatif** (bit tertinggi = 1) |
| `CF` | Carry Flag | ada *carry*/*borrow* pada bilangan **unsigned** |
| `OF` | Overflow Flag | hasil melebihi batas bilangan **signed** |

Instruksi `cmp` dan `test` bekerja dengan cara menghitung lalu **membuang hasilnya**, hanya
untuk menyalakan flag. Instruksi loncat bersyarat kemudian membaca flag itu:

```asm
test    eax, eax     ; hitung eax AND eax → hasil 0 hanya jika eax == 0 → ZF menyala
jne     .denied      ; jump if not equal = loncat jika ZF = 0
```

> [!IMPORTANT]
> Ini jawaban cliffhanger PART 0: `strcmp` menaruh hasil di `EAX`, `test` mengubahnya menjadi
> `ZF`, dan `jne` mengambil keputusan berdasarkan `ZF`. Rantainya:
> **nilai → flag → keputusan**. Ubah salah satu, dan program berubah perilaku.

Pasangan loncat yang paling sering ditemui: `je`/`jne` membaca `ZF`; `jb`/`ja` membaca `CF`
(unsigned); `jl`/`jg` membaca `SF`+`OF`+`ZF` (signed).

### 3.4 Aturan pemanggilan fungsi (System V ABI)

**ABI** (Application Binary Interface) adalah kesepakatan baku tentang bagaimana fungsi
menerima argumen dan mengembalikan nilai. Di Linux x86-64 (System V ABI):

| Argumen ke- | 1 | 2 | 3 | 4 | 5 | 6 | 7 dan seterusnya |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Lewat | `RDI` | `RSI` | `RDX` | `RCX` | `R8` | `R9` | stack |

- Nilai kembalian ditaruh di `RAX`.
- Register `RBX`, `RBP`, `R12`–`R15` bersifat *callee-saved* (fungsi yang dipanggil wajib
  mengembalikan nilai aslinya); sisanya *caller-saved* (boleh dirusak oleh fungsi yang dipanggil).

> [!TIP]
> Ingat urutan argumen dengan jembatan keledai: **D**ua **S**api **D**uduk **C**antik **8** **9**
> → `RDI, RSI, RDX, RCX, R8, R9`. Di PART 0, `strcmp(input, secret)` berarti `RDI` = input dan
> `RSI` = secret. Itu sebabnya `x/s $rdi` bisa langsung menampilkan password yang kamu ketik.

## 4. Peta memori proses

Saat program dijalankan, OS memberinya **virtual address space** (ruang alamat seolah-olah
milik sendiri, padahal dipetakan OS ke RAM fisik). Ruang itu dibagi menjadi beberapa wilayah:

```text
+-----------------------------------+  alamat tinggi (0x7fff...)
|           Kernel space            |  wilayah OS, tidak bisa diakses program
+-----------------------------------+
|              Stack                |  ▼ tumbuh ke alamat rendah
|               ...                 |
|   (celah besar: mmap / library)   |  ← libc & file yang di-map tinggal di sini
|               ...                 |
|              Heap                 |  ▲ tumbuh ke alamat tinggi
+-----------------------------------+
|      BSS  (global tanpa nilai)    |
+-----------------------------------+
|      Data (global berisi nilai)   |
+-----------------------------------+
|      Text (kode program) r-x      |
+-----------------------------------+  alamat rendah
```

| Wilayah | Isi | Sifat | Analogi |
| :--- | :--- | :--- | :--- |
| **Text** (`.text`) | instruksi program | baca + jalankan, **tidak bisa ditulis** | buku resep (tidak boleh dicoret) |
| **Data** (`.data`) | variabel global/statis yang sudah punya nilai awal | baca + tulis | papan tulis yang sudah terisi |
| **BSS** (`.bss`) | variabel global/statis tanpa nilai awal (otomatis diisi nol) | baca + tulis | papan tulis kosong |
| **Heap** | alokasi dinamis: `malloc`, `calloc`, `new` | baca + tulis, tumbuh ke atas | gudang yang bisa disewa sesuai kebutuhan |
| **Stack** | argumen, variabel lokal, return address | baca + tulis, tumbuh ke bawah | tumpukan nampan kerja |

> [!NOTE]
> - Text bersifat *read-only* agar kode tidak bisa diubah saat berjalan. Karena itu, patch
>   seperti di PART 0 dilakukan pada **file**, bukan pada memori yang sedang berjalan.
> - Alamat di gambar hanya skema. Karena **ASLR** (dan **PIE** untuk binary modern), alamat
>   asli berubah setiap kali program dijalankan.
> - Di `gdb`, ASLR dimatikan secara default (`set disable-randomization on`), jadi alamat
>   terlihat sama terus **di dalam** gdb walaupun di luar berbeda.

## 5. Stack dan stack frame

### 5.1 Prinsip LIFO

**Stack** adalah struktur data **LIFO** (Last-In, First-Out): yang terakhir masuk, pertama
keluar — seperti tumpukan piring. Puncaknya selalu ditunjuk `RSP`. Di x86-64, satu slot = 8 byte.

| Operasi | Langkah | Efek pada `RSP` |
| :--- | :--- | :--- |
| `PUSH <nilai>` | `RSP = RSP - 8`, lalu tulis nilai ke `[RSP]` | berkurang 8 |
| `POP <register>` | baca `[RSP]` ke register, lalu `RSP = RSP + 8` | bertambah 8 |

Tanda `[...]` artinya **dereference**: "isi memori pada alamat itu", bukan alamatnya sendiri.
Jadi `rsp` = alamatnya, `[rsp]` = isinya.

### 5.2 Apa yang dilakukan CALL dan RET

`CALL <fungsi>` melakukan dua hal sekaligus:

1. Menyimpan **return address** (alamat instruksi tepat setelah `CALL`) ke puncak stack.
2. Melompat ke fungsi tujuan (`RIP = alamat fungsi`).

`RET` melakukan kebalikannya: mengambil return address dari stack dan menaruhnya ke `RIP`,
sehingga program kembali ke pemanggil.

> [!IMPORTANT]
> Return address hidup **di dalam stack** — data yang bisa ditulis. Kalau sebuah program bisa
> dipaksa menulis melewati batas buffer sampai menimpa return address, alur program bisa
> dibelokkan. Itulah akar **buffer overflow** (lab di Fase 5).

### 5.3 Prologue dan epilogue

Setiap fungsi membangun "ruang kerja" sendiri (*stack frame*) dengan pola standar:

```asm
; ---- prologue (pembuka) ----
push    rbp          ; simpan RBP milik pemanggil
mov     rbp, rsp     ; jadikan posisi ini sebagai patok frame baru
sub     rsp, 0x20    ; sediakan 32 byte untuk variabel lokal

; ---- badan fungsi: akses variabel lewat offset RBP ----
mov     DWORD PTR [rbp-0x4], eax    ; variabel lokal

; ---- epilogue (penutup) ----
mov     rsp, rbp     ; buang variabel lokal
pop     rbp          ; pulihkan RBP pemanggil
ret                  ; kembali ke return address
```

> [!NOTE]
> - Dua instruksi epilogue `mov rsp, rbp` + `pop rbp` sering disingkat menjadi satu instruksi
>   **`leave`** — itu yang kamu lihat di disassembly `vault` pada PART 0.
> - `sub rsp, N` biasanya berkelipatan 16: ABI mewajibkan stack **16-byte aligned** pada saat
>   `call`. Kalau tidak, fungsi library tertentu (mis. yang memakai instruksi SSE) bisa crash.
> - Kalau compiler memakai mode *omit frame pointer* (`-O2`), `RBP` bisa dipakai sebagai
>   register biasa dan variabel diakses lewat `RSP`. Jangan kaget kalau polanya hilang.

### 5.4 Anatomi stack frame

Contoh nyata dari lab di bagian 6 (`add(7, 5)`):

```text
        alamat tinggi
        +---------------------------+
        |   ... milik main ...      |
        +---------------------------+
        |   return address          |  ← [rbp+8]
        +---------------------------+
        |   rbp lama (saved rbp)    |  ← rbp   (patok frame add)
        +---------------------------+
        |   sum  (variabel lokal)   |  ← [rbp-0x4]
        +---------------------------+
        |   a    (salinan argumen)  |  ← [rbp-0x14]
        +---------------------------+
        |   b    (salinan argumen)  |  ← [rbp-0x18]  ← rsp
        +---------------------------+
        alamat rendah (stack tumbuh ke arah sini)
```

Alur lengkapnya: argumen masuk lewat register → `call` menaruh return address → prologue
memasang patok `RBP` → variabel lokal diakses sebagai `[rbp-offset]` → epilogue membereskan
frame → `ret` pulang.

## 6. Praktik: GDB

### 6.1 Siapkan target

Buat `frame.c`:

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

`-g` menyertakan **debug symbol** (nama fungsi, nama variabel, nomor baris) supaya gdb bisa
memanggil `add` dan `sum` dengan namanya. `-O0` mematikan optimasi agar kode tidak "dilebur"
seperti di Latihan 7 PART 0. `-q` hanya menghilangkan banner pembuka gdb.

### 6.2 Konvensi blok perintah

> [!NOTE]
> Blok berlabel `bash` = ketik di **terminal**. Blok berlabel `gdb` = ketik **di dalam gdb**
> (prompt `(gdb)` tidak ditulis supaya bisa langsung di-copy-paste). Blok `text` = output.

Wajib dilakukan pertama kali:

```gdb
set disassembly-flavor intel
```

Tanpa ini, gdb menampilkan sintaks **AT&T** (`mov %rsp,%rbp`) yang urutan operandnya terbalik
dari sintaks **Intel** (`mov rbp,rsp`) yang kita pakai sejak PART 0. Mau permanen? Tambahkan
baris itu ke `~/.gdbinit` (opsional).

### 6.3 Perintah dasar GDB

| Perintah | Singkat | Gunanya |
| :--- | :--- | :--- |
| `disassemble main` | `disas main` | tampilkan assembly fungsi `main` |
| `disassemble /r main` | — | assembly **+ byte mentahnya** (untuk patching nanti) |
| `break *add` | `b *add` | berhenti tepat di **instruksi pertama** `add` |
| `break add` | `b add` | berhenti di awal badan fungsi (prologue biasanya sudah dilewati) |
| `break frame.c:5` | — | berhenti di baris tertentu (butuh `-g`) |
| `run` | `r` | jalankan program dari awal |
| `starti` | — | berhenti di instruksi paling pertama program |
| `info registers` | `i r` | lihat semua register |
| `info registers eax eflags` | — | lihat register tertentu saja |
| `stepi` | `si` | maju **satu instruksi**, masuk ke dalam `call` |
| `nexti` | `ni` | maju satu instruksi, **lompati** `call` |
| `finish` | — | jalankan sampai fungsi saat ini selesai, tampilkan nilai kembaliannya |
| `continue` | `c` | lanjut sampai breakpoint berikutnya |
| `backtrace` | `bt` | lihat rantai pemanggilan fungsi |
| `info frame` | — | detail stack frame aktif (di mana saved rbp & return address) |
| `p/x $rax` | — | cetak nilai register dalam hex |
| `set $rax = 0` | — | **ubah** nilai register saat program berjalan |
| `layout regs` | — | tampilan TUI: register + assembly bergerak bersamaan |
| `quit` | `q` | keluar |

### 6.4 Membaca memori dengan `x/`

Formatnya: `x/<jumlah><format><ukuran> <alamat>`

- **jumlah** = berapa elemen yang ditampilkan.
- **format** = `x` hex, `d` desimal, `s` string, `i` instruksi.
- **ukuran** = `b` 1 byte, `h` 2 byte, `w` 4 byte, `g` 8 byte.

| Perintah | Artinya |
| :--- | :--- |
| `x/16gx $rsp` | 16 slot **8-byte** dari puncak stack, dalam hex (paling cocok untuk stack x86-64) |
| `x/20wx $rsp` | 20 slot 4-byte dari puncak stack (berguna untuk melihat array `int`) |
| `x/10i $rip` | 10 instruksi mulai dari posisi sekarang |
| `x/s $rdi` | string yang ditunjuk `RDI` |
| `x/gx $rbp` | isi 8 byte di alamat `RBP` (= saved rbp) |
| `x/wx $rbp-0x4` | variabel lokal 4 byte (`int`) di `[rbp-0x4]` |

### 6.5 Lab 1 — Lihat argumen, variabel lokal, dan return address

**Tujuan:** membuktikan bahwa argumen datang lewat register, variabel lokal hidup di stack,
dan return address tersimpan di sana.

```gdb
set disassembly-flavor intel
break *add
run
info registers rdi rsi rip rsp
x/gx $rsp
```

Output (dirapikan; alamat di mesinmu akan berbeda):

```text
Breakpoint 1, 0x0000555555555135 in add ()
rdi            0x7                 7
rsi            0x5                 5
rip            0x555555555135      0x555555555135 <add>
rsp            0x7fffffffe098      0x7fffffffe098
0x7fffffffe098: 0x0000555555555174
```

Baca hasilnya: `rdi = 7` dan `rsi = 5` → itulah `x` dan `y` yang dikirim `main`, sesuai ABI.
Isi `[rsp]` adalah **return address**: alamat instruksi `main` tepat setelah `call add`.

Sekarang jalankan prologue satu instruksi demi satu:

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

`RSP` turun 8 byte (`...e098` → `...e090`) karena `push rbp`. Setelah `mov rbp, rsp`, `RBP` dan
`RSP` sama. Dua slot teratas stack sekarang: **rbp lama** lalu **return address** — persis
gambar di bagian 5.4.

Lanjut ke badan fungsi untuk melihat variabel lokal:

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

Bandingkan dengan disassembly `add` (hasil `objdump -d -M intel frame`, sudah diverifikasi):

```asm
add:
    push   rbp
    mov    rbp,rsp
    mov    DWORD PTR [rbp-0x14],edi    ; a disalin dari RDI ke stack
    mov    DWORD PTR [rbp-0x18],esi    ; b disalin dari RSI ke stack
    mov    edx,DWORD PTR [rbp-0x14]
    mov    eax,DWORD PTR [rbp-0x18]
    add    eax,edx                     ; hasil perhitungan di EAX
    mov    DWORD PTR [rbp-0x4],eax     ; sum = variabel lokal
    mov    eax,DWORD PTR [rbp-0x4]     ; nilai kembalian di EAX
    pop    rbp
    ret
```

> [!IMPORTANT]
> **THE AHA:** Nama `a`, `b`, dan `sum` sudah dibuang compiler (PART 0), tetapi **posisinya
> tetap**: `[rbp-0x14]`, `[rbp-0x18]`, `[rbp-0x4]`. Reverse engineer membaca variabel dari
> pola offset ini, bukan dari nama.

### 6.6 Lab 2 — Belokkan keputusan `vault` lewat register

**Tujuan:** membayar cliffhanger PART 0 — masuk ke `vault` **tanpa** mengubah satu byte pun
di file, hanya dengan mengubah register.

```bash
gcc -g -O0 vault.c -o vault
gdb -q ./vault
```

```gdb
set disassembly-flavor intel
break strcmp
run
```

Ketik password salah, mis. `tebak`, lalu tekan Enter. Program berhenti di dalam `strcmp`.
Kembali ke `main` dan lihat instruksi yang menunggu:

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

`EAX = 1` (tidak nol) berarti password berbeda. Sekarang ubah nilainya **sebelum** `test`
dijalankan, lalu amati flag-nya:

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

Rantai kejadiannya: `EAX = 0` → `test eax,eax` menghasilkan nol → **`ZF` menyala** → `jne`
(loncat jika `ZF = 0`) **tidak** loncat → program masuk ke cabang `ACCESS GRANTED`.

> [!TIP]
> Bisa juga langsung menyalakan flag-nya: `set $eflags = $eflags | 0x40` (`0x40` = bit `ZF`).
> Untuk melihat kondisi awal, jalankan `info registers eflags` sebelum `si`: tanpa `ZF`,
> nilainya `0x202 [ IF ]`.

> [!NOTE]
> `break strcmp` bisa dipakai karena `vault` terhubung dinamis ke libc. Kalau binary-nya
> statis atau simbolnya dihapus (*stripped*), pasang breakpoint di alamatnya instead —
> teknik itu dibahas di Fase 3 dan 5.

> [!IMPORTANT]
> **THE AHA:** PART 0 mengubah **data di file** (permanen). Lab ini mengubah **data di memori
> saat program berjalan** (hilang begitu program ditutup — jalankan ulang gdb dan hasilnya
> kembali `ACCESS DENIED`). Dua-duanya membelokkan keputusan tanpa menyentuh logikanya.
> Langkah berikutnya: mengubah **instruksinya** sendiri.

### 6.7 Amati perubahan real-time

Saat melangkah dengan `si`, perhatikan tiga hal ini:

- [ ] `RIP` bertambah sesuai panjang instruksi, dan **meloncat** saat `call`/`jne`/`ret`.
- [ ] `RSP` turun 8 saat `push`, naik 8 saat `pop`.
- [ ] Slot stack di `[rsp]` berubah menjadi return address begitu `call` dieksekusi.

Biar tidak perlu mengetik ulang terus:

```gdb
display/i $rip
layout regs
```

`display/i $rip` otomatis mencetak instruksi berikutnya setiap kali program berhenti;
`layout regs` membuka tampilan TUI (keluar dengan `Ctrl+X` lalu `A`, atau `tui disable`).

## 7. Rekap

| Konsep | Inti yang harus kamu ingat |
| :--- | :--- |
| CPU & `RIP` | siklus fetch-decode-execute; `RIP` menentukan instruksi berikutnya |
| Register | storage tercepat; `RAX` nilai kembalian, `RDI`/`RSI` argumen pertama/kedua |
| `RFLAGS` | `cmp`/`test` menyalakan flag, loncat bersyarat membacanya (`ZF` → `je`/`jne`) |
| Memori | kode di Text (read-only), global di Data/BSS, dinamis di Heap, fungsi di Stack |
| Stack | LIFO, tumbuh ke alamat rendah; `PUSH`/`POP` menggeser `RSP` 8 byte |
| Stack frame | `RBP` jadi patok; variabel lokal = `[rbp-offset]`; return address = `[rbp+8]` |
| GDB | `break *fungsi`, `si`/`ni`, `info registers`, `x/`, `set $reg` |

### Cek dirimu

Kamu siap lanjut ke Fase 2 jika bisa menjawab:

- [ ] Apa tiga tahap siklus CPU, dan apa peran `RIP` di dalamnya?
- [ ] Di mana argumen ke-1 sampai ke-6 sebuah fungsi berada (Linux x86-64)?
- [ ] Apa beda `RSP` dan `RBP`, dan kenapa `RSP` berkurang saat `push`?
- [ ] Apa yang disimpan `CALL` ke stack, dan apa yang dilakukan `RET`?
- [ ] Kenapa `test eax,eax` bisa menentukan apakah `jne` loncat atau tidak?
- [ ] Kenapa kode di Text segment tidak bisa ditambal saat program berjalan?
- [ ] Bagaimana kamu membuat `vault` menerima password salah **tanpa** mengubah file-nya?

## 8. Cliffhanger: dari register ke instruksi

Kamu sudah bisa membaca **keadaan** mesin (register, flag, stack) dan mengubah **datanya**.
Yang belum: membaca dan menulis **instruksinya**.

Coba lihat byte mentah di balik keputusan `vault`:

```gdb
disassemble /r main
```

Kamu akan melihat baris seperti `75 0e` di samping `jne`. `75` adalah *opcode* `jne`, dan
`0e` adalah jarak lompatannya. Ganti `75` menjadi `74` (`je`) dan logikanya terbalik permanen
di dalam file — persis ancaman yang disebut di PART 0.

Tapi untuk melakukan itu dengan aman kamu harus bisa membaca instruksi apa pun, bukan cuma
menebak satu byte.

> [!IMPORTANT]
> Itulah **Fase 2 — Assembly Language**: membaca disassembly x86-64 dengan lancar, lalu
> mengenali instruksi mana yang aman untuk diubah.

## 9. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| CPU | otak komputer; menjalankan instruksi satu per satu |
| Fetch-decode-execute | siklus kerja CPU: ambil instruksi → terjemahkan → jalankan |
| Opcode | angka yang mewakili sebuah instruksi (`75` = `jne`, `74` = `je`) |
| Register | tempat penyimpanan super cepat di dalam CPU (`RAX`, `RSP`) |
| GPR | *General Purpose Register*: register serbaguna untuk hitung & pindah data |
| Sub-register | potongan dari register yang sama: `RAX` → `EAX` → `AX` → `AL` |
| RIP | *Instruction Pointer*: alamat instruksi berikutnya |
| RSP | *Stack Pointer*: alamat puncak stack saat ini |
| RBP | *Base/Frame Pointer*: patok dasar stack frame fungsi aktif |
| RFLAGS | register berisi bit status hasil operasi terakhir |
| ZF / SF / CF / OF | flag nol / negatif / carry (unsigned) / overflow (signed) |
| cmp / test | menghitung untuk menyalakan flag, hasilnya dibuang |
| ABI | aturan baku cara fungsi menerima argumen & mengembalikan nilai (System V di Linux) |
| Caller-saved / callee-saved | register yang boleh / wajib dipertahankan oleh fungsi yang dipanggil |
| Pointer | nilai yang berisi alamat memori, bukan datanya langsung |
| Dereference `[...]` | "ambil isi memori pada alamat ini" (`[rsp]` = isi di alamat `rsp`) |
| Word / dword / qword | 2 byte / 4 byte / 8 byte (di gdb: `h` / `w` / `g`) |
| Virtual address space | ruang alamat "seolah milik sendiri" yang dilihat tiap program, dipetakan OS |
| Text segment | wilayah memori berisi kode program (read-only, executable) |
| Data / BSS | variabel global: `.data` sudah punya nilai awal, `.bss` masih kosong (nol) |
| Heap | memori alokasi dinamis (`malloc`), tumbuh ke alamat tinggi |
| Stack | memori LIFO untuk variabel lokal & alur fungsi, tumbuh ke alamat rendah |
| LIFO | *Last-In, First-Out*: yang terakhir masuk, pertama keluar |
| PUSH / POP | menaruh / mengambil data di puncak stack (menggeser `RSP` 8 byte) |
| Stack frame | "ruang kerja" satu pemanggilan fungsi di dalam stack |
| CALL / RET | `CALL` menyimpan return address lalu loncat; `RET` pulang ke alamat itu |
| Return address | alamat instruksi yang dijalankan setelah fungsi selesai (`[rbp+8]`) |
| Prologue / epilogue | instruksi pembuka/penutup standar sebuah fungsi |
| leave | singkatan dari `mov rsp, rbp` + `pop rbp` |
| Stack alignment | aturan stack harus berkelipatan 16 byte saat `call` |
| mmap / library region | wilayah tempat shared library (mis. libc) dipetakan, antara heap dan stack |
| ASLR / PIE | pengacakan alamat memori oleh OS / binary yang bisa dipindah alamatnya |
| Debug symbol (`-g`) | info nama fungsi, variabel, dan baris yang disertakan saat kompilasi |
| Stripped | binary yang simbolnya sudah dihapus (lebih sulit dianalisis) |
| GDB | *GNU Debugger*: menjalankan & memeriksa program secara langsung (dinamis) |
| Breakpoint | titik henti yang dipasang di debugger |
| stepi / nexti / finish | maju 1 instruksi (masuk `call`) / maju 1 instruksi (lompati `call`) / selesaikan fungsi |
| Intel vs AT&T | dua gaya penulisan assembly; urutan operandnya terbalik (`mov rbp,rsp` vs `mov %rsp,%rbp`) |

---

[← PART 0](../PART0/note.id.md) · [Peta belajar](../INDEX.id.md) · **Selanjutnya:** PART 2 —
Bahasa Assembly ([catatan](../PART2/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>