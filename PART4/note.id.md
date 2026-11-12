# PART 4 — Analisis Statis (Static Analysis)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 4 dari 10**

> [!NOTE]
> **Sumber:** [PART 4 di YouTube](https://www.youtube.com/watch?v=sbGsMc9_qvg)
> Istilah teknis tetap ditulis dalam bahasa aslinya, tetapi selalu dijelaskan dengan bahasa
> sederhana. Kamus istilah: [bagian 12](#12-kamus-istilah).

**Prasyarat:** [PART 3](../PART3/note.id.md) selesai — kamu tahu apa itu section, simbol,
`strip`, dan bedanya alamat memori dengan offset file.

**Setelah part ini kamu bisa:** mengimpor binary ke Ghidra, menavigasi tiga panel utamanya,
**menemukan fungsi penting walau namanya sudah di-strip** (lewat string + XREF), membaca dan
merapikan pseudo-C, serta mem-patch instruksi langsung dari GUI.

## Daftar isi

- [PART 4 — Analisis Statis (Static Analysis)](#part-4--analisis-statis-static-analysis)
  - [Daftar isi](#daftar-isi)
  - [1. Kenapa butuh decompiler?](#1-kenapa-butuh-decompiler)
  - [2. Apa itu Ghidra](#2-apa-itu-ghidra)
  - [3. Instalasi Ghidra](#3-instalasi-ghidra)
  - [4. Project dan import](#4-project-dan-import)
    - [Import summary = ELF header yang dibaca otomatis](#import-summary--elf-header-yang-dibaca-otomatis)
  - [5. Auto-analysis](#5-auto-analysis)
  - [6. Tiga panel Code Browser](#6-tiga-panel-code-browser)
  - [7. Membaca pseudo-code Ghidra](#7-membaca-pseudo-code-ghidra)
    - [XREF: senjata utama analis statis](#xref-senjata-utama-analis-statis)
  - [8. Refactoring: rename, retype, comment](#8-refactoring-rename-retype-comment)
  - [9. Praktik: membongkar vault di Ghidra](#9-praktik-membongkar-vault-di-ghidra)
    - [9.1 Siapkan target](#91-siapkan-target)
    - [9.2 Lab 1 — Import dan baca metadata sebagai ELF header](#92-lab-1--import-dan-baca-metadata-sebagai-elf-header)
    - [9.3 Lab 2 — Temukan fungsi utama tanpa nama](#93-lab-2--temukan-fungsi-utama-tanpa-nama)
    - [9.4 Lab 3 — Baca pseudo-C, lalu rapikan](#94-lab-3--baca-pseudo-c-lalu-rapikan)
    - [9.5 Lab 4 — Patch instruksi dari GUI](#95-lab-4--patch-instruksi-dari-gui)
    - [9.6 Cheat sheet navigasi](#96-cheat-sheet-navigasi)
  - [10. Rekap](#10-rekap)
    - [Cek dirimu](#cek-dirimu)
  - [11. Cliffhanger: ketika decompiler menebak salah](#11-cliffhanger-ketika-decompiler-menebak-salah)
  - [12. Kamus istilah](#12-kamus-istilah)

---

## 1. Kenapa butuh decompiler?

Sejauh ini kita membaca manual, baris demi baris:

| Fase | Yang kita lakukan | Alat |
| :---: | :--- | :--- |
| 0 | Membongkar rahasia tanpa assembly | `strings`, `ltrace`, `sed` |
| 1 | Memahami register, flags, stack frame | `gdb` |
| 2 | Membaca instruksi & mem-patch byte | `objdump`, `dd` |
| 3 | Membaca peta file: section, simbol, offset | `readelf`, `nm` |

`vault` hanya punya satu fungsi logika. Program nyata punya **ribuan**. Membaca semuanya dengan
`objdump` = berhari-hari, dan kamu akan tenggelam sebelum menemukan bagian yang penting.

> [!TIP]
> Analogi: PART 2 itu membaca gedung **bata demi bata**. Ghidra memberi kamu **denah lengkap
> plus daftar ruangan**, lalu menebak "ruangan ini sepertinya dapur". Kamu tetap perlu
> memverifikasi tebakannya, tapi tidak perlu lagi menghitung bata.

Yang kita butuhkan: alat yang memetakan seluruh fungsi sekaligus, lalu menerjemahkan assembly
kembali menjadi kode yang mirip C. Itulah **decompiler**.

## 2. Apa itu Ghidra

**Ghidra** adalah framework reverse engineering gratis dan open-source yang dirilis NSA
(badan keamanan AS). Dua kemampuan utamanya:

| Kemampuan | Apa yang dilakukan | Padanan manual kita |
| :--- | :--- | :--- |
| **Disassembler** | memetakan seluruh instruksi + fungsi + referensi silang | `objdump -d` (PART 2) |
| **Decompiler** | menebak kode mirip C dari assembly | tidak ada — ini kemampuan baru |

> [!IMPORTANT]
> Hasil decompiler disebut **pseudo-code**: mirip C, tapi **bukan source code asli**. Nama
> variabel, tipe data, dan kadang struktur kontrolnya adalah **tebakan** Ghidra. Kode asli
> sudah dibuang compiler (PART 0) dan tidak bisa dikembalikan persis.

Karena analisisnya statis, binary **tidak dijalankan** — inilah cara paling aman membedah
sampel berbahaya. Tetap kerjakan di VM (disiplin PART 0): tool analisis sendiri bisa punya
celah saat mengurai file jahat.

## 3. Instalasi Ghidra

Ghidra butuh **Java JDK** (versi 17 atau 21 tergantung rilis Ghidra — cek catatan rilis versinya)
dan tampilan grafis (GUI).

<details>
<summary><b>APT</b> — Kali Linux (paket tersedia)</summary>

```bash
sudo apt update
sudo apt install ghidra
ghidra
```

</details>

<details>
<summary><b>Pacman</b> — Arch Linux</summary>

```bash
sudo pacman -S ghidra
ghidraRun
```

</details>

<details>
<summary><b>Manual</b> — semua distro (Fedora, Void, Debian, dll.)</summary>

Unduh rilis resmi (berformat `.zip`) dari
[github.com/NationalSecurityAgency/ghidra/releases](https://github.com/NationalSecurityAgency/ghidra/releases),
lalu pasang ke `/opt`:

```bash
# JDK dulu (pilih sesuai package manager)
sudo apt install openjdk-21-jdk        # Debian/Ubuntu/Kali
sudo pacman -S jdk21-openjdk           # Arch
java -version                          # pastikan terbaca

# pasang Ghidra ke /opt, lalu buat pintasan di PATH
sudo unzip ghidra_*_PUBLIC_*.zip -d /opt/
sudo ln -s /opt/ghidra_*_PUBLIC/ghidraRun /usr/local/bin/ghidra
ghidra
```

</details>

> [!NOTE]
> Ghidra tidak ada di repo resmi semua distro. Kalau package manager-mu tidak menyediakannya,
> cara manual di atas justru lebih rapi: satu folder di `/opt`, satu symlink, dan mudah
> dihapus (`rm -rf /opt/ghidra_*` + hapus symlink).

> [!TIP]
> Untuk otomatisasi nanti, Ghidra punya mode tanpa GUI: `support/analyzeHeadless` — bisa
> menjalankan analisis dan skrip dari terminal.

## 4. Project dan import

Ghidra bekerja dengan **project**, bukan langsung membuka file.

1. **File → New Project** → pilih **Non-Shared Project** → Next.
2. Tentukan **Project Directory** (folder tempat database analisis disimpan) dan
   **Project Name**, mis. `vault-lab` → Finish.
3. **File → Import File** → pilih binary `vault` → OK.

> [!IMPORTANT]
> Project menyimpan **hasil analisismu** (nama baru, komentar, tipe data) di database terpisah.
> File binary asli **tidak berubah**. Ini beda dengan `sed`/`dd` di PART 0–2 yang langsung
> menulis ke file.

### Import summary = ELF header yang dibaca otomatis

Setelah impor, Ghidra menampilkan ringkasan:

| Atribut | Nilai | Dari mana asalnya (PART 3) |
| :--- | :--- | :--- |
| Language ID | `x86:LE:64:default` | `e_machine` + endianness + `EI_CLASS` |
| Processor | x86 | `e_machine = 0x3e` |
| Endian | Little | byte ke-5 ELF header = `01` |
| Address size | 64 | byte ke-4 ELF header = `02` |
| Compiler | GCC | tebakan dari pola kode & `.comment` |
| Format | ELF | magic `7f 45 4c 46` |

`x86:LE:64:default` dibaca: prosesor **x86**, **L**ittle-**E**ndian, **64**-bit, varian default.

> [!IMPORTANT]
> Ghidra tidak melakukan sihir di tahap ini — ia membaca 64 byte pertama yang sudah kamu
> terjemahkan manual di PART 3 Lab 1. Kalau Language ID-nya salah, artinya header binary-nya
> memang aneh (atau file itu bukan ELF biasa).

## 5. Auto-analysis

Klik kanan `vault` di jendela project → **Open with Code Browser**. Muncul pertanyaan
"…has not been analyzed. Would you like to analyze it now?" → **Yes** → biarkan opsi
**default** → **Analyze**.

Yang dikerjakan Ghidra di belakang layar:

- membaca section & segment (peta PART 3), lalu menandai mana kode dan mana data;
- menelusuri alur dari entry point dan fungsi-fungsi yang dipanggil, lalu memotong kode
  menjadi **fungsi**;
- mengenali pola prologue/epilogue (PART 1) dan signature fungsi library;
- membangun **XREF** (cross-reference): daftar "siapa memanggil siapa" dan "siapa memakai
  data ini";
- menjalankan decompiler untuk setiap fungsi.

> [!NOTE]
> Untuk binary kecil, analisis selesai dalam hitungan detik. Untuk binary besar bisa menit.
> Opsi default sudah cukup untuk 95% kasus; opsi lanjutan baru relevan saat menghadapi kode
> yang dikaburkan (Fase 8).

## 6. Tiga panel Code Browser

| Panel | Posisi | Isi | Analogi |
| :--- | :--- | :--- | :--- |
| **Symbol Tree** | kiri | daftar Functions, Imports, Exports, Labels, dan Data Type Manager | daftar isi buku |
| **Listing** | tengah | assembly per alamat, byte mentah, label, komentar | halaman asli buku |
| **Decompiler** | kanan | pseudo-C dari fungsi yang sedang dipilih | ringkasan bahasa manusia |

Ketiganya **tersinkron**: klik fungsi di Symbol Tree → Listing melompat ke alamatnya →
Decompiler menampilkan pseudo-C-nya. Klik baris di Decompiler → baris assembly terkait
disorot di Listing.

Jendela pendukung yang wajib kamu tahu:

| Jendela | Cara membuka | Gunanya |
| :--- | :--- | :--- |
| **Defined Strings** | Window → Defined Strings | semua string yang dikenali + alamatnya (versi GUI dari `strings`, PART 0) |
| **Function Graph** | ikon graph di toolbar Listing | alur percabangan fungsi sebagai diagram blok |
| **Function Call Graph** | Window → Function Call Graph |谁 memanggil siapa, satu tingkat |
| **Memory Map** | Window → Memory Map | section/segment + image base (PART 3) |

> [!TIP]
> Kalau panel Decompiler tidak terlihat: **Window → Decompile**. Tata letak bisa diatur ulang
> lewat drag; tidak ada risiko merusak binary.

## 7. Membaca pseudo-code Ghidra

Ghidra memberi nama otomatis yang konsisten. Begitu kamu hafal polanya, pseudo-C langsung
terbaca:

| Nama | Arti |
| :--- | :--- |
| `FUN_00101165` | fungsi belum bernama; angkanya adalah **alamatnya** |
| `LAB_001011c6` | label, yaitu target loncatan di dalam fungsi |
| `DAT_0010202b` | data belum bernama di alamat itu |
| `PTR_...` | data yang berisi pointer |
| `local_58` | variabel lokal; angka hex = offset di stack frame |
| `param_1` | argumen pertama fungsi |
| `iVar1` / `uVar1` | variabel sementara bertipe signed / unsigned |
| `undefined8` / `undefined4` | tipe belum diketahui, berukuran 8 / 4 byte |
| `__isoc99_scanf` | nama asli simbol libc (varian C99 dari `scanf`) |

Dua hal yang sering membingungkan pemula:

> [!NOTE]
> - **Ghidra menulis `JNZ`, `objdump` menulis `jne`.** Instruksinya sama, ejaannya beda —
>   mirip situasi Intel vs AT&T di PART 2. Pasangan lainnya: `JZ` = `je`, `JLE` = `jle`.
> - **Alamat Ghidra bukan alamat runtime.** Untuk ELF PIE, Ghidra memakai *image base*
>   default `0x00100000`. Jadi `FUN_00101165` = `0x1165` di `objdump` (PART 0) = `0x555555555165`
>   saat dijalankan di gdb (PART 1). Tiga angka, satu tempat.

### XREF: senjata utama analis statis

**XREF (cross-reference)** adalah daftar referensi ke sebuah alamat: siapa yang memanggil
fungsi ini, atau instruksi mana yang memakai data ini.

Rahasianya: **kamu bisa berjalan mundur**. Dari string yang menarik → lihat siapa yang
memakainya → sampai ke fungsi yang relevan. Nama fungsi boleh hilang, string biasanya tidak.

## 8. Refactoring: rename, retype, comment

Tiga aksi yang mengubah pseudo-C dari "tebakan mesin" menjadi "catatan yang bisa dibaca":

| Aksi | Cara | Efek |
| :--- | :--- | :--- |
| **Rename** | klik kanan nama → Rename Variable/Function (`L`) | `iVar1` → `match`, `FUN_00101165` → `main` |
| **Retype** | klik kanan → Retype (`Ctrl+L`) | `undefined8` → `int`, `local_58` → `char[64]` |
| **Comment** | klik kanan baris → Set Comment (`;`) | catatanmu muncul di Listing **dan** Decompiler |

> [!IMPORTANT]
> Rename, retype, dan comment **hanya mengubah database project**, bukan file binary.
> Buktikan sendiri: `sha256sum vault` sebelum dan sesudah — hash-nya identik. Yang benar-benar
> mengubah byte hanyalah **patching** (Lab 4), itu pun baru tersimpan ke file kalau kamu export.

> [!TIP]
> Beri nama berdasarkan **perilaku**, bukan tebakan asal: `check_password`, `decrypt_config`,
> `send_beacon`. Nama yang baik adalah dokumentasi yang bertahan saat kamu kembali minggu depan.

## 9. Praktik: membongkar vault di Ghidra

### 9.1 Siapkan target

```bash
gcc -g -O0 vault.c -o vault
cp vault vault.stripped
strip --strip-all vault.stripped
sha256sum vault vault.stripped
```

Kita sengaja memakai **versi stripped** untuk sebagian lab, supaya kamu merasakan kondisi
nyata: nama fungsi hilang (PART 3 Lab 4), tapi logikanya tetap ada.

> [!NOTE]
> Semua nama, alamat, dan tampilan di bawah adalah **contoh**. Versi Ghidra dan gcc yang
> berbeda bisa menghasilkan nama/offset yang sedikit berbeda. Yang penting polanya.

### 9.2 Lab 1 — Import dan baca metadata sebagai ELF header

**Tujuan:** membuktikan bahwa ringkasan impor hanyalah PART 3 yang dibaca otomatis.

Import `vault.stripped` ke project → catat Language ID, Endian, Address size, Format →
bandingkan dengan:

```bash
readelf -h vault.stripped | grep -E 'Class|Data|Machine|Type'
```

```text
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
```

Cocok satu per satu dengan tabel di bagian 4.

> [!IMPORTANT]
> **THE AHA:** Ghidra bukan kotak hitam. Setiap kolom di ringkasan impor punya asal byte yang
> sudah kamu baca manual di PART 3.

### 9.3 Lab 2 — Temukan fungsi utama tanpa nama

**Tujuan:** menemukan `main` di binary yang sudah di-strip, tanpa menebak alamat.

1. **Window → Defined Strings**.
2. Cari `sup3rs3cr3t` → klik dua kali. Listing melompat ke alamat `00102010` (di `.rodata`).
3. Klik kanan alamat itu → **References → Show References to Address**.
4. Muncul satu referensi dari `FUN_00101165` → klik dua kali untuk melompat ke sana.

```text
Address     String
00102010    sup3rs3cr3t

XREF to 00102010:
  FUN_00101165  0010116d  LEA RDI=>s_sup3rs3cr3t_00102010
```

Bandingkan dengan PART 3: string itu memang tinggal di `.rodata` pada offset `0x2010`, dan
`0x1165` adalah alamat `main` yang dulu masih punya nama.

Sekarang beri nama: klik `FUN_00101165` → tekan `L` → ketik `main` → OK.

> [!TIP]
> Jalur alternatif kalau string-nya dienkripsi: cari pemanggilan fungsi menarik di
> **Symbol Tree → Imports** (`strcmp`, `memcmp`, `strstr`), lalu lihat XREF-nya. Fungsi
> pembanding hampir selalu dipanggil dari tempat keputusan dibuat.

> [!IMPORTANT]
> **THE AHA:** Nama boleh dibuang, tapi **referensi tidak bisa disembunyikan dengan mudah**.
> String → XREF → fungsi adalah jalur tercepat menuju inti program, dan itu murni analisis
> statis: programnya tidak pernah dijalankan.

### 9.4 Lab 3 — Baca pseudo-C, lalu rapikan

**Tujuan:** membaca hasil decompiler dan mengubahnya menjadi catatan yang manusiawi.

Klik `main` → panel Decompiler menampilkan (contoh):

```c
undefined8 main(void)
{
  int iVar1;
  char local_58 [64];
  char *local_10;

  local_10 = "sup3rs3cr3t";
  printf("Enter the vault password: ");
  __isoc99_scanf(&DAT_0010202b,local_58);
  iVar1 = strcmp(local_58,local_10);
  if (iVar1 == 0) {
    puts("ACCESS GRANTED");
  }
  else {
    puts("ACCESS DENIED");
  }
  return 0;
}
```

Bandingkan dengan source asli di PART 0. Tiga perbedaan yang normal:

| Perbedaan | Kenapa |
| :--- | :--- |
| `undefined8 main(void)` | Ghidra belum tahu tipe kembaliannya (`int`) |
| `local_58`, `local_10` | nama variabel dibuang compiler; ini offset stack (PART 1) |
| `local_10 = "sup3rs3cr3t"` lalu dipakai | compiler menyimpan pointer ke stack dulu (`[rbp-0x8]`, PART 0) |
| `&DAT_0010202b` | format string `"%63s"` belum diberi nama |

Rapikan dengan rename + retype:

| Sebelum | Sesudah | Aksi |
| :--- | :--- | :--- |
| `undefined8 main` | `int main` | Retype return (`Ctrl+L`) |
| `iVar1` | `match` | Rename (`L`) |
| `local_58` | `input` | Rename |
| `local_10` | `secret` | Rename |
| `DAT_0010202b` | `fmt_password` | Rename label |

Hasilnya:

```c
int main(void)
{
  int match;
  char input [64];
  char *secret;

  secret = "sup3rs3cr3t";
  printf("Enter the vault password: ");
  __isoc99_scanf(&fmt_password,input);
  match = strcmp(input,secret);
  if (match == 0) {
    puts("ACCESS GRANTED");
  }
  else {
    puts("ACCESS DENIED");
  }
  return 0;
}
```

Tambahkan komentar (`;`) di baris `strcmp`: `keputusan autentikasi — lihat PART 2 Lab 4`.

```bash
sha256sum vault.stripped
```

Hash-nya **tidak berubah** sejak 9.1.

> [!IMPORTANT]
> **THE AHA:** Dekompilasi bukan mengembalikan source code, tapi **membangun pemahaman**.
> Pekerjaan reverse engineer adalah mengganti tebakan mesin dengan nama yang bermakna —
> dan semuanya tersimpan di project, bukan di binary.

### 9.5 Lab 4 — Patch instruksi dari GUI

**Tujuan:** melakukan patch `75` → `74` seperti PART 2 Lab 4, tapi lewat Ghidra.

1. Di Listing, lompat ke alamat `001011b6` (tekan `G`, ketik alamatnya).
2. Kamu akan melihat `JNZ LAB_001011c6` — itu `jne` yang sama dari PART 2.
3. Klik kanan instruksi → **Patch Instruction** (`Ctrl+Shift+H`) → ganti `JNZ` menjadi `JZ` → Enter.
4. Lihat panel Decompiler: kondisinya sekarang terbalik (`if (match != 0)`).
5. Simpan project (`Ctrl+S`), lalu **File → Export Program** → format **Original File** →
   simpan sebagai `vault.ghidra-patched`.

Verifikasi di terminal:

```bash
xxd -s 0x11b6 -l 2 vault.stripped
xxd -s 0x11b6 -l 2 vault.ghidra-patched
chmod +x vault.ghidra-patched
./vault.ghidra-patched
```

```text
000011b6: 750e                                     u.
000011b6: 740e                                     t.
Enter the vault password: tebak
ACCESS GRANTED
```

> [!NOTE]
> Nama format export bisa berbeda antar versi Ghidra. Kalau ragu, pakai cara yang sudah pasti
> dari PART 2: catat offsetnya dari Ghidra, lalu `printf '\164' | dd ...`. Ghidra untuk
> **menemukan**, `dd` untuk **menulis**.

> [!WARNING]
> Patch Instruction mengubah byte di **database project** dulu; file asli baru berubah saat
> di-export. Simpan hasil export dengan nama baru supaya binary asli tetap utuh.

> [!IMPORTANT]
> **THE AHA:** GUI dan command line melakukan hal yang persis sama: mengganti satu byte.
> Bedanya, di Ghidra kamu langsung **melihat efeknya pada pseudo-C** — jadi kamu paham
> keputusan mana yang sedang kamu balik.

### 9.6 Cheat sheet navigasi

| Shortcut | Aksi |
| :--- | :--- |
| `G` | Go To alamat |
| `L` | rename label / fungsi / variabel |
| `Ctrl+L` | retype (ganti tipe data) |
| `;` | set comment |
| `Ctrl+Shift+H` | patch instruction |
| `Ctrl+S` | simpan project |
| `Ctrl+Z` | undo |
| klik dua kali | lompat ke referensi / fungsi |

> [!NOTE]
> Shortcut adalah bawaan default dan bisa berbeda antar versi atau kalau kamu pernah
> mengubah key binding. Semua aksi yang sama selalu tersedia lewat menu klik kanan.

## 10. Rekap

| Konsep | Inti yang harus kamu ingat |
| :--- | :--- |
| Kenapa Ghidra | analisis manual tidak skala untuk ribuan fungsi |
| Disassembler vs decompiler | memetakan instruksi vs menebak kode mirip C |
| Pseudo-code | tebakan, bukan source asli; nama & tipe bisa salah |
| Project | menyimpan analisismu; binary asli tidak tersentuh |
| Import summary | ELF header (PART 3) yang dibaca otomatis |
| Tiga panel | Symbol Tree (daftar isi), Listing (assembly), Decompiler (pseudo-C) |
| Nama otomatis | `FUN_`, `LAB_`, `DAT_`, `local_`, `undefined8` |
| XREF | jalur mundur: string → siapa yang memakai → fungsi inti |
| Refactoring | rename/retype/comment = dokumentasi, bukan modifikasi binary |
| Patching | satu-satunya aksi yang mengubah byte; export untuk menulis ke file |
| Batas statis | tidak ada nilai runtime → butuh Fase 5 |

### Cek dirimu

Kamu siap lanjut ke Fase 5 jika bisa menjawab:

- [ ] Apa beda disassembler dan decompiler, dan kenapa pseudo-code bukan source asli?
- [ ] Dari mana Ghidra mendapatkan Language ID `x86:LE:64:default`?
- [ ] Apa fungsi Symbol Tree, Listing, dan Decompiler?
- [ ] Bagaimana menemukan fungsi utama di binary stripped tanpa menebak alamat?
- [ ] Apa arti `FUN_00101165`, `local_58`, dan `undefined8`?
- [ ] Kenapa `sha256sum` binary tidak berubah setelah kamu me-rename semua variabel?
- [ ] Kenapa Ghidra menulis `JNZ` padahal `objdump` menulis `jne`?
- [ ] Informasi apa yang **tidak bisa** diberikan analisis statis?

## 11. Cliffhanger: ketika decompiler menebak salah

Ghidra terlihat sangat yakin. Tapi perhatikan yang tidak bisa ia ketahui:

- Isi register **saat program berjalan** — ia hanya melihat kode, bukan eksekusi.
- Nilai yang baru muncul saat runtime: hasil dekripsi, data dari jaringan, input pengguna.
- Tipe data sebenarnya, kalau kode dioptimasi (`-O2`, PART 0 Latihan 7) atau dikaburkan.
- Apakah sebuah cabang benar-benar pernah dieksekusi.

Contoh nyata: pseudo-C menampilkan `strcmp(input, secret)`, tetapi kalau `secret` diisi hasil
dekripsi saat runtime, Ghidra hanya akan menampilkan `FUN_001012a0(...)` — dan kamu tidak akan
tahu isinya dari file saja.

> [!IMPORTANT]
> Itulah **Fase 5 — Dynamic Analysis**: menjalankan program di `gdb`, memasang breakpoint di
> alamat yang sudah kamu temukan di Ghidra, dan membaca nilai aslinya. Statis memberi peta,
> dinamis memberi kebenaran.

> [!NOTE]
> Kalau programnya sengaja dibuat sulit dianalisis — **obfuscation** (kode dikaburkan),
> **packing** (kode dikompres/dienkripsi), dan **anti-analysis** (mendeteksi debugger) —
> itu dibahas di **Fase 8**, bukan sekarang. `vault` kita tidak punya proteksi apa pun.

## 12. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| Ghidra | framework RE gratis & open-source dari NSA: disassembler + decompiler |
| Disassembler | menerjemahkan machine code menjadi assembly |
| Decompiler | menebak kode tingkat tinggi (mirip C) dari assembly |
| Pseudo-code | hasil decompiler: mirip C, tapi tebakan, bukan source asli |
| Project | database tempat Ghidra menyimpan hasil analisismu |
| Code Browser | jendela kerja utama untuk satu binary |
| Auto-analysis | proses otomatis memetakan fungsi, data, dan referensi |
| Symbol Tree | panel daftar fungsi, import, label, dan tipe data |
| Listing | panel assembly per alamat, lengkap dengan byte mentah |
| Decompiler window | panel pseudo-C dari fungsi yang dipilih |
| Defined Strings | daftar string yang dikenali beserta alamatnya |
| XREF (cross-reference) | daftar "siapa memakai alamat ini" |
| Function Graph | diagram blok percabangan satu fungsi |
| Call Graph | diagram siapa memanggil siapa |
| Image base | alamat basis tempat binary dipetakan di Ghidra (default `0x00100000`) |
| `FUN_` | nama otomatis fungsi yang belum diberi nama |
| `LAB_` | nama otomatis label (target loncatan) |
| `DAT_` | nama otomatis data yang belum diberi nama |
| `local_` | variabel lokal, dinamai dari offset stack |
| `param_` | argumen fungsi |
| `undefined8` | tipe belum diketahui berukuran 8 byte |
| Rename | mengganti nama fungsi/variabel agar mudah dibaca |
| Retype | memperbaiki tipe data hasil tebakan decompiler |
| Refactoring | merapikan nama & struktur tanpa mengubah perilaku |
| Patch Instruction | mengganti instruksi (byte) langsung di Ghidra |
| Export Program | menulis hasil perubahan ke file baru |
| Stripped | binary tanpa tabel simbol (PART 3) |
| Headless analyzer | mode Ghidra tanpa GUI untuk otomatisasi |
| Obfuscation | mengaburkan kode agar sulit dibaca (Fase 8) |
| Packing | mengompresi/menyembunyikan kode asli di dalam file (Fase 8) |
| Anti-analysis | teknik mendeteksi debugger/sandbox lalu mengelak (Fase 8) |

---

[← PART 3](../PART3/note.id.md) · [Peta belajar](../INDEX.id.md) · **Selanjutnya:** PART 5 —
Analisis Dinamis ([catatan](../PART5/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>