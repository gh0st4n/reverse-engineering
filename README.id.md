# reverse-engineering

[English](README.md) · **Bahasa Indonesia**

Perjalananku belajar **Reverse Engineering (RE / rekayasa balik)** — membongkar program
untuk memahami cara kerjanya, padahal yang kita punya cuma file biner hasil kompilasi,
bukan kode sumbernya. Catatan ditulis dari nol, setiap bagian ada praktik langsung, dan
setiap istilah teknis dijelaskan dulu dengan bahasa sehari-hari sebelum dipakai.
Cocok untuk pemula.

## Isi

- [reverse-engineering](#reverse-engineering)
  - [Isi](#isi)
  - [Untuk siapa](#untuk-siapa)
  - [Cara pakai catatan ini](#cara-pakai-catatan-ini)
  - [Peta belajar](#peta-belajar)
  - [Struktur repositori](#struktur-repositori)
  - [Kebutuhan](#kebutuhan)
  - [Konvensi penamaan](#konvensi-penamaan)
  - [Batasan \& etika](#batasan--etika)

## Untuk siapa

- Kamu sudah biasa pakai terminal (`cd`, `ls`, menjalankan perintah), tapi belum pernah
  membaca assembly atau membuka debugger.
- Kamu tahu sedikit bahasa C, atau mau belajar secukupnya sambil jalan.
- Kamu ingin paham *kenapa* sebuah program berperilaku seperti itu — bukan sekadar
  menjalankan tool.

## Cara pakai catatan ini

1. **Mulai dari PART 0.** Siapkan lab dulu (compiler, debugger, binutils) supaya semua
   perintah di bagian berikutnya benar-benar bisa jalan.
2. **Ikuti urutannya.** Setiap part mengasumsikan part sebelumnya sudah selesai.
3. **Ketik perintahnya sendiri.** Ubah satu nilai, kompilasi ulang, lihat apa yang berubah
   — di situ belajarnya.
4. **Dua bahasa, isi sama.** `note.md` (Inggris) dan `note.id.md` (Indonesia) adalah materi
   yang sama, bukan konten berbeda.

## Peta belajar

Peta lengkap: **[INDEX.id.md](INDEX.id.md)** — 10 fase (PART 0 → PART 9).
Yang sudah ditulis: PART 0 → PART 6 (draf Indonesia); versi Inggris menyusul.

| Part | Materi | Video | Catatan |
| :--- | :--- | :---: | :--- |
| **0** | **Fondasi** — pola pikir RE + kemenangan pertama lewat praktik | [tonton](https://www.youtube.com/watch?v=zDJtr9r2RFs&t=197s) | [ID](PART0/note.id.md) · [EN](PART0/note.md) |
| **1** | **Dasar Arsitektur** — CPU, register, memori, stack | [tonton](https://www.youtube.com/watch?v=eEcXSvd8PCM) | [ID](PART1/note.id.md) · [EN](PART1/note.md) |
| **2** | **Bahasa Assembly** — membaca disassembly x86-64 dengan lancar | [tonton](https://www.youtube.com/watch?v=HkMetTR67XU) | [ID](PART2/note.id.md) · [EN](PART2/note.md) |
| **3** | **Format Executable** — anatomi ELF (Linux) & PE (Windows) | [tonton](https://www.youtube.com/watch?v=NefasNimlIk) | [ID](PART3/note.id.md) · [EN](PART3/note.md) |
| **4** | **Analisis Statis** — memetakan seluruh binary tanpa menjalankannya (Ghidra) | [tonton](https://www.youtube.com/watch?v=sbGsMc9_qvg) | [ID](PART4/note.id.md) · [EN](PART4/note.md) |
| **5** | **Analisis Dinamis** — menjalankan program sambil mengamatinya (gdb) | [tonton](https://www.youtube.com/watch?v=iuYRHUH51yA) | [ID](PART5/note.id.md) · [EN](PART5/note.md) |
| ↳ 5.1 | **Lab: Buffer Overflow** — melihat stack rusak, lalu ret2win | [tonton](https://www.youtube.com/watch?v=UVdciB72aEk) | [ID](PART5/BufferOverflow/note.id.md) · [EN](PART5/BufferOverflow/note.md) |
| **6** | **Pola Reversing** — mengenali pola buatan compiler (`if`, loop, `switch`) | – | [ID](PART6/note.id.md) · [EN](PART6/note.md) |
| 7–9 | CrackMe & Tantangan · Anti-reversing & Packing · Spesialisasi — *rencana* | – | – |

## Struktur repositori

```
.
├── README.md            # versi Inggris
├── README.id.md         # berkas ini (Indonesia)
├── INDEX.md             # peta belajar, 10 fase (Inggris)
├── INDEX.id.md          # peta belajar (Indonesia)
├── PART0/               # Fondasi
│   ├── note.md
│   └── note.id.md
├── PART1/
│   └── note.id.md
├── PART2/
│   └── note.id.md
├── PART3/
│   └── note.id.md
├── PART4/
│   └── note.id.md
├── PART5/
│   ├── note.id.md
│   └── BufferOverflow/  # lab khusus di dalam PART 5
│       └── note.id.md
└── PART6/
    └── note.id.md
```

## Kebutuhan

Komputer Linux (x86-64) dengan:

| Tool | Gunanya |
| :--- | :--- |
| `gcc` | Mengompilasi target latihan berbahasa C |
| `gdb` | Debugger — menghentikan program yang berjalan untuk diperiksa isinya |
| `binutils` | `objdump`, `readelf`, `strings`, `nm` — tool untuk membedah biner |
| `ltrace` | Menampilkan panggilan fungsi library (mis. `strcmp`, `printf`) |
| `strace` | Menampilkan permintaan program ke kernel (open, read, write) |
| `xxd` | Penampil heksadesimal — memperlihatkan byte mentah sebuah file |

Mulai Fase 4 kamu juga butuh **Ghidra** + Java JDK — langkah instalasi di
[PART 4](PART4/note.id.md#3-instalasi-ghidra).

Instalasi (pilih sesuai package manager):

```bash
# Arch / pacman
sudo pacman -S base-devel gdb binutils ltrace strace xxd

# Debian / Ubuntu
sudo apt install build-essential gdb binutils ltrace strace xxd

# Fedora
sudo dnf install gcc gdb binutils ltrace strace vim-common   # xxd ada di vim-common
```

Setup lab langkah demi langkah, termasuk cara memverifikasinya, ada di
**[PART 0 → Setup lab](PART0/note.id.md#7-setup-lab)**.

## Konvensi penamaan

- `PART<N>/` — satu fase dari peta belajar.
- `note.md` — catatan Inggris. `note.id.md` — catatan Indonesia (`*.id.md` = Indonesia).
- **Setiap part wajib punya dua file itu**, dengan struktur dan judul bab yang sama.
- Lab yang dibahas mendalam punya subfolder sendiri dan aturan yang sama,
  mis. `PART5/BufferOverflow/note.md` + `note.id.md`.

## Batasan & etika

- Semua di sini untuk **belajar**: biner yang kamu kompilasi sendiri, crackme, atau target
  yang kamu punya izin tegas untuk dianalisis.
- Tidak ada catatan di sini yang menyasar software yang bukan milikmu atau tanpa izin uji.
- Beberapa perintah sengaja membuat program crash — jalankan di VM atau lingkungan
  yang siap dibuang.

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>