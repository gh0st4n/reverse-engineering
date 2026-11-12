# PART 0 — Fondasi (Foundations)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 0 dari 10**

> [!NOTE]
> **Sumber:** [PART 0 di YouTube](https://www.youtube.com/watch?v=zDJtr9r2RFs&t=197s)
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar
> kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan
> bahasa sederhana. Kamus istilah ada di [bagian 12](#12-kamus-istilah).

**Prasyarat:** Linux x86-64 (boleh di VM) dan bisa memakai terminal (`cd`, `ls`, menjalankan
perintah). Belum perlu paham assembly.

**Setelah part ini kamu bisa:** menjelaskan bagaimana source code menjadi binary, memilih
antara analisis statis dan dinamis, serta menjebol pemeriksaan password sederhana dengan
tiga cara berbeda — tanpa membaca satu baris assembly pun.

## Daftar isi

- [PART 0 — Fondasi (Foundations)](#part-0--fondasi-foundations)
  - [Daftar isi](#daftar-isi)
  - [1. Apa itu reverse engineering?](#1-apa-itu-reverse-engineering)
  - [2. Kenapa mempelajari ini?](#2-kenapa-mempelajari-ini)
  - [3. Ide inti: compiler membuang informasi](#3-ide-inti-compiler-membuang-informasi)
  - [4. Dari source code ke binary: 4 tahap](#4-dari-source-code-ke-binary-4-tahap)
    - [Menjalankan tiap tahap dengan `gcc`](#menjalankan-tiap-tahap-dengan-gcc)
  - [5. Dua pendekatan: statis vs dinamis](#5-dua-pendekatan-statis-vs-dinamis)
  - [6. Kapan memakai yang mana?](#6-kapan-memakai-yang-mana)
  - [7. Setup lab](#7-setup-lab)
    - [7.1 Virtual machine (VM)](#71-virtual-machine-vm)
    - [7.2 Install tool](#72-install-tool)
    - [7.3 Cek instalasi](#73-cek-instalasi)
  - [8. Starter toolkit](#8-starter-toolkit)
  - [9. Praktik: membongkar `vault`](#9-praktik-membongkar-vault)
    - [Latihan 1 — Baca rahasianya (read the secret)](#latihan-1--baca-rahasianya-read-the-secret)
    - [Latihan 2 — Sadap secara langsung (sniff it live)](#latihan-2--sadap-secara-langsung-sniff-it-live)
    - [Latihan 3 — Edit binary dengan tangan](#latihan-3--edit-binary-dengan-tangan)
    - [Latihan 4 — Bypass autentikasi tanpa assembly](#latihan-4--bypass-autentikasi-tanpa-assembly)
    - [Latihan 5 — Rusak-kan magic bytes](#latihan-5--rusak-kan-magic-bytes)
    - [Latihan 6 — Lihat endianness dengan mata sendiri](#latihan-6--lihat-endianness-dengan-mata-sendiri)
    - [Latihan 7 — Fungsi yang menghilang](#latihan-7--fungsi-yang-menghilang)
  - [10. Rekap](#10-rekap)
    - [Cek dirimu](#cek-dirimu)
  - [11. Cliffhanger: cara ke-4](#11-cliffhanger-cara-ke-4)
  - [12. Kamus istilah](#12-kamus-istilah)

---

## 1. Apa itu reverse engineering?

**Reverse Engineering (RE)**, atau rekayasa balik, adalah memahami cara kerja sebuah program
tanpa memiliki source code-nya.

- **Source code** (kode sumber) = tulisan asli programmer, misalnya file `.c`, yang bisa
  dibaca manusia.
- **Binary** = file program hasil kompilasi (`.exe` di Windows, ELF di Linux). Isinya
  **machine code** (kode mesin): deretan angka yang dimengerti CPU, bukan manusia.

Kamu hanya punya binary-nya. Tugasmu adalah menyusun ulang logika program dari binary itu.

> [!TIP]
> Berpikirlah seperti detektif. **Compiler** (program yang menerjemahkan source code menjadi
> binary) bekerja seperti mesin penghancur kertas: nama variabel, nama fungsi, komentar, dan
> struktur kode dibuang. Tetapi logikanya tetap bertahan, dan logika itulah yang kamu rangkai
> kembali.

## 2. Kenapa mempelajari ini?

| Bidang | Apa yang dikerjakan |
| :--- | :--- |
| **Malware analysis** (analisis malware) | Membongkar program berbahaya untuk tahu persis apa yang dilakukannya |
| **Firmware & IoT** | Membedah kode di dalam perangkat tertanam (*embedded*): router, mobil, kamera CCTV. *Firmware* = program yang tertanam di perangkat keras |
| **CTF & crackme** | *CTF* (*Capture The Flag*) adalah kompetisi keamanan; *crackme* adalah program kecil yang sengaja dibuat untuk dibongkar sebagai latihan. Tujuannya menyelesaikan tantangan dan menangkap *flag* (kode bukti berhasil) |
| **Vulnerability research** (riset celah keamanan) | Membongkar software untuk menemukan celah yang bisa dieksploitasi |

## 3. Ide inti: compiler membuang informasi

Kiri: fungsi sederhana yang ditulis programmer. Kanan: apa yang kamu lihat saat membongkar
binary-nya — hasil **disassembly** (kode mesin yang diterjemahkan kembali ke teks assembly).

```text
 SOURCE CODE (C)                          BINARY → DISASSEMBLY
┌─────────────────────────────┐          ┌──────────────────────────┐
│ // menjumlahkan dua angka   │          │ sub_401136:              │
│ int add(int a, int b) {     │   ───▶   │     lea eax, [rdi+rsi]   │
│     return a + b;           │          │     ret                  │
│ }                           │          │                          │
└─────────────────────────────┘          └──────────────────────────┘
```

| Hilang | Tetap ada |
| :--- | :--- |
| Nama (`add`, `a`, `b`) → diganti nama otomatis seperti `sub_401136` | Logika: ada penjumlahan dua nilai lalu hasilnya dikembalikan |
| Komentar | Urutan operasi dan perkabangan (alur percabangan) |
| Struktur kode asli (tipe data, format, tata letak) | Konstanta dan string yang dipakai program |

Penjelasan singkat potongan assembly di atas (detailnya di Fase 1 dan 2):

- `rdi` dan `rsi` adalah **register** (tempat penyimpanan super cepat di dalam CPU). Di Linux
  x86-64, argumen pertama dan kedua fungsi dititipkan di sana.
- `lea eax, [rdi+rsi]` menghitung `rdi + rsi` lalu menyimpannya di `eax` (register tempat
  hasil fungsi).
- `ret` = selesai, kembali ke pemanggil.

> [!IMPORTANT]
> Mengubah binary kembali menjadi kode yang bisa dipahami — itulah pekerjaan reverse engineer.

## 4. Dari source code ke binary: 4 tahap

```text
 vault.c ──▶ Preprocess ──▶ Compile ──▶ Assemble ──▶ Link ──▶ vault
 (source)      (.i)          (.s)         (.o)      (ELF/.exe) (binary)
```

| # | Tahap | Apa yang terjadi | Hasil |
| :---: | :--- | :--- | :--- |
| 1 | **Preprocess** | Menjabarkan `#include` dan `#define` (menempelkan isi file header, mengganti makro) | `.i` (masih kode C) |
| 2 | **Compile** | Mengubah kode C menjadi assembly | `.s` |
| 3 | **Assemble** | Mengubah assembly menjadi machine code | `.o` (*object file*) |
| 4 | **Link** | Menggabungkan potongan-potongan (`.o` + *library*) menjadi file eksekusi akhir | executable |

> [!NOTE]
> Setiap tool RE hanya memperlihatkan hasil dari salah satu tahap ini. Misalnya `objdump` dan
> Ghidra bekerja pada hasil akhir (tahap 4), lalu mencoba menerjemahkan mundur.

### Menjalankan tiap tahap dengan `gcc`

```bash
gcc -E test.c -o test.i   # 1. preprocess → C yang sudah dijabarkan
gcc -S test.c -o test.s   # 2. compile    → assembly
gcc -c test.c -o test.o   # 3. assemble   → machine code (belum bisa dijalankan)
gcc test.c -o test        # 4. link       → executable
```

Setiap flag (`-E`, `-S`, `-c`) menghentikan pipeline satu tahap lebih awal. File `.s` adalah
tempat kode C berubah menjadi logika mesin yang masih bisa dibaca manusia.

> [!TIP]
> Coba sendiri: buat `test.c` kecil, jalankan `gcc -S test.c`, lalu buka `test.s` dengan
> editor teks.

## 5. Dua pendekatan: statis vs dinamis

Analogi: **statis** = membaca denah rumah tanpa masuk. **Dinamis** = masuk ke rumah dan
melihat penghuninya beraktivitas.

| | Analisis statis (*static analysis*) | Analisis dinamis (*dynamic analysis*) |
| :--- | :--- | :--- |
| **Konsep** | Membaca/menganalisis binary **tanpa menjalankannya** | **Menjalankan** program dan mengamati perilakunya langsung |
| **Kelebihan** | Bisa melihat seluruh struktur kode sekaligus. Aman, karena tidak ada kode berbahaya yang berjalan | Bisa melihat nilai register dan memori secara *real-time*. Tahu perilaku asli program saat berinteraksi dengan sistem |
| **Kekurangan** | Tidak menampilkan *runtime values* (nilai asli saat program berjalan) | Hanya melihat *code path* (jalur kode) yang benar-benar dieksekusi. Kode tersembunyi yang tidak terpanggil tidak terlihat |
| **Tools** | `strings`, `objdump`, `ghidra` | `gdb`, `ltrace`, `strace` |

## 6. Kapan memakai yang mana?

- **Static dulu → buat PETA.** Lihat fungsi apa saja yang ada dan bagaimana keterhubungannya.
- **Dynamic untuk zoom.** Hilangkan kebingungan di satu titik spesifik, misalnya: "isi
  register ini sebenarnya apa sekarang?"

> [!TIP]
> Aturan praktis: statis untuk memetakan keseluruhan, dinamis untuk memperbesar satu titik.
> Reverse engineer yang baik terus berpindah-pindah di antara keduanya.

## 7. Setup lab

### 7.1 Virtual machine (VM)

**Kenapa pakai VM?** VM adalah komputer virtual di dalam komputermu. Saat menganalisis
malware sungguhan, malware hanya berjalan di dalam VM sehingga sistem utamamu tidak ikut
terinfeksi.

Kebiasaan baik untuk lab malware (berlaku nanti, bukan untuk Part 0):

- Ambil **snapshot** (titik simpan) sebelum menjalankan sampel, supaya mudah dikembalikan.
- Matikan *shared folder* dan *clipboard sharing* antara VM dan host.
- Gunakan jaringan *host-only* atau tanpa jaringan.

> [!NOTE]
> Latihan di Part 0 memakai program buatan kita sendiri (`vault.c`) dan tidak berbahaya, jadi
> boleh dikerjakan langsung di sistemmu. Tetapi biasakan disiplin ini sejak awal.

### 7.2 Install tool

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
# xxd ada di dalam paket vim-common pada Fedora
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
> Paket `binutils` saja sudah menyediakan `objdump`, `readelf`, `nm`, dan `strings`, yaitu
> alat-alat utama untuk Fase 3 dan 4. `python3` hanya dipakai di Latihan 6 dan biasanya sudah
> terpasang. **Ghidra** tidak ada di semua package manager: unduh dari
> [situs resminya](https://github.com/NationalSecurityAgency/ghidra) (butuh Java/JDK).
> Untuk Part 0, kamu belum butuh Ghidra.

### 7.3 Cek instalasi

```bash
gcc --version | head -1
gdb --version | head -1
objdump --version | head -1
command -v strings xxd file ltrace strace python3
```

Semua perintah harus mencetak versi atau path. Kalau ada yang kosong, install dulu sebelum
lanjut.

## 8. Starter toolkit

| Tool | Fungsi | Jenis |
| :--- | :--- | :---: |
| `strings` | Mengambil teks yang bisa dibaca langsung dari data biner | Statis |
| `objdump` | **Disassembler**: membongkar machine code kembali menjadi assembly | Statis |
| `ltrace` | Melihat *library call* (pemanggilan fungsi *library*, mis. `strcmp`, `printf`) saat program berjalan | Dinamis |
| `strace` | Melihat *system call* (permintaan program ke kernel, mis. membuka/membaca file) saat program berjalan | Dinamis |
| `gdb` | **Debugger**: jeda program, jalankan selangkah demi selangkah, periksa memori langsung | Dinamis |
| `ghidra` | Disassembler + **decompiler** lengkap dan gratis (IDE utama kita). Decompiler mencoba menebak kode C dari assembly | Statis |

Tool pembantu yang dipakai di latihan: `file` (mengenali file dari isinya), `xxd` (penampil
hex), `sed` dan `dd` (menulis ulang byte), `python3` (membuat byte percobaan).

## 9. Praktik: membongkar `vault`

**Program target:** `vault.c`

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

Program ini meminta password, membandingkannya dengan `secret` memakai `strcmp` (fungsi
pembanding string: hasilnya `0` jika sama), lalu mencetak hasilnya.

Kelihatannya aman, padahal tidak. Kita akan menjebolnya dengan **tiga cara** — read, sniff,
patch (Latihan 1–4) — lalu **tiga lab bonus**: magic bytes, endianness, dan perilaku compiler
(Latihan 5–7). Tanpa perlu menguasai assembly.

> [!WARNING]
> - Latihan 3, 4, dan 5 **mengubah file `vault`**. Untuk mulai dari kondisi bersih, kompilasi
>   ulang: `gcc vault.c -o vault`, atau buat cadangan: `cp vault vault.bak`.
> - Latihan RE hanya pada program milikmu sendiri atau yang memang diizinkan untuk dibongkar
>   (kode sendiri, crackme, target dengan izin).

> [!NOTE]
> Output di bawah sudah diverifikasi di Linux x86-64. Hasilmu bisa sedikit berbeda: alamat
> berubah setiap kali dijalankan (ASLR/PIE), dan `gcc` versi baru menyisipkan instruksi
> tambahan seperti `endbr64`.

Coba jalankan dulu sebelum dibongkar:

```bash
gcc vault.c -o vault
./vault
```

```text
Enter the vault password: tebak
ACCESS DENIED
```

### Latihan 1 — Baca rahasianya (read the secret)

**Tujuan:** menemukan password tanpa menjalankan program (statis).

```bash
gcc vault.c -o vault
strings vault | grep sup
```

```text
sup3rs3cr3t
```

`strings` mencetak semua deretan karakter yang bisa dibaca di dalam file binary, dan
`grep sup` menyaring yang mengandung "sup".

> [!IMPORTANT]
> **THE AHA:** Rahasia yang dikompilasi ke dalam binary bukanlah rahasia. Inilah alasan
> *hardcoded credentials* (kredensial yang ditulis langsung di dalam kode) menjadi temuan
> nyata dalam pentest (*penetration testing*, uji penetrasi).

### Latihan 2 — Sadap secara langsung (sniff it live)

**Tujuan:** menangkap password saat program berjalan (dinamis).

```bash
ltrace ./vault
```

Ketik password salah apa pun, mis. `testsalah`. Output yang sudah dirapikan:

```text
printf("Enter the vault password: ")                 = 26
__isoc99_scanf(0x55bf3a66402b, 0x7ffe25e3a060, 0, 0)  = 1
strcmp("testsalah", "sup3rs3cr3t")                    = 1
puts("ACCESS DENIED")                                = 14
+++ exited (status 0) +++
```

Cara membacanya:

- `strcmp("testsalah", "sup3rs3cr3t")` — input kamu dan rahasianya tampil **berdampingan
  sebagai teks biasa**. Hasil `0` berarti sama; nilai bukan nol berarti berbeda. (Angka
  persisnya adalah selisih byte, jadi jangan bergantung pada nilai `1`.)
- Kenapa tertulis `puts` padahal kode kita memakai `printf`? Karena compiler cukup pintar:
  `printf("...\n")` tanpa format variabel diubah menjadi `puts` yang lebih efisien. Ini
  contoh pertama bahwa binary tidak persis mencerminkan source code (lanjut di Latihan 7).

> [!NOTE]
> Output asli `ltrace` bercampur dengan teks yang dicetak program sendiri; di atas sudah
> dirapikan supaya mudah dibaca. Alamat seperti `0x55bf3a66402b` berbeda tiap kali dijalankan
> karena **ASLR** (pengacakan alamat memori oleh sistem operasi).

> [!IMPORTANT]
> **THE AHA:** Sekalipun rahasianya diacak atau disembunyikan di dalam file, ia harus muncul
> sebagai *plaintext* (teks biasa) di memori pada saat perbandingan. Analisis dinamis bisa
> menangkapnya.

### Latihan 3 — Edit binary dengan tangan

**Tujuan:** mengubah teks yang dicetak program, tanpa kompilasi ulang.

```bash
./vault                                        # sebelum: ACCESS DENIED
sed -i 's/ACCESS DENIED/H4CK3D BY YOU/' vault
./vault                                        # sesudah: H4CK3D BY YOU
```

`sed -i 's/lama/baru/' file` mencari teks `lama` dan menggantinya dengan `baru` langsung di
dalam file (`-i` = *in-place*). Karena teks itu tersimpan utuh di dalam binary, `sed` bisa
mengeditnya.

Hitung sendiri: `ACCESS DENIED` = 13 karakter, `H4CK3D BY YOU` = 13 karakter.
**Panjangnya harus sama.**

> [!TIP]
> Alternatif yang presisi per byte (lebih aman untuk binary — tidak menyentuh byte lain):
>
> ```bash
> gcc vault.c -o vault                       # mulai dari kondisi bersih
> off=$(grep -abo 'ACCESS DENIED' vault | head -1 | cut -d: -f1)
> printf 'H4CK3D BY YOU' | dd of=vault bs=1 seek="$off" conv=notrunc
> ```
>
> `grep -abo` memberi **offset byte** teks itu; `dd` lalu menulis tepat 13 byte di posisi tersebut.

> [!IMPORTANT]
> **THE AHA:** Binary hanyalah file berisi byte yang bisa kamu tulis ulang. Panjang yang sama
> itu penting: mengubah ukuran akan menggeser semua byte setelahnya (alamat, offset) dan bisa
> merusak program.

### Latihan 4 — Bypass autentikasi tanpa assembly

**Tujuan:** masuk dengan password pilihanmu sendiri.

```bash
gcc vault.c -o vault
sed -i 's/sup3rs3cr3t/AAAAAAAAAAA/' vault
./vault
```

```text
Enter the vault password: AAAAAAAAAAA
ACCESS GRANTED
```

`sup3rs3cr3t` dan `AAAAAAAAAAA` sama-sama 11 karakter. Kita mengganti kunci jawaban di dalam
binary dengan jawaban yang sudah kita ketahui.

> [!IMPORTANT]
> **THE AHA:** Kamu melewati pemeriksaan autentikasi dengan mengedit **DATA**, bukan
> logikanya. Kamu mengubah jawaban yang benar menjadi jawaban yang sudah kamu tahu.

### Latihan 5 — Rusak-kan magic bytes

**Tujuan:** melihat bagaimana Linux memutuskan sebuah file adalah program.

```bash
xxd vault | head -1
file vault
```

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
vault: ELF 64-bit LSB pie executable, x86-64, ...
```

`xxd` menampilkan isi file dalam **hexadecimal** (angka basis 16; satu byte = dua digit).
Kolom kanan menampilkan versi teksnya: empat byte pertama `7f 45 4c 46` terbaca `.ELF`.

Sekarang rusak-kan byte pertama:

```bash
gcc vault.c -o vault                       # mulai dari kondisi bersih
printf '\000' | dd of=vault bs=1 seek=0 count=1 conv=notrunc
./vault
file vault
```

```text
bash: ./vault: cannot execute binary file: Exec format error
vault: data
```

Arti opsi `dd`: `bs=1` (blok 1 byte), `seek=0` (mulai menulis di posisi 0), `count=1` (tulis
1 blok), `conv=notrunc` (jangan potong file).

> [!TIP]
> Kenapa `\000` dan bukan `\x00`? Di beberapa shell (mis. `dash`, yang sering menjadi
> `/bin/sh`), `printf '\x00'` tidak didukung dan akan menulis karakter `\` yang salah. Format
> oktal `\000` bekerja di semua shell.

> [!IMPORTANT]
> **THE AHA:** Linux mengenali file executable dari **magic bytes**-nya (`7f 45 4c 46` =
> `.ELF`), bukan dari ekstensi file. Perhatikan `file` sekarang hanya bilang "data" — ia
> membaca isi, bukan nama. Ini pengantar ke topik format file (Fase 3).

### Latihan 6 — Lihat endianness dengan mata sendiri

**Tujuan:** melihat urutan byte yang benar-benar dipakai x86.

```bash
python3 -c "import struct,sys; sys.stdout.buffer.write(struct.pack('<I', 0x41424344))" | xxd
```

```text
00000000: 4443 4241                                DCBA
```

Kita menulis angka `0x41424344` (empat byte `41 42 43 44`, yaitu huruf `A B C D`), tapi yang
tersimpan adalah `44 43 42 41` (`D C B A`). Urutannya terbalik.

> [!IMPORTANT]
> **THE AHA:** Arsitektur x86 menyimpan nilai multi-byte dengan meletakkan byte paling tidak
> signifikan (*least-significant byte*) lebih dulu. Cara pengurutan ini disebut
> **little-endian**. Nanti saat membaca dump memori di `gdb`, kamu akan sering menemukan
> angka yang "tampak terbalik" — sekarang kamu tahu alasannya.

### Latihan 7 — Fungsi yang menghilang

**Tujuan:** membuktikan compiler menulis ulang kodemu.

```c
static int compute(void) { return 3 * 4 * 5; }
int main(void)           { return compute(); }
```

Hasil assembly (disederhanakan, sintaks Intel):

```text
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

```bash
gcc -S -O0 -masm=intel c.c -o c_O0.s
gcc -S -O2 -masm=intel c.c -o c_O2.s
```

> [!NOTE]
> Perhatikan huruf **`O` besar** pada `-O0` / `-O2`. Huruf kecil `-o` artinya "nama file
> output". File `.s` asli berisi baris tambahan (direktif `.cfi`, `endbr64`, dsb.) yang
> dibuang pada gambar di atas supaya fokus.

Dua hal yang terjadi:

1. **Constant folding** — compiler menghitung `3*4*5 = 60` saat kompilasi, sehingga tidak ada
   instruksi perkalian sama sekali (bahkan pada `-O0`).
2. **Inlining** — pada `-O2`, isi `compute()` disisipkan langsung ke `main()`, dan fungsi
   `compute` dibuang karena sudah tidak diperlukan.

> [!IMPORTANT]
> Machine code **BUKAN** cerminan source code kamu. Mengenali apa yang dilakukan compiler
> (*optimization*, pengoptimalan kode) adalah keterampilan inti untuk semua materi berikutnya.

## 10. Rekap

| # | Cara | Tool | Jenis analisis | Hasil |
| :---: | :--- | :--- | :--- | :--- |
| 1 | Read | `strings` | Statis | Rahasia tersaji sebagai teks biasa di dalam file |
| 2 | Sniff | `ltrace` | Dinamis | Rahasia tertangkap langsung di memori saat perbandingan |
| 3 | Patch | `sed` / `dd` | Edit data | Kita menulis ulang data sehingga jawaban yang kita ketahui diterima |

Latihan 5–7 adalah lab bonus: magic bytes, endianness, dan perilaku compiler.

### Cek dirimu

Kamu siap lanjut ke Fase 1 jika bisa menjawab:

- [ ] Apa beda source code dan binary?
- [ ] Apa saja yang dibuang compiler, dan apa yang tetap bertahan?
- [ ] Sebutkan 4 tahap dari `.c` sampai executable beserta flag `gcc`-nya.
- [ ] Kapan memakai analisis statis, kapan dinamis?
- [ ] Kenapa menyimpan password di dalam kode itu berbahaya?
- [ ] Kenapa panjang string pengganti harus sama saat mem-patch?

## 11. Cliffhanger: cara ke-4

Cara ke-4 membutuhkan... **assembly**.

Perhatikan potongan disassembly `vault` ini (disederhanakan):

```asm
    call    strcmp          ; bandingkan input dengan rahasia → hasil di eax (0 = sama)
    test    eax, eax        ; apakah eax == 0 ?
    jne     .denied         ; kalau TIDAK sama → loncat ke DENIED   ◀── balik INI
    ; ... puts("ACCESS GRANTED")
.denied:
    ; ... puts("ACCESS DENIED")
```

`jne` = *jump if not equal* (loncat jika tidak sama).

**Patch keputusannya sendiri:** ubah `jne` menjadi `je` (*jump if equal*) dan logikanya
terbalik — password salah malah diterima. Atau timpa lompatannya dengan `NOP` ("tidak
melakukan apa-apa") dan password **apa pun** akan diterima. Tetapi sekarang kamu harus bisa
membaca dan mengedit instruksi sebenarnya.

> [!IMPORTANT]
> Itulah yang akan dibuka oleh **Fase 1** (arsitektur) dan **Fase 2** (assembly).

## 12. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| Source code | Kode asli yang ditulis programmer, bisa dibaca manusia |
| Binary | File program hasil kompilasi, berisi machine code |
| Compiler | Program penerjemah source code menjadi binary (contoh: `gcc`) |
| Assembly | Bahasa tingkat rendah yang hampir 1:1 dengan instruksi CPU (`mov`, `lea`, `ret`) |
| Machine code (kode mesin) | Instruksi dalam bentuk angka yang langsung dimengerti CPU |
| Disassembler | Alat yang menerjemahkan machine code menjadi assembly (contoh: `objdump`) |
| Decompiler | Alat yang menebak kode tingkat tinggi (mirip C) dari binary (contoh: Ghidra) |
| Register | Tempat penyimpanan super cepat di dalam CPU (`eax`, `rdi`, `rsi`) |
| Instruction | Satu perintah untuk CPU (mis. `test eax, eax`) |
| Opcode | Angka yang mewakili sebuah instruksi (`75` = `jne`, `74` = `je`) |
| Static / dynamic analysis | Menganalisis tanpa / dengan menjalankan program |
| Runtime | Saat program sedang berjalan |
| Library call | Pemanggilan fungsi dari *library* (kumpulan fungsi siap pakai), mis. `strcmp` |
| System call | Permintaan program kepada kernel (inti sistem operasi), mis. membuka file |
| Debugger | Alat untuk menjeda dan memeriksa program yang sedang berjalan (`gdb`) |
| Hardcoded credentials | Password/kunci yang ditulis langsung di dalam kode |
| ELF | Format file executable di Linux (padanan `.exe` / PE di Windows) |
| Magic bytes | Beberapa byte awal file yang menandakan jenis file (ELF = `7f 45 4c 46`) |
| Hexadecimal (hex) | Angka basis 16 (`0-9`, `a-f`); satu byte ditulis dua digit hex |
| Endianness | Urutan penyimpanan byte dari angka multi-byte; x86 memakai little-endian |
| Patch | Mengubah byte di dalam binary untuk mengubah perilakunya |
| NOP | Instruksi "tidak melakukan apa-apa" (opcode `90` di x86) |
| Optimization | Proses compiler membuat kode lebih cepat/kecil (`-O0` = tanpa, `-O2` = sedang) |
| Constant folding | Compiler menghitung ekspresi konstan saat kompilasi (`3*4*5` → `60`) |
| Inlining | Compiler menyisipkan isi fungsi langsung ke pemanggilnya |
| ASLR | Pengacakan alamat memori oleh OS agar sulit ditebak penyerang |
| CTF / crackme | Kompetisi keamanan / program latihan untuk dibongkar |
| Pentest | *Penetration testing*: menguji keamanan sistem secara resmi |

---

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Selanjutnya:** PART 1 —
Dasar Arsitektur ([catatan](../PART1/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>