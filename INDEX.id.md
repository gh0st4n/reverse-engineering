# Peta Belajar — 10 Fase

[English](INDEX.md) · **Bahasa Indonesia**

Jalur dari *"belum pernah buka debugger"* sampai *"bisa membongkar program yang diproteksi"*.
Setiap fase adalah satu folder (`PART<N>/`), dan setiap folder berisi materi yang sama dalam
dua bahasa: `note.md` (Inggris) dan `note.id.md` (Indonesia).

Baru mulai? Baca [README](README.id.md) dulu, lalu masuk ke Fase 0.

## Cara membaca peta ini

- **Urutan itu penting.** Setiap fase mengasumsikan fase sebelumnya sudah selesai. Fase 0
  tidak boleh dilewati — di situ lab disiapkan, dan semua perintah setelahnya bergantung padanya.
- **Kolom "Yang kamu pelajari"** ditulis dengan bahasa sehari-hari; istilah teknis tetap
  dipakai, tapi dijelaskan saat pertama kali muncul.
- **Catatan** mengarah ke materinya. `EN segera` = draf Indonesia sudah ada, versi Inggris
  sedang ditulis. `–` = foldernya belum dibuat.

**Arti status:** ✅ Siap (dua bahasa, sudah rapi) · 🚧 Draf (catatan ada, sedang dirapikan /
diterjemahkan) · 🗓 Rencana (belum dimulai)

## Empat tahap besar

| Tahap | Fase | Setelah tahap ini kamu bisa… |
| :--- | :---: | :--- |
| **Fondasi** | 0 | Menyiapkan lab dan membongkar program pertamamu dari awal sampai akhir |
| **Mesin & bahasa** | 1–3 | Membaca assembly dan tahu di mana setiap bagian program disimpan |
| **Analisis** | 4–6 | Memetakan biner secara statis dan dinamis, serta mengenali bentuk kode aslinya |
| **RE terapan** | 7–9 | Menaklukkan crackme berproteksi dan memilih spesialisasi |

## Daftar fase

| Fase | Topik | Yang kamu pelajari | Catatan | Status |
| :---: | :--- | :--- | :--- | :---: |
| **0** | **Foundations** (Fondasi) | Pola pikir reverse engineering + kemenangan pertama lewat praktik langsung | [ID](PART0/note.id.md) · [EN](PART0/note.md) | ✅ Siap |
| **1** | **Architecture Fundamentals** (Dasar Arsitektur) | CPU, register (kotak penyimpanan super cepat di dalam CPU), memori, dan stack (tumpukan memori sementara untuk pemanggilan fungsi) | [ID](PART1/note.id.md) · [EN](PART1/note.md) | ✅ Siap |
| **2** | **Assembly Language** (Bahasa Assembly) | Membaca disassembly x86-64 (teks instruksi hasil bongkaran biner) dengan lancar | [ID](PART2/note.id.md) · [EN](PART2/note.md) | ✅ Siap |
| **3** | **Executable Formats** (Format Executable) | Anatomi file program: ELF (Linux) dan PE (Windows) | [ID](PART3/note.id.md) · [EN](PART3/note.md) | ✅ Siap |
| **4** | **Static Analysis** (Analisis Statis) | Ghidra: memetakan seluruh isi binary tanpa menjalankannya | [ID](PART4/note.id.md) · [EN](PART4/note.md) | ✅ Siap |
| **5** | **Dynamic Analysis** (Analisis Dinamis) | gdb: menjalankan program sambil mengamatinya langsung, langkah demi langkah | [ID](PART5/note.id.md) · [EN](PART5/note.md) | ✅ Siap |
| ↳ 5.1 | **Lab: Buffer Overflow** | Melihat stack rusak di debugger, lalu mengarahkan alur eksekusi (ret2win) | [ID](PART5/BufferOverflow/note.id.md) · [EN](PART5/BufferOverflow/note.md) | ✅ Siap |
| **6** | **Reversing Patterns** (Pola Reversing) | Mengenali pola kode buatan compiler (`if`, loop, `switch`) di dalam assembly | [ID](PART6/note.id.md) · [EN](PART6/note.md) | ✅ Siap |
| **7** | **CrackMes & Challenges** | Membuat keygen (pembuat kunci lisensi) dan mem-patch (menambal) target nyata | – | 🗓 Rencana |
| **8** | **Anti-reversing & Packing** | Unpack (membuka program yang dikompres/diacak) dan mengalahkan proteksi anti-debug | – | 🗓 Rencana |
| **9** | **Specialization** (Spesialisasi) | Memilih jalur: malware, firmware, mobile, atau exploit | – | 🗓 Rencana |

## Aturan folder

- Satu fase = satu folder: `PART0/` … `PART9/`.
- Setiap folder fase wajib punya **dua file**: `note.md` dan `note.id.md`, dengan judul bab
  yang sama.
- Lab yang dibahas mendalam punya subfolder sendiri dengan aturan yang sama,
  mis. `PART5/BufferOverflow/`.
- Konvensi lengkap dan tree terbaru: [README → Konvensi penamaan](README.id.md#konvensi-penamaan).

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>