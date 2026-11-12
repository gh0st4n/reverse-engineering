# PART 5 — Analisis Dinamis (Dynamic Analysis)

[English](note.md) · **Bahasa Indonesia**

[← README](../README.id.md) · [Peta belajar](../INDEX.id.md) · **Fase 5 dari 10**

> [!NOTE]
> **Sumber:** catatan mandiri mengikuti [peta belajar](../INDEX.id.md). Video yang beredar
> dengan label "PART 5" ([iuYRHUH51yA](https://www.youtube.com/watch?v=iuYRHUH51yA)) membahas
> **packing** — materi itu ada di [PART 8](../PART8/note.id.md).
> Kamus istilah: [bagian 9](#9-kamus-istilah).

**Prasyarat:** [PART 1](../PART1/note.id.md) (dasar gdb) dan [PART 4](../PART4/note.id.md)
(Ghidra, XREF, image base).

**Setelah part ini kamu bisa:** menerjemahkan alamat Ghidra menjadi alamat runtime, memakai
watchpoint untuk menangkap perubahan memori, membaca nilai yang mustahil dilihat statis, dan
memakai `ltrace`/`strace` untuk triage cepat.

## Daftar isi

- [PART 5 — Analisis Dinamis (Dynamic Analysis)](#part-5--analisis-dinamis-dynamic-analysis)
  - [Daftar isi](#daftar-isi)
  - [1. Kenapa dinamis?](#1-kenapa-dinamis)
  - [2. Toolkit dinamis dan pembagian tugas](#2-toolkit-dinamis-dan-pembagian-tugas)
  - [3. Menjembatani alamat: Ghidra ke gdb](#3-menjembatani-alamat-ghidra-ke-gdb)
  - [4. Perintah gdb yang belum dipakai](#4-perintah-gdb-yang-belum-dipakai)
  - [5. Praktik](#5-praktik)
    - [5.1 Lab 1 — Terjemahkan alamat Ghidra menjadi breakpoint](#51-lab-1--terjemahkan-alamat-ghidra-menjadi-breakpoint)
    - [5.2 Lab 2 — Baca nilai yang tidak ada di file](#52-lab-2--baca-nilai-yang-tidak-ada-di-file)
    - [5.3 Lab 3 — Watchpoint: siapa yang menimpa return address?](#53-lab-3--watchpoint-siapa-yang-menimpa-return-address)
  - [6. Batas analisis dinamis](#6-batas-analisis-dinamis)
  - [7. Rekap](#7-rekap)
    - [Cek dirimu](#cek-dirimu)
  - [8. Cliffhanger: dari nilai ke pola](#8-cliffhanger-dari-nilai-ke-pola)
  - [9. Kamus istilah](#9-kamus-istilah)

---

## 1. Kenapa dinamis?

PART 4 ditutup dengan pertanyaan yang tidak bisa dijawab Ghidra: **apa isi variabel ini saat
program berjalan?** Analisis statis melihat kode, bukan eksekusi.

| Pertanyaan | Statis | Dinamis |
| :--- | :---: | :---: |
| Fungsi apa saja yang ada? | ✅ | ⚠️ hanya yang terpanggil |
| Siapa memanggil siapa? | ✅ | ✅ |
| Apa isi register **sekarang**? | ❌ | ✅ |
| Apa hasil dekripsi runtime? | ❌ | ✅ |
| Cabang mana yang benar-benar dijalankan? | ❌ | ✅ |
| File/jaringan apa yang disentuh? | ⚠️ tebakan | ✅ |

> [!TIP]
> Analogi PART 0 dipakai lagi: statis = membaca denah rumah; dinamis = masuk dan melihat
> penghuninya. Ghidra memberi denah yang jauh lebih bagus, tapi tetap denah.

## 2. Toolkit dinamis dan pembagian tugas

| Tool | Melihat apa | Kapan dipakai |
| :--- | :--- | :--- |
| `ltrace` | panggilan **library** (`strcmp`, `malloc`) | triage 10 detik: apakah ada perbandingan string? |
| `strace` | **system call** (open, read, write, connect) | triage: file apa yang dibuka, koneksi ke mana? |
| `gdb` | register, memori, alur instruksi | pertanyaan presisi di satu titik |
| `/proc/<pid>/maps` | wilayah memori proses | mencari basis pemuatan, melihat library |

Triage cepat sebelum membuka gdb:

```bash
ltrace ./vault 2>&1 | grep -E 'strcmp|memcmp|strncmp'
strace -f ./vault 2>&1 | grep -E 'openat|read|write' | head -10
```

> [!WARNING]
> `ltrace`/`strace` **menjalankan** program. Untuk sampel tidak terpercaya, kerjakan di VM
> terisolasi tanpa jaringan (PART 0). Untuk analisis tanpa eksekusi, pakai `readelf -d`
> (PART 3) atau Ghidra (PART 4).

## 3. Menjembatani alamat: Ghidra ke gdb

Ini keterampilan yang paling sering dibutuhkan sehari-hari: kamu menemukan sesuatu di Ghidra,
lalu ingin memeriksanya saat program jalan. **Angka alamatnya berbeda.**

| Sumber | Contoh alamat | Kenapa begitu |
| :--- | :--- | :--- |
| `objdump` (PART 2) | `0x11b6` | offset relatif dalam binary |
| Ghidra (PART 4) | `0x001011b6` | image base default `0x00100000` + offset |
| `gdb` saat runtime (PIE) | `0x5555555551b6` | basis acak ASLR + offset |
| `gdb` saat runtime (`-no-pie`) | `0x4011b6` | basis tetap, tanpa pengacakan |

Rumusnya:

```text
alamat runtime = basis pemuatan + (alamat Ghidra - 0x00100000)
```

> [!NOTE]
> gdb mematikan ASLR secara default (`set disable-randomization on`), jadi basis PIE sering
> terlihat `0x555555554000`. Di luar gdb, alamatnya berubah setiap kali dijalankan.

## 4. Perintah gdb yang belum dipakai

| Perintah | Gunanya |
| :--- | :--- |
| `watch *(long*)($rbp+8)` | **watchpoint**: berhenti saat nilai memori itu **berubah** |
| `rwatch` / `awatch` | berhenti saat dibaca / saat dibaca atau ditulis |
| `break *0x... if $rdi == 0` | breakpoint bersyarat |
| `catch syscall openat` | berhenti saat system call tertentu dipanggil |
| `info proc mappings` | daftar wilayah memori + basis pemuatan |
| `starti` | berhenti di instruksi paling pertama |
| `set var $eax = 0` | mengubah nilai saat runtime (PART 1) |
| `display/i $rip` | cetak instruksi berikutnya setiap berhenti |
| `find $rsp, $rsp+0x200, "sup3r"` | mencari pola di memori |
| `dump binary memory out.bin A B` | menulis isi memori ke file (dipakai di PART 8) |

> [!TIP]
> Watchpoint adalah jawaban untuk pertanyaan "siapa yang mengubah nilai ini?" — jauh lebih
> efisien daripada menebak lalu memasang puluhan breakpoint.

## 5. Praktik

### 5.1 Lab 1 — Terjemahkan alamat Ghidra menjadi breakpoint

**Tujuan:** memasang breakpoint di alamat yang kamu temukan secara statis.

Dari PART 4: keputusan `vault` ada di Ghidra `0x001011b6` → offset `0x11b6`.

```bash
gcc -g -O0 vault.c -o vault
gdb -q ./vault
```

```gdb
set disassembly-flavor intel
starti
info proc mappings
```

```text
      Start Addr       End Addr   Size  Offset  Perms  objfile
    0x555555554000   0x555555555000  0x1000    0x0  r--    /path/vault
    0x555555555000   0x555555556000  0x1000 0x1000  r-x    /path/vault
```

Basis = `0x555555554000`. Hitung: `0x555555554000 + 0x11b6 = 0x5555555551b6`.

```gdb
break *0x5555555551b6
continue
x/i $rip
```

```text
Breakpoint 1, 0x00005555555551b6 in main ()
0x5555555551b6 <main+81>:  jne    0x5555555551c6 <main+97>
```

> [!IMPORTANT]
> **THE AHA:** Ghidra dan gdb menunjuk **instruksi yang sama** dengan angka berbeda. Begitu kamu
> bisa menghitung basisnya, hasil analisis statis langsung bisa dipakai saat runtime.

### 5.2 Lab 2 — Baca nilai yang tidak ada di file

**Tujuan:** melihat data yang hanya muncul saat program berjalan.

```gdb
break strcmp
continue
x/s $rdi
x/s $rsi
info registers eax
finish
```

```text
0x7fffffffe0a0: "tebak"
0x402010:       "sup3rs3cr3t"
```

Kalau rahasianya diisi hasil dekripsi runtime, di file ia hanya berupa byte acak — tetapi di
sini ia tetap harus muncul sebagai plaintext (prinsip PART 0 Latihan 2).

> [!IMPORTANT]
> **THE AHA:** Analisis statis memberi ** peta**; analisis dinamis memberi **nilai**. Keduanya
> bukan pilihan, tapi pasangan: statis untuk menemukan tempat, dinamis untuk membacanya.

### 5.3 Lab 3 — Watchpoint: siapa yang menimpa return address?

**Tujuan:** menangkap penimpaan memori tepat saat terjadi. Pakai `vuln.c` dari
[lab Buffer Overflow](BufferOverflow/note.id.md).

```bash
gcc -g -O0 -fno-stack-protector -no-pie vuln.c -o vuln
gdb -q ./vuln
```

```gdb
set disassembly-flavor intel
break *main+0x2a
run
watch *(long*)($rbp+8)
continue
```

```text
Hardware watchpoint 2: *(long*)($rbp+8)
Old value = 0x7ffff7e0a2c0
New value = 0x4141414141414141
```

gdb berhenti **tepat pada instruksi** yang menulis ke slot return address, dan menunjukkan nilai
sebelum/sesudah.

> [!IMPORTANT]
> **THE AHA:** Watchpoint mengubah pertanyaan "di mana bug-nya?" menjadi "tunjukkan detik saat
> data ini berubah". Inilah jembatan langsung ke [lab Buffer Overflow](BufferOverflow/note.id.md).

## 6. Batas analisis dinamis

| Batas | Penjelasan |
| :--- | :--- |
| Hanya jalur yang dieksekusi | kode yang tidak terpanggil tetap tak terlihat |
| Butuh input yang tepat | cabang tersembunyi perlu input/kondisi khusus |
| Bisa dideteksi | anti-debug mendeteksi debugger lalu berbohong (Fase 8) |
| Berisiko | kode benar-benar berjalan → wajib VM terisolasi |
| Tidak lengkap sendiri | tanpa peta statis, kamu tidak tahu harus berhenti di mana |

> [!NOTE]
> Karena itu alur kerja nyata selalu bolak-balik: Ghidra → gdb → Ghidra. Setiap putaran
> menambah nama, komentar, dan pemahaman.

## 7. Rekap

| Konsep | Inti yang harus kamu ingat |
| :--- | :--- |
| Statis vs dinamis | peta vs nilai; dipakai berpasangan |
| Triage | `ltrace` (library), `strace` (system call) sebelum gdb |
| Alamat | runtime = basis + (alamat Ghidra − `0x00100000`) |
| Basis | dari `info proc mappings`; gdb mematikan ASLR by default |
| Watchpoint | `watch *(long*)(addr)` → berhenti saat nilai berubah |
| Conditional break | `break *addr if kondisi` |
| `catch syscall` | berhenti pada system call tertentu |
| Batas | hanya jalur yang jalan, bisa dideteksi, berisiko |

### Cek dirimu

- [ ] Informasi apa yang hanya bisa didapat saat runtime?
- [ ] Bagaimana menghitung alamat runtime dari alamat Ghidra?
- [ ] Kenapa alamat di gdb stabil padahal di luar berubah?
- [ ] Kapan memakai watchpoint, bukan breakpoint?
- [ ] Apa beda `ltrace` dan `strace`, dan kapan masing-masing dipakai?
- [ ] Kenapa analisis dinamis tetap butuh analisis statis?

## 8. Cliffhanger: dari nilai ke pola

Sekarang kamu bisa menemukan tempat (statis) dan membaca nilainya (dinamis). Yang masih mahal
adalah **volume**: membaca 200 fungsi satu per satu tetap melelahkan.

Jalan keluarnya bukan tool baru, tapi **pengenalan pola**: compiler menghasilkan bentuk yang
berulang untuk `if`, loop, `switch`, dan pemanggilan fungsi. Begitu kamu mengenalinya, kamu
bisa melewati boilerplate dan langsung menuju inti.

> [!IMPORTANT]
> Itulah **Fase 6 — Reversing Patterns**.

## 9. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| Dynamic analysis | menganalisis program saat berjalan |
| Runtime value | nilai asli yang hanya ada saat eksekusi |
| `ltrace` | melihat panggilan fungsi library |
| `strace` | melihat system call ke kernel |
| System call | permintaan program ke kernel (open, read, write) |
| `/proc/<pid>/maps` | daftar wilayah memori sebuah proses |
| Load base | alamat basis tempat binary dimuat |
| Image base | basis yang dipakai Ghidra (default `0x00100000`) |
| ASLR | pengacakan alamat oleh OS |
| `disable-randomization` | opsi gdb yang mematikan ASLR saat debugging |
| Breakpoint | titik henti pada alamat/instruksi |
| Conditional breakpoint | breakpoint yang hanya aktif jika syarat terpenuhi |
| Watchpoint | titik henti saat nilai memori berubah |
| `rwatch` / `awatch` | watchpoint pada pembacaan / keduanya |
| `catch syscall` | berhenti saat system call tertentu dipanggil |
| `starti` | berhenti di instruksi pertama program |
| `info proc mappings` | melihat wilayah memori + basis di gdb |
| `find` | mencari pola byte/string di memori |
| `dump binary memory` | menulis isi memori ke file |
| Code path | jalur kode yang benar-benar dieksekusi |
| Triage | pemeriksaan cepat untuk menentukan arah analisis |
| Anti-debug | teknik program mendeteksi debugger (Fase 8) |

---

[← PART 4](../PART4/note.id.md) · [Peta belajar](../INDEX.id.md) · **Lab:**
[Buffer Overflow](BufferOverflow/note.id.md) · **Selanjutnya:** PART 6 — Pola Reversing
([catatan](../PART6/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>