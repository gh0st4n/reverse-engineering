# PART 0 — Fondasi (*Foundations*)

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 0 di YouTube](https://www.youtube.com/watch?v=zDJtr9r2RFs&t=197s)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

## Daftar Isi

1. [Apa itu Reverse Engineering?](#1-apa-itu-reverse-engineering)
2. [Kenapa mempelajari ini?](#2-kenapa-mempelajari-ini)
3. [Ide inti: compiler membuang informasi](#3-ide-inti-compiler-membuang-informasi)
4. [Dari source code ke binary: 4 tahap](#4-dari-source-code-ke-binary-4-tahap)
5. [Dua pendekatan: statis vs dinamis](#5-dua-pendekatan-statis-vs-dinamis)
6. [Kapan memakai yang mana?](#6-kapan-memakai-yang-mana)
7. [Setup lab](#7-setup-lab)
8. [Starter toolkit](#8-starter-toolkit)
9. [Praktik: membongkar `vault`](#9-praktik-membongkar-vault)
10. [Rekap](#10-rekap)
11. [Cliffhanger: cara ke-4](#11-cliffhanger-cara-ke-4)
12. [Kamus istilah](#12-kamus-istilah)

---

## 1. Apa itu Reverse Engineering?

**Reverse Engineering (RE)**, atau *rekayasa balik*, adalah memahami cara kerja sebuah program **tanpa memiliki source code-nya**.

- **Source code** (kode sumber) = tulisan asli programmer, misalnya file `.c`, yang bisa dibaca manusia.
- **Binary** = file program hasil kompilasi (`.exe` di Windows, ELF di Linux). Isinya *machine code* (kode mesin): deretan angka yang dimengerti CPU, bukan manusia.

Kamu hanya punya binary-nya. Tugasmu adalah **menyusun ulang logika program** dari binary itu.

> [!TIP]
> **Berpikirlah seperti detektif.** *Compiler* (program yang menerjemahkan source code menjadi binary) bekerja seperti **mesin penghancur kertas**: nama variabel, nama fungsi, komentar, dan struktur kode dibuang. Tetapi **logikanya tetap bertahan**, dan logika itulah yang kamu rangkai kembali.

---

## 2. Kenapa mempelajari ini?

| Bidang | Apa yang dikerjakan |
|---|---|
| **Malware Analysis** (analisis malware) | Membongkar program berbahaya untuk tahu persis apa yang dilakukannya |
| **Firmware & IoT** | Membedah kode di dalam perangkat tertanam (*embedded*): router, mobil, kamera CCTV, dll. *Firmware* = program yang tertanam di perangkat keras |
| **CTF & Crackme** | *CTF* (*Capture The Flag*) adalah kompetisi keamanan. *Crackme* adalah program kecil yang sengaja dibuat untuk dibongkar sebagai latihan. Tujuannya menyelesaikan tantangan dan menangkap *flag* (kode bukti berhasil) |
| **Vulnerability Research** (riset celah keamanan) | Membongkar software untuk menemukan celah yang bisa dieksploitasi |

---

## 3. Ide inti: compiler membuang informasi

Berikut fungsi sederhana di sisi kiri (yang ditulis programmer), dan sisi kanan adalah apa yang kamu lihat saat membongkar binary-nya (hasil *disassembly*, yaitu kode mesin yang diterjemahkan ke bahasa *assembly*):

```
 SOURCE CODE (C)                          BINARY → DISASSEMBLY
┌─────────────────────────────┐          ┌──────────────────────────┐
│ // menjumlahkan dua angka   │          │ sub_401136:              │
│ int add(int a, int b) {     │   ───▶   │     lea eax, [rdi+rsi]   │
│     return a + b;           │          │     ret                  │
│ }                           │          │                          │
└─────────────────────────────┘          └──────────────────────────┘
```

| Hilang | Tetap ada |
|---|---|
| Nama (`add`, `a`, `b`) → diganti nama otomatis seperti `sub_401136` | **Logika**: ada penjumlahan dua nilai lalu hasilnya dikembalikan |
| Komentar | |
| Struktur kode asli | |

Penjelasan singkat potongan assembly di atas (detailnya di Fase 1 dan 2):

- `rdi` dan `rsi` adalah **register** (tempat penyimpanan super cepat di dalam CPU). Di Linux x86_64, argumen pertama dan kedua fungsi dititipkan di sana.
- `lea eax, [rdi+rsi]` menghitung `rdi + rsi` lalu menyimpannya di `eax` (tempat hasil fungsi).
- `ret` = selesai, kembali ke pemanggil.

> [!IMPORTANT]
> Mengubah binary kembali menjadi kode yang bisa dipahami itulah pekerjaan reverse engineer.

---

## 4. Dari source code ke binary: 4 tahap

```
 vault.c ──▶ Preprocess ──▶ Compile ──▶ Assemble ──▶ Link ──▶ vault
 (source)      (.i)          (.s)         (.o)       (ELF/.exe) (binary)
```

| # | Tahap | Apa yang terjadi | Hasil |
|---|---|---|---|
| 1 | **Preprocess** | Menjabarkan `#include` dan `#define` (menempelkan isi file header, mengganti makro) | `.i` (masih kode C) |
| 2 | **Compile** | Mengubah kode **C** menjadi bahasa **rakitan (assembly)** | `.s` |
| 3 | **Assemble** | Mengubah assembly menjadi **kode mesin** | `.o` (*object file*) |
| 4 | **Link** | Menggabungkan potongan-potongan (`.o` + *library*) menjadi file eksekusi akhir | executable |

> [!NOTE]
> Setiap tool RE hanya memperlihatkan hasil dari **salah satu** tahap ini. Misalnya `objdump` dan Ghidra bekerja pada hasil akhir (tahap 4), lalu mencoba menerjemahkan mundur.

### Menjalankan tiap tahap dengan `gcc`

```shell
$ gcc -E test.c -o test.i   # 1. preprocess → C yang sudah dijabarkan
$ gcc -S test.c -o test.s   # 2. compile    → assembly
$ gcc -c test.c -o test.o   # 3. assemble   → kode mesin (belum bisa dijalankan)
$ gcc test.c -o test        # 4. link       → executable
```

Setiap flag (`-E`, `-S`, `-c`) menghentikan pipeline **satu tahap lebih awal**. File `.s` adalah tempat kode C berubah menjadi logika mesin yang masih bisa dibaca manusia.

> [!TIP]
> Coba sendiri: buat `test.c` kecil, jalankan `gcc -S test.c`, lalu buka `test.s` dengan editor teks.

---

## 5. Dua pendekatan: statis vs dinamis

Analogi: **statis** = membaca *denah rumah* tanpa masuk. **Dinamis** = masuk ke rumah dan melihat penghuninya beraktivitas.

| | **Analisis Statis** (*Static Analysis*) | **Analisis Dinamis** (*Dynamic Analysis*) |
|---|---|---|
| **Konsep** | Membaca/menganalisis binary **tanpa menjalankannya** | **Menjalankan** program dan mengamati perilakunya langsung |
| **Kelebihan** | Bisa melihat seluruh struktur kode sekaligus. Aman, karena tidak ada kode berbahaya yang berjalan | Bisa melihat nilai register dan memori secara *real-time*. Tahu perilaku asli program saat berinteraksi dengan sistem |
| **Kekurangan** | Tidak menampilkan *runtime values* (nilai asli saat program berjalan) | Hanya melihat *code path* (jalur kode) yang benar-benar dieksekusi. Kode tersembunyi yang tidak terpanggil tidak akan terlihat |
| **Tools** | `strings`, `objdump`, `ghidra` | `gdb`, `ltrace`, `strace` |

---

## 6. Kapan memakai yang mana?

- **Static dulu → buat PETA.** Lihat fungsi apa saja yang ada dan bagaimana keterhubungannya.
- **Dynamic untuk *zoom*.** Hilangkan kebingungan di **satu titik** spesifik, misalnya: "isi register ini sebenarnya apa sekarang?"

> [!TIP]
> **Aturan praktis:** statis untuk memetakan keseluruhan, dinamis untuk memperbesar satu titik. Reverse engineer yang baik terus berpindah-pindah di antara keduanya.

---

## 7. Setup lab

### Virtual Machine (VM)

**Kenapa pakai VM?** VM adalah komputer virtual di dalam komputermu. Saat menganalisis **malware sungguhan**, malware hanya berjalan di dalam VM sehingga **sistem utamamu tidak ikut terinfeksi**.

Kebiasaan baik untuk lab malware (berlaku nanti, bukan untuk Part 0):

- Ambil **snapshot** (titik simpan) sebelum menjalankan sampel, supaya mudah dikembalikan.
- Matikan *shared folder* dan *clipboard sharing* antara VM dan host.
- Gunakan jaringan *host-only* atau tanpa jaringan.

> [!NOTE]
> Latihan di Part 0 memakai program buatan kita sendiri (`vault.c`) dan tidak berbahaya, jadi boleh dikerjakan langsung di sistemmu. Tetapi biasakan disiplin ini sejak awal.

### Install tool

Ghidra tidak ada di semua package manager. Unduh terpisah dari [situs resminya](https://github.com/NationalSecurityAgency/ghidra) (butuh Java/JDK), atau pakai package manager jika tersedia. Untuk Part 0, kamu belum butuh Ghidra.

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
> Paket **binutils** saja sudah menyediakan `objdump`, `readelf`, `nm`, dan `strings`, yaitu alat-alat utama untuk Fase 3 dan 4.

Cek instalasi:

```shell
gcc --version && gdb --version && objdump --version | head -1
```

---

## 8. Starter toolkit

| Tool | Fungsi | Jenis |
|---|---|---|
| `strings` | Mengambil **teks yang bisa dibaca** langsung dari data biner | Statis |
| `objdump` | **Disassembler**: membongkar kode mesin kembali menjadi kode assembly | Statis |
| `ltrace` | Melihat **library call** (pemanggilan fungsi *library*, misalnya `strcmp`, `printf`) saat program berjalan | Dinamis |
| `strace` | Melihat **system call** (permintaan program ke kernel, misalnya membuka/membaca file) saat program berjalan | Dinamis |
| `gdb` | **Debugger**: jeda program, jalankan selangkah demi selangkah, periksa memori secara langsung | Dinamis |
| `ghidra` | **Disassembler + decompiler** lengkap dan gratis (IDE utama kita). *Decompiler* mencoba menebak kode C dari assembly | Statis |

---

## 9. Praktik: membongkar `vault`

### Program target: `vault.c`

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

Program ini meminta password, membandingkannya dengan `secret` memakai `strcmp` (fungsi pembanding string: hasilnya `0` jika sama), lalu mencetak hasilnya.

**Kelihatannya aman, padahal tidak.** Kita akan menjebolnya dengan cara-cara berikut **tanpa perlu menguasai assembly**:

1. **Read** the secret: baca rahasianya
2. **Sniff** it live: sadap saat program berjalan
3. **Patch** the data: ubah datanya

> [!WARNING]
> **Sebelum mulai:**
> - Latihan 3, 4, dan 5 **mengubah file `vault`**. Setiap kali ingin mulai dari kondisi bersih, kompilasi ulang: `gcc vault.c -o vault`
> - Atau buat cadangan dulu: `cp vault vault.bak`
> - Latihan RE hanya pada program milikmu sendiri atau yang memang diizinkan untuk dibongkar (misalnya crackme).

Coba jalankan dulu:

```shell
$ gcc vault.c -o vault
$ ./vault
Enter the vault password: tebak
ACCESS DENIED
```

---

### Latihan 1: Baca rahasianya (*Read the secret*)

```shell
$ gcc vault.c -o vault
$ strings vault | grep sup
sup3rs3cr3t
```

`strings` mencetak semua deretan karakter yang bisa dibaca di dalam file binary, dan `grep sup` menyaring yang mengandung "sup".

> [!IMPORTANT]
> **THE AHA:** Rahasia yang dikompilasi ke dalam binary **bukanlah rahasia**. Inilah alasan *hardcoded credentials* (kredensial yang ditulis langsung di dalam kode) menjadi temuan nyata dalam pentest (*penetration testing*, uji penetrasi).

---

### Latihan 2: Sadap secara langsung (*Sniff it live*)

```shell
$ ltrace ./vault
printf("Enter the vault password: ")              = 26
__isoc99_scanf(0x55bf3a66402b, 0x7ffe25e3a060, 0, 0) = 1
strcmp("testsalah", "sup3rs3cr3t")                 = 1
puts("ACCESS DENIED")                              = 14
+++ exited (status 0) +++
```

> [!NOTE]
> Output asli `ltrace` bercampur dengan teks yang dicetak program sendiri (`Enter the vault password: ...`). Di atas sudah dirapikan supaya mudah dibaca. Alamat seperti `0x55bf3a66402b` akan berbeda tiap kali dijalankan (karena **ASLR**, pengacakan alamat memori oleh sistem operasi).

Cara membacanya:

- `strcmp("testsalah", "sup3rs3cr3t") = 1`: program membandingkan password yang kamu ketik dengan rahasia. **Keduanya tampil utuh sebagai teks biasa.** Hasil `1` (bukan `0`) berarti **tidak sama**.
- Kenapa tertulis `puts` padahal di kode kita memakai `printf`? Karena compiler cukup pintar: `printf("...\n")` tanpa format variabel diubah menjadi `puts`, yang lebih efisien. Ini contoh pertama bahwa **binary tidak persis mencerminkan source code** (lebih lanjut di Latihan 7).

> [!IMPORTANT]
> **THE AHA:** Sekalipun rahasianya diacak atau disembunyikan di dalam file, ia **harus muncul sebagai plaintext (teks biasa) di memori** pada saat perbandingan. Analisis dinamis bisa menangkapnya.

---

### Latihan 3: Edit binary dengan tangan (*Edit a Binary by Hand*)

```shell
# Sebelum
$ ./vault        # → ACCESS DENIED

# Tukar teks dengan panjang SAMA (13 karakter)
$ sed -i 's/ACCESS DENIED/H4CK3D BY YOU/' vault

# Sesudah
$ ./vault        # → H4CK3D BY YOU
```

`sed -i 's/lama/baru/' file` mencari teks `lama` dan menggantinya dengan `baru` langsung di dalam file (`-i` = *in-place*). Karena teks itu tersimpan utuh di dalam binary, `sed` bisa mengeditnya.

Hitung sendiri: `ACCESS DENIED` = 13 karakter, `H4CK3D BY YOU` = 13 karakter. **Panjangnya harus sama.**

> [!IMPORTANT]
> **THE AHA:** Binary hanyalah file berisi byte yang bisa kamu tulis ulang. **Panjang yang sama itu penting.** Mengubah ukuran akan menggeser semua byte setelahnya (alamat, *offset*) dan bisa merusak program.

---

### Latihan 4: Bypass autentikasi tanpa assembly

```shell
$ sed -i 's/sup3rs3cr3t/AAAAAAAAAAA/' vault
$ ./vault
Enter the vault password: AAAAAAAAAAA
ACCESS GRANTED
```

`sup3rs3cr3t` dan `AAAAAAAAAAA` sama-sama 11 karakter. Kita mengganti **kunci jawaban** di dalam binary dengan jawaban yang sudah kita ketahui.

> [!IMPORTANT]
> **THE AHA:** Kamu melewati pemeriksaan autentikasi dengan mengedit **DATA**, bukan logikanya. Kamu mengubah jawaban yang benar menjadi jawaban yang sudah kamu tahu.

---

### Latihan 5: Rusak-kan *Magic Bytes*

```shell
$ xxd vault | head -1
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
```

`xxd` menampilkan isi file dalam **hexadecimal** (angka basis 16, satu byte = dua digit). Kolom kanan menampilkan versi teksnya: empat byte pertama `7f 45 4c 46` terbaca `.ELF`.

Sekarang rusak-kan byte pertama:

```shell
# (kompilasi ulang dulu: gcc vault.c -o vault)
$ printf '\000' | dd of=vault bs=1 seek=0 count=1 conv=notrunc
$ ./vault
bash: ./vault: cannot execute binary file: Exec format error
```

Arti opsi `dd`: `bs=1` (blok 1 byte), `seek=0` (mulai menulis di posisi 0), `count=1` (tulis 1 blok), `conv=notrunc` (jangan potong file).

> [!TIP]
> Kenapa `\000` dan bukan `\x00`? Di beberapa shell (misalnya `dash`, yang sering menjadi `/bin/sh`), `printf '\x00'` tidak didukung dan akan menulis karakter `\` yang salah. Format oktal `\000` bekerja di semua shell.

> [!IMPORTANT]
> **THE AHA:** Linux mengenali file yang bisa dieksekusi dari **magic bytes**-nya (`7f 45 4c 46` = `.ELF`), **bukan dari ekstensi file**. Ini pengantar ke topik format file (Fase 3). **ELF** adalah format file executable di Linux.

---

### Latihan 6: Lihat *endianness* dengan mata sendiri

```shell
$ python3 -c "import struct,sys; sys.stdout.buffer.write(struct.pack('<I', 0x41424344))" | xxd
00000000: 4443 4241                                DCBA
```

Kita menulis angka `0x41424344` (empat byte: `41 42 43 44`, yaitu huruf `A B C D`), tapi yang tersimpan adalah `44 43 42 41` (`D C B A`). Urutannya **terbalik**.

> [!IMPORTANT]
> **THE AHA:** Arsitektur x86 menyimpan nilai multi-byte dengan meletakkan **byte yang paling tidak signifikan (*least-significant byte*) lebih dulu**. Cara pengurutan data di memori ini disebut **little-endian**. Nanti saat membaca dump memori di `gdb`, kamu akan sering menemukan angka yang "tampak terbalik". Sekarang kamu tahu alasannya.

---

### Latihan 7: Fungsi yang menghilang (*The disappearing function*)

Source code:

```c
static int compute(void) { return 3 * 4 * 5; }
int main(void)           { return compute(); }
```

Hasil assembly (disederhanakan; ditampilkan dalam sintaks Intel):

```
 gcc -O0  (tanpa optimasi)               gcc -O2  (dioptimasi)
┌─────────────────────────────────┐     ┌──────────────────────────────┐
│ compute:                        │     │ main:                        │
│     push rbp                    │     │     mov  eax, 60             │
│     mov  rbp, rsp               │     │     ret                      │
│     mov  eax, 60                │     │                              │
│     pop  rbp                    │     │ ; compute() lenyap: sudah    │
│     ret                         │     │ ; "dilebur" ke dalam main    │
│ main:                           │     │                              │
│     push rbp                    │     │                              │
│     mov  rbp, rsp               │     │                              │
│     call compute                │     │                              │
│     pop  rbp                    │     │                              │
│     ret                         │     │                              │
└─────────────────────────────────┘     └──────────────────────────────┘
```

Coba sendiri:

```shell
$ gcc -S -O0 -masm=intel c.c -o c_O0.s
$ gcc -S -O2 -masm=intel c.c -o c_O2.s
```

> [!NOTE]
> Perhatikan huruf **O besar** pada `-O0` / `-O2`. Huruf kecil `-o` artinya "nama file output". File `.s` asli berisi baris tambahan (direktif `.cfi`, `endbr64`, dsb.) yang sudah dibuang pada gambar di atas supaya fokus.

Dua hal yang terjadi:

1. **Constant folding**: compiler menghitung `3*4*5 = 60` saat kompilasi, sehingga tidak ada instruksi perkalian sama sekali (bahkan pada `-O0`).
2. **Inlining**: pada `-O2`, isi `compute()` disisipkan langsung ke `main()` dan fungsi `compute` dibuang karena sudah tidak diperlukan.

> [!IMPORTANT]
> **Kode mesin BUKAN cerminan dari source code kamu.** Mengenali apa yang dilakukan compiler (*optimization*, pengoptimalan kode) adalah keterampilan inti untuk semua materi berikutnya.

---

## 10. Rekap

| # | Cara | Tool | Jenis analisis | Hasil |
|---|---|---|---|---|
| 1 | **Read** | `strings` | Statis | Rahasia tersaji sebagai teks biasa di dalam file |
| 2 | **Sniff** | `ltrace` | Dinamis | Rahasia tertangkap langsung di memori saat perbandingan |
| 3 | **Patch** | `sed` | Edit data | Kita menulis ulang data sehingga jawaban yang kita ketahui diterima |

Latihan 5 sampai 7 adalah bonus: *magic bytes*, *endianness*, dan perilaku compiler.

### Cek dirimu

Kamu siap lanjut jika bisa menjawab:

- [ ] Apa beda source code dan binary?
- [ ] Apa saja yang dibuang compiler, dan apa yang tetap bertahan?
- [ ] Sebutkan 4 tahap dari `.c` sampai executable beserta flag `gcc`-nya.
- [ ] Kapan memakai analisis statis, kapan dinamis?
- [ ] Kenapa menyimpan password di dalam kode itu berbahaya?
- [ ] Kenapa panjang string pengganti harus sama saat mem-patch?

---

## 11. Cliffhanger: cara ke-4

**Cara ke-4 membutuhkan... assembly.**

Perhatikan potongan *disassembly* `vault` ini (disederhanakan):

```asm
    call    strcmp          ; bandingkan input dengan rahasia → hasil di eax (0 = sama)
    test    eax, eax        ; apakah eax == 0 ?
    jne     .denied         ; kalau TIDAK 0 (beda) → loncat ke DENIED   ◀── balik INI
    ; ... cetak ACCESS GRANTED
.denied:
    ; ... cetak ACCESS DENIED
```

Bacanya: `jne` = *jump if not equal* (loncat jika tidak sama).

**Patch keputusannya itu sendiri:** ubah `jne` menjadi `je` (*jump if equal*) dan logikanya **terbalik**: password yang salah malah diterima. Atau hapus lompatannya sama sekali (ditimpa instruksi `NOP`, "tidak melakukan apa-apa") dan password **apa pun** akan diterima.

Tetapi sekarang kamu harus bisa **membaca dan mengedit instruksi sebenarnya**.

> [!IMPORTANT]
> Itulah yang akan dibuka oleh **Fase 1 (arsitektur)** dan **Fase 2 (assembly)**.

---

## 12. Kamus istilah

| Istilah | Arti sederhana |
|---|---|
| **Source code** | Kode asli yang ditulis programmer, bisa dibaca manusia |
| **Binary** | File program hasil kompilasi, berisi kode mesin |
| **Compiler** | Program penerjemah source code menjadi binary (contoh: `gcc`) |
| **Assembly** | Bahasa tingkat rendah yang hampir 1:1 dengan instruksi CPU (mis. `mov`, `lea`, `ret`) |
| **Machine code** (kode mesin) | Instruksi dalam bentuk angka yang langsung dimengerti CPU |
| **Disassembler** | Alat yang menerjemahkan machine code menjadi assembly (contoh: `objdump`) |
| **Decompiler** | Alat yang menebak kode tingkat tinggi (mirip C) dari binary (contoh: Ghidra) |
| **Register** | Tempat penyimpanan super cepat di dalam CPU (mis. `eax`, `rdi`, `rsi`) |
| **Instruction** | Satu perintah untuk CPU (mis. `test eax, eax`) |
| **Opcode** | Angka yang mewakili sebuah instruksi (mis. `75` = `jne`, `74` = `je`) |
| **Static / Dynamic analysis** | Menganalisis tanpa / dengan menjalankan program |
| **Runtime** | Saat program sedang berjalan |
| **Library call** | Pemanggilan fungsi dari *library* (kumpulan fungsi siap pakai), mis. `strcmp` |
| **System call** | Permintaan program kepada kernel (inti sistem operasi), mis. membuka file |
| **Debugger** | Alat untuk menjeda dan memeriksa program yang sedang berjalan (`gdb`) |
| **Hardcoded credentials** | Password/kunci yang ditulis langsung di dalam kode |
| **ELF** | Format file executable di Linux (padanan `.exe`/PE di Windows) |
| **Magic bytes** | Beberapa byte awal file yang menandakan jenis file (ELF = `7f 45 4c 46`) |
| **Hexadecimal (hex)** | Angka basis 16 (0-9, a-f); satu byte ditulis dua digit hex |
| **Endianness** | Urutan penyimpanan byte dari angka multi-byte; x86 memakai *little-endian* |
| **Patch** | Mengubah byte di dalam binary untuk mengubah perilakunya |
| **NOP** | Instruksi "tidak melakukan apa-apa" (opcode `90` di x86) |
| **Optimization** | Proses compiler membuat kode lebih cepat/kecil (`-O0` = tanpa, `-O2` = sedang) |
| **Constant folding** | Compiler menghitung ekspresi konstan saat kompilasi (`3*4*5` → `60`) |
| **Inlining** | Compiler menyisipkan isi fungsi langsung ke pemanggilnya |
| **ASLR** | Pengacakan alamat memori oleh OS agar sulit ditebak penyerang |
| **CTF / Crackme** | Kompetisi keamanan / program latihan untuk dibongkar |
| **Pentest** | *Penetration testing*: menguji keamanan sistem secara resmi |

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>
