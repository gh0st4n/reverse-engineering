# PART 3 — Format Executable (Executable Formats)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 3 dari 10**

> [!NOTE]
> **Sumber:** catatan mandiri mengikuti [peta belajar](../INDEX.id.md). Video seri "PART 3"
> ([NefasNimlIk](https://www.youtube.com/watch?v=NefasNimlIk)) sebenarnya membahas
> `cmp`/`test`/jump dan patching — materi itu ada di [PART 2](../PART2/note.id.md).
> Istilah teknis tetap ditulis dalam bahasa aslinya, tetapi selalu dijelaskan dengan bahasa
> sederhana. Kamus istilah: [bagian 12](#12-kamus-istilah).

**Prasyarat:** [PART 2](../PART2/note.id.md) selesai — kamu sudah bisa membaca disassembly dan
pernah mem-patch byte dengan `dd`.

**Setelah part ini kamu bisa:** membaca ELF header, membedakan section dan segment, tahu di
section mana kode/string/nama fungsi tinggal, **menghitung offset file dari alamat memori**
(tanpa menebak), menjelaskan efek `strip`, dan mengenali struktur dasar PE (Windows).

## Daftar isi

- [PART 3 — Format Executable (Executable Formats)](#part-3--format-executable-executable-formats)
  - [Daftar isi](#daftar-isi)
  - [1. Kenapa format file?](#1-kenapa-format-file)
  - [2. Header ELF: 64 byte pertama](#2-header-elf-64-byte-pertama)
  - [3. Sections dan segments](#3-sections-dan-segments)
  - [4. Section yang wajib dikenali](#4-section-yang-wajib-dikenali)
  - [5. Alamat memori vs offset file](#5-alamat-memori-vs-offset-file)
    - [PIE vs non-PIE](#pie-vs-non-pie)
  - [6. Simbol dan strip](#6-simbol-dan-strip)
  - [7. Dynamic linking: PLT dan GOT](#7-dynamic-linking-plt-dan-got)
  - [8. PE (Windows) secara singkat](#8-pe-windows-secara-singkat)
  - [9. Praktik: membaca ELF](#9-praktik-membaca-elf)
    - [9.1 Siapkan target](#91-siapkan-target)
    - [9.2 Lab 1 — Terjemahkan header dengan tangan](#92-lab-1--terjemahkan-header-dengan-tangan)
    - [9.3 Lab 2 — Temukan rumah rahasia](#93-lab-2--temukan-rumah-rahasia)
    - [9.4 Lab 3 — Buktikan rumus alamat ke offset](#94-lab-3--buktikan-rumus-alamat-ke-offset)
    - [9.5 Lab 4 — Nama yang hilang (dan yang bertahan)](#95-lab-4--nama-yang-hilang-dan-yang-bertahan)
  - [10. Rekap](#10-rekap)
    - [Cek dirimu](#cek-dirimu)
  - [11. Cliffhanger: dari file ke kode C](#11-cliffhanger-dari-file-ke-kode-c)
  - [12. Kamus istilah](#12-kamus-istilah)

---

## 1. Kenapa format file?

Di PART 2 kita mem-patch `vault` dengan rumus `offset = alamat - Addr + Off`, dan kebetulan
`Addr` dan `Off` sama. Empat pertanyaan yang belum terjawab:

- Kenapa alamat memori dan posisi di file bisa berbeda?
- Dari mana `objdump` tahu fungsi itu bernama `add`?
- Di mana `sup3rs3cr3t` sebenarnya disimpan?
- Apa isi 60 byte setelah magic `7f 45 4c 46`?

Semua jawabannya ada di **kepala file**.

> [!TIP]
> Analogi: binary adalah sebuah gedung. **Header** = denah di lobi (pintu masuk di mana, ada
> berapa lantai). **Section** = ruangan bernomor untuk tukang bangunan (dapur, gudang, kantor).
> **Segment** = blok yang benar-benar diangkut petugas saat gedung dipakai. Ruangan yang
> berdekatan dan seizinkan bisa diangkut sebagai satu blok.

File executable bukan tumpukan kode mentah. Ia membawa peta yang memberi tahu OS: mana kode,
mana data, mulai jalan dari mana, dan library apa yang harus disambungkan.

## 2. Header ELF: 64 byte pertama

**ELF** (Executable and Linkable Format) adalah format executable di Linux. 64 byte pertamanya
adalah **ELF header** (`Elf64_Ehdr`).

```bash
xxd -l 64 vault
```

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 6010 0000 0000 0000  ..>.....`.......
00000020: 4000 0000 0000 0000 0000 0000 0000 0000  @...............
00000030: 0000 0000 4000 3800 0d00 4000 1f00 1e00  ....@.8...@.....
```

Cara membaca 16 byte pertama (`e_ident`):

| Offset | Byte | Arti |
| :---: | :--- | :--- |
| 0–3 | `7f 45 4c 46` | magic: `\x7fELF` (LAB PART 0) |
| 4 | `02` | kelas: `01` = 32-bit, `02` = **64-bit** |
| 5 | `01` | endianness: `01` = **little-endian**, `02` = big |
| 6 | `01` | versi ELF |
| 7 | `00` | OS/ABI: `00` = System V |
| 8–15 | `00 ...` | padding |

Sisanya adalah angka **little-endian** (PART 0 Latihan 6 — makanya `3e00` dibaca `0x003e`):

| Offset | Ukuran | Field | Contoh | Arti |
| :---: | :---: | :--- | :--- | :--- |
| 16 | 2 | `e_type` | `0300` | `02` = EXEC (alamat tetap), `03` = **DYN** (PIE) |
| 18 | 2 | `e_machine` | `3e00` | `0x3e` = 62 = **x86-64** |
| 24 | 8 | `e_entry` | `6010...` | alamat mulai eksekusi = `0x1060` |
| 32 | 8 | `e_phoff` | `4000...` | posisi **program header** di file = 64 |
| 40 | 8 | `e_shoff` | — | posisi **section header** di file |
| 52 | 2 | `e_ehsize` | `4000` | ukuran ELF header = 64 byte |
| 56 | 2 | `e_phnum` | `0d00` | jumlah program header = 13 |
| 60 | 2 | `e_shnum` | `1f00` | jumlah section header = 31 |

Versi manusianya:

```bash
readelf -h vault
```

```text
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Entry point address:               0x1060
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13624 (bytes into file)
  Number of program headers:         13
  Number of section headers:         31
```

> [!IMPORTANT]
> **Entry point bukan `main`.** Yang dijalankan pertama adalah `_start` (kode startup dari
> libc), yang menyiapkan runtime lalu baru memanggil `main`. Kalau kamu mulai membaca dari
> entry point, kamu akan membaca kode startup, bukan logika program.

> [!NOTE]
> `Type: DYN` bukan berarti file ini library. Binary modern dikompilasi sebagai **PIE**
> (Position-Independent Executable) agar bisa diacak ASLR. Bandingkan dengan `Type: EXEC`
> di Lab 3.

## 3. Sections dan segments

File ELF punya **dua peta untuk dua pembaca berbeda**:

| | Sections | Segments |
| :--- | :--- | :--- |
| Untuk siapa | linker & tool analisis | **loader** OS saat menjalankan |
| Perintah | `readelf -S` | `readelf -l` |
| Isi | daftar ruangan bernomor (`.text`, `.rodata`, ...) | daftar blok yang dimuat ke memori |
| Jumlah | banyak (puluhan) | sedikit (biasanya 4 blok `LOAD`) |

```text
   SECTION (peta tukang)              SEGMENT (peta pengangkut)
   .interp .dynsym .dynstr .rela  ──▶  LOAD  R      (baca saja)
   .init .plt .text .fini         ──▶  LOAD  R E    (baca + jalankan)
   .rodata .eh_frame              ──▶  LOAD  R      (baca saja)
   .data .bss .got                ──▶  LOAD  RW     (baca + tulis)
```

```bash
readelf -SW vault      # -W = wide, satu baris per section (dipakai di PART 2)
readelf -lW vault
```

```text
  Type   Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg  Align
  LOAD   0x000000 0x0000000000000000 0x0000000000000000 0x000628 0x000628 R    0x1000
  LOAD   0x001000 0x0000000000001000 0x0000000000001000 0x0001d5 0x0001d5 R E  0x1000
  LOAD   0x002000 0x0000000000002000 0x0000000000002000 0x0001ac 0x0001ac R    0x1000
  LOAD   0x002db8 0x0000000000003db8 0x0000000000003db8 0x000258 0x000260 RW   0x1000
```

Perhatikan hubungannya dengan peta memori PART 1: `R E` → Text segment, `RW` → Data/BSS.
Loader tidak peduli nama section; ia hanya membaca izin (`Flg`) dan menyalin blok.

> [!IMPORTANT]
> Lihat baris terakhir: `FileSiz 0x258` tetapi `MemSiz 0x260`. Selisih 8 byte itu adalah
> **`.bss`** — variabel global tanpa nilai awal. Ia **tidak memakan ruang di file** (tipe
> section `NOBITS`), tapi tetap dialokasikan nol di memori. Itulah sebabnya file bisa lebih
> kecil daripada jejak memorinya.

## 4. Section yang wajib dikenali

| Section | Isi | Kenapa penting untuk RE |
| :--- | :--- | :--- |
| `.text` | kode mesin | target utama disassembly & patching |
| `.rodata` | konstanta & string literal (**read-only**) | tempat password hardcoded tinggal |
| `.data` | global/statis yang punya nilai awal | konfigurasi, variabel yang bisa berubah |
| `.bss` | global/statis tanpa nilai awal | tidak ada di file, hanya di memori |
| `.plt` / `.got` | jembatan ke fungsi library | menjelaskan `call strcmp@plt` |
| `.dynsym` / `.dynstr` | simbol & nama untuk dynamic linking | **tetap ada** walau di-`strip` |
| `.symtab` / `.strtab` | tabel simbol & nama lengkap | sumber nama `add`, `main` di `objdump` |
| `.rela.dyn` / `.rela.plt` | catatan relokasi | di mana alamat harus ditambal loader |
| `.init` / `.fini` | kode sebelum/sesudah `main` | trik menyembunyikan logika (Fase 8) |
| `.comment` | versi compiler | fingerprint cepat: GCC versi berapa |
| `.debug_*` | info debug (DWARF), hanya jika `-g` | nama variabel lokal & nomor baris |

> [!TIP]
> Urutan kerja cepat pada binary asing: `readelf -h` (jenis & arsitektur) → `readelf -SW`
> (section apa saja) → `strings -t x` (cari teks menarik + offsetnya) → `nm` (masih ada nama
> fungsi?) → baru `objdump`.

## 5. Alamat memori vs offset file

Ada **dua sistem koordinat** yang tidak boleh tertukar:

| | Offset file | Alamat virtual (VMA) |
| :--- | :--- | :--- |
| Dihitung dari | byte ke-0 **file** | basis pemuatan di **memori** |
| Dipakai oleh | `xxd`, `dd`, hex editor | `objdump`, `gdb` |
| Contoh | `0x11b6` | `0x5555555551b6` (saat jalan) |

Rumus konversinya, memakai data section:

```text
offset file = alamat - sh_addr + sh_offset
              (Addr)         (Off)      ← dua kolom dari readelf -SW
```

Atau memakai segment (lebih tepat untuk yang benar-benar dimuat):

```text
offset file = alamat - p_vaddr + p_offset
```

Kenapa di PART 2 keduanya kebetulan sama? Karena loader mensyaratkan `p_offset` dan `p_vaddr`
**kongruen modulo ukuran halaman** (`0x1000`). Pada binary kecil, hasilnya sering
`Addr == Off` untuk `.text` — tapi itu kebetulan yang harus **dibuktikan**, bukan diasumsikan.

### PIE vs non-PIE

| | `-no-pie` (EXEC) | default modern (DYN / PIE) |
| :--- | :--- | :--- |
| Alamat saat jalan | tetap, mis. `0x401136` | basis acak + offset, mis. `0x555555555136` |
| ASLR | tidak mengacak kode | mengacak seluruh binary |
| Contoh di catatan | `sub_401136` di PART 0 | `0x5555555551b6` di PART 1 |

> [!IMPORTANT]
> Untuk mem-patch **file**, selalu pakai **offset file**, tidak pernah alamat runtime.Alamat
> `0x5555555551b6` di gdb tidak ada artinya bagi `dd`; yang dipakai adalah `0x11b6`.

## 6. Simbol dan strip

**Simbol** adalah nama yang dikaitkan dengan alamat: fungsi (`main`, `add`) atau variabel
global. disimpan di `.symtab`, dan teks namanya di `.strtab`.

```bash
nm vault | grep ' T '     # T = simbol kode (text) yang global
readelf -sW vault | head -20
```

```text
0000000000001135 T add
000000000000114f T main
```

`strip` membuang tabel itu:

```bash
cp vault vault.stripped
strip --strip-all vault.stripped
nm vault.stripped
```

```text
nm: vault.stripped: no symbols
```

> [!IMPORTANT]
> Ini **memperjelas** ucapan PART 0 bahwa "compiler membuang nama". Lebih tepatnya:
> - kode mesin memang tidak memakai nama — compiler hanya menghasilkan alamat & offset;
> - **nama fungsi masih bisa tersimpan** di `.symtab` sampai file di-`strip`;
> - **nama variabel lokal** hanya ada kalau dikompilasi dengan `-g` (info debug DWARF);
> - nama fungsi library (`strcmp`) tetap ada di `.dynsym` walau di-strip, karena dynamic
>   linking membutuhkannya saat runtime.

Itulah sebabnya `objdump` masih menampilkan `<strcmp@plt>` pada binary stripped, tetapi
`<main>` berubah menjadi alamat polos seperti `0x114f`.

## 7. Dynamic linking: PLT dan GOT

`vault` tidak berisi kode `strcmp`. Kode itu ada di **libc**, library bersama yang dimuat saat
program jalan.

```bash
readelf -dW vault | grep NEEDED
ldd vault
```

```text
 0x0000000000000001 (NEEDED)  Shared library: [libc.so.6]
	linux-vdso.so.1 (0x00007ffd...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

Rantai `call 1050 <strcmp@plt>`:

```text
call .plt  ──▶  jmp [GOT]  ──▶  alamat strcmp di libc
   (stub)        (tabel pointer, diisi loader)
```

- **PLT** (Procedure Linkage Table) = potongan kode kecil di `.plt`, satu per fungsi library.
- **GOT** (Global Offset Table) = tabel pointer di `.got`; loader menuliskan alamat asli ke
  sini. Defaultnya *lazy binding*: diisi saat pertama kali dipanggil.

> [!WARNING]
> `ldd` menjalankan dynamic loader, jadi pada binary **tidak terpercaya** ia bisa mengeksekusi
> kode. Untuk sampel malware, pakai `readelf -d` atau `objdump -p` yang hanya membaca file.

> [!TIP]
> GOT yang bisa ditulis adalah alasan teknik seperti *GOT overwrite* ada. Simpan istilah ini;
> relevan di Fase 5 dan 9.

## 8. PE (Windows) secara singkat

Windows memakai **PE** (Portable Executable). Konsepnya sama, nama field-nya berbeda:

| Konsep | ELF (Linux) | PE (Windows) |
| :--- | :--- | :--- |
| Magic | `7f 45 4c 46` (`.ELF`) | `4d 5a` (`MZ`) lalu `50 45 00 00` (`PE\0\0`) |
| Header | ELF header (64 byte) | DOS header → `e_lfanew` → PE header |
| Mulai eksekusi | `e_entry` | `AddressOfEntryPoint` (RVA) |
| Basis alamat | acak jika PIE | `ImageBase` (exe biasanya `0x140000000`) |
| Kode | `.text` | `.text` |
| Konstanta/string | `.rodata` | `.rdata` |
| Data global | `.data` / `.bss` | `.data` |
| Resource | — | `.rsrc` (ikon, dialog, versi) |
| Relokasi | `.rela.*` | `.reloc` |

Alamat di PE disebut **RVA** (Relative Virtual Address) — relatif terhadap `ImageBase`:

```text
alamat runtime = ImageBase + RVA
offset file    = RVA - VirtualAddress + PointerToRawData
```

Rumus kedua itu **persis** rumus ELF di bagian 5, hanya beda nama kolom.

```bash
objdump -x sample.exe | head -40      # bisa untuk PE juga
```

Tool lain: `pev`/`pedump` (Linux), CFF Explorer atau Detect It Easy (Windows).

> [!NOTE]
> Lab di part ini memakai ELF karena lab kita Linux. Pola pikirnya identik untuk PE: baca
> header → temukan section → konversi alamat → baru patch.

## 9. Praktik: membaca ELF

### 9.1 Siapkan target

```bash
gcc -g -O0 vault.c -o vault
gcc -g -O0 frame.c -o frame
cp vault vault.bak
```

> [!NOTE]
> Blok `bash` = terminal, `text` = output. Semua angka di bawah adalah **contoh**; hitung ulang
> di mesinmu, dan selalu verifikasi dengan `xxd` sebelum menulis apa pun.

### 9.2 Lab 1 — Terjemahkan header dengan tangan

**Tujuan:** membuktikan bahwa "magic bytes" PART 0 hanyalah awal dari sebuah struktur.

```bash
readelf -h vault
xxd -l 16 vault
```

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
```

Cocokkan sendiri dengan tabel di bagian 2, lalu jawab: 32 atau 64 bit? Little atau big endian?
EXEC atau DYN? Berapa section-nya? Semua sudah tertulis di 64 byte itu.

> [!IMPORTANT]
> **THE AHA:** Header bukan sihir — ia hanya struct C yang ditulis mentah ke file. Kalau kamu
> bisa membaca tabel offset, kamu bisa membaca format apa pun (ELF, PE, PNG, ZIP).

### 9.3 Lab 2 — Temukan rumah rahasia

**Tujuan:** tahu persis di mana `sup3rs3cr3t` tinggal, bukan cuma "ketemu di strings".

```bash
strings -t x vault | grep sup3rs3cr3t
readelf -SW vault | grep -E '\.rodata|\.text'
```

```text
   2010 sup3rs3cr3t
  [16] .rodata PROGBITS 0000000000002000 002000 000042 00   A  0  0  4
```

`strings -t x` mencetak **offset file** dalam hex. Offset `0x2010` jatuh di dalam `.rodata`
yang mulai `0x2000` dan berukuran `0x42` → cocok. Lihat isinya:

```bash
readelf -p .rodata vault
xxd -s 0x2010 -l 16 vault
```

```text
String dump of section '.rodata':
  [     4]  Enter the vault password:
  [    20]  sup3rs3cr3t
  [    2c]  %63s
  [    31]  ACCESS GRANTED
  [    40]  ACCESS DENIED
00002010: 7375 7033 7273 3363 7233 7400 2536 3373  sup3rs3cr3t.%63s
```

Perhatikan `00` setelah `sup3rs3cr3t`: itu **null terminator**, penanda akhir string di C.

> [!IMPORTANT]
> **THE AHA:** `strings` bukan alat ajaib — ia hanya menyapu file mencari deretan karakter yang
> bisa dicetak. Rahasia itu ada karena compiler menaruh string literal apa adanya di `.rodata`.
> Semua teks patchable di PART 0 (`ACCESS DENIED`) juga tinggal di sini.

### 9.4 Lab 3 — Buktikan rumus alamat ke offset

**Tujuan:** berhenti menebak offset saat patching, dan melihat efek PIE.

```bash
gcc -g -O0 -no-pie vault.c -o vault.nopie
readelf -h vault.nopie | grep -E 'Type|Entry'
objdump -d -M intel vault.nopie | grep -A3 'strcmp@plt'
readelf -SW vault.nopie | grep '\.text'
```

```text
  Type:                              EXEC (Executable file)
  Entry point address:               0x401050
    401196:	e8 c5 fe ff ff       	call   401030 <strcmp@plt>
    40119b:	85 c0                	test   eax,eax
    40119d:	75 0e                	jne    4011ad <main+0x61>
  [14] .text PROGBITS 0000000000401050 001050 0001a2 00  AX  0  0  16
```

Hitung: `offset = 0x40119d - 0x401050 + 0x1050 = 0x119d`. Buktikan sebelum menulis:

```bash
xxd -s 0x119d -l 2 vault.nopie
```

```text
0000119d: 750e                                     u.
```

Kalau `750e` muncul, rumusnya terbukti. Bandingkan dengan versi PIE:

```bash
readelf -h vault | grep Type
```

```text
  Type:                              DYN (Position-Independent Executable file)
```

> [!NOTE]
> Alamat `0x401...` pada non-PIE inilah yang menjelaskan `sub_401136` di contoh PART 0. Angka
> itu bukan konstanta universal — ia ciri binary EXEC tanpa ASLR.

> [!IMPORTANT]
> **THE AHA:** `Addr` dan `Off` sering sama, tapi **alasannya** adalah kongruensi halaman, bukan
> kebetulan. Dengan rumus ini kamu bisa mem-patch binary apa pun, termasuk yang alamat dan
> offsetnya berbeda jauh.

### 9.5 Lab 4 — Nama yang hilang (dan yang bertahan)

**Tujuan:** melihat sendiri apa yang dibuang `strip`, dan apa yang tetap bertahan.

```bash
nm frame | grep ' T '
objdump -d -M intel frame | grep -E '^[0-9a-f]+ <' | head -5
cp frame frame.stripped
strip --strip-all frame.stripped
nm frame.stripped
objdump -d -M intel frame.stripped | grep -E '^[0-9a-f]+ <' | head -5
ls -l frame frame.stripped
```

```text
0000000000001135 T add
000000000000114f T main
0000000000001135 <add>:
000000000000114f <main>:
nm: frame.stripped: no symbols
0000000000001040 <printf@plt>:
-rwxr-xr-x 1 user user 17K frame
-rwxr-xr-x 1 user user 14K frame.stripped
```

Fungsi `add` tetap ada — kodenya tidak berubah sebyte pun. Yang hilang hanya **label namanya**.

> [!IMPORTANT]
> **THE AHA:** Stripping bukan menghapus logika, hanya menghapus peta nama. Reverse engineer
> tetap bisa membaca fungsinya, lalu **memberi nama sendiri** (`sub_401136` → `verify_password`).
> Itu pekerjaan sehari-hari di Fase 4 dan 6.

## 10. Rekap

| Konsep | Inti yang harus kamu ingat |
| :--- | :--- |
| ELF header | 64 byte: magic, kelas, endianness, EXEC/DYN, entry point, posisi tabel |
| Entry point | `_start`, bukan `main` |
| Sections | peta untuk linker/tool: `.text`, `.rodata`, `.data`, `.bss`, `.symtab` |
| Segments | peta untuk loader: blok `LOAD` dengan izin `R` / `R E` / `RW` |
| `.bss` | `NOBITS`: tidak makan ruang file, ada di memori (`MemSiz > FileSiz`) |
| Dua koordinat | offset file (`xxd`/`dd`) vs alamat virtual (`objdump`/`gdb`) |
| Konversi | `offset = alamat - Addr + Off` (section) atau `- p_vaddr + p_offset` (segment) |
| PIE vs EXEC | PIE = alamat diacak ASLR; EXEC (`-no-pie`) = alamat tetap `0x401...` |
| Simbol | `.symtab` dibuang `strip`; `.dynsym` bertahan → `strcmp@plt` masih terlihat |
| PLT/GOT | jembatan ke library; GOT diisi loader saat runtime |
| PE | konsep sama: `MZ` → `PE\0\0`, RVA, `ImageBase`, `.rdata`/`.rsrc` |

### Cek dirimu

Kamu siap lanjut ke Fase 4 jika bisa menjawab:

- [ ] Apa arti byte ke-4 dan ke-5 pada ELF header?
- [ ] Kenapa entry point bukan `main`, dan apa yang ada di sana?
- [ ] Apa beda section dan segment, dan siapa yang membaca masing-masing?
- [ ] Kenapa `MemSiz` bisa lebih besar daripada `FileSiz`?
- [ ] Di section mana string literal tinggal, dan kenapa `strings` bisa menemukannya?
- [ ] Bagaimana mengubah alamat `0x40119d` menjadi offset file — dan kenapa tidak boleh
      memakai alamat runtime untuk `dd`?
- [ ] Apa yang hilang dan apa yang bertahan setelah `strip`?
- [ ] Apa peran PLT dan GOT saat program memanggil `strcmp`?

## 11. Cliffhanger: dari file ke kode C

Sekarang kamu punya peta lengkap: header, section, simbol, dan cara menghitung offset. Tapi
semua analisis masih dilakukan **baris demi baris**, manual.

Pertanyaan berikutnya: kalau nama fungsi sudah hilang dan programnya punya 500 fungsi, apakah
kita harus membaca semuanya satu per satu?

Tidak. Ada tool yang membaca struktur ELF ini untuk kita, memetakan seluruh fungsi, membangun
grafik alur, lalu **menebak kembali kode C**-nya.

> [!IMPORTANT]
> Itulah **Fase 4 — Static Analysis** dengan Ghidra: dari membaca byte menjadi membaca
> *decompiled C*, dan memberi nama ulang pada fungsi yang sudah kehilangan namanya.

## 12. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| ELF | format file executable Linux (Executable and Linkable Format) |
| PE | format file executable Windows (Portable Executable) |
| Magic bytes | byte awal penanda jenis file (ELF = `7f 45 4c 46`, PE = `MZ`) |
| ELF header | 64 byte pertama: identitas file + posisi tabel-tabelnya |
| `e_type` | jenis executable: `EXEC` (alamat tetap) atau `DYN` (PIE) |
| Entry point | alamat pertama yang dijalankan (`_start`, bukan `main`) |
| Section | "ruangan" bernama di file, untuk linker & tool analisis |
| Segment | blok yang benar-benar dimuat loader ke memori, dengan izin akses |
| `LOAD` | tipe segment yang isinya disalin ke memori |
| `.text` | section berisi kode mesin |
| `.rodata` | section konstanta & string literal, read-only |
| `.data` / `.bss` | global yang punya nilai awal / yang belum (nol) |
| `NOBITS` | tipe section yang tidak menyimpan byte di file (`.bss`) |
| `FileSiz` / `MemSiz` | ukuran di file / ukuran di memori |
| `.symtab` / `.strtab` | tabel simbol dan teks namanya |
| `.dynsym` / `.dynstr` | simbol untuk dynamic linking, bertahan walau di-strip |
| Simbol | nama yang dikaitkan dengan alamat (fungsi/variabel) |
| `nm` | tool daftar simbol sebuah binary |
| `strip` | menghapus tabel simbol & info debug dari file |
| Stripped | binary tanpa tabel simbol |
| DWARF / `.debug_*` | info debug (`-g`): nama variabel lokal & nomor baris |
| Relokasi | catatan alamat yang harus ditambal loader saat memuat |
| PLT | Procedure Linkage Table: stub kode pemanggil fungsi library |
| GOT | Global Offset Table: tabel pointer yang diisi loader |
| Lazy binding | pengisian GOT ditunda sampai fungsi pertama kali dipanggil |
| Dynamic linking | menyambungkan ke library saat runtime, bukan saat kompilasi |
| `ldd` | melihat library yang dibutuhkan (hati-hati pada file tak terpercaya) |
| PIE | binary yang bisa dimuat di alamat acak (mendukung ASLR) |
| ASLR | pengacakan alamat memori oleh OS |
| Offset file | posisi byte dihitung dari awal file |
| VMA | alamat virtual saat program berjalan |
| RVA | alamat relatif terhadap `ImageBase` (istilah PE) |
| `ImageBase` | alamat basis pemuatan PE |
| `.rsrc` / `.reloc` | section resource / relokasi pada PE |
| `readelf` | tool membaca header, section, segment, simbol ELF |

---

[← PART 2](../PART2/note.id.md) · [Peta belajar](../INDEX.id.md) · **Selanjutnya:** PART 4 —
Analisis Statis ([catatan](../PART4/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>