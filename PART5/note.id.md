# PART 5 — Dynamic Analysis

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 5 di YouTube](https://youtu.be/iuYRHUH51yA?si=p9SObV6ZbQU85wGV)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

---

## 1. Konsep Utama Malware Packing
* **Analisis Biasa vs Malware**: Pada program standar, perkakas analisis statis (seperti GDB atau Ghidra) dapat membaca kode, instruksi assembly, dan fungsi `main` secara langsung. Namun, malware di dunia nyata menggunakan teknik penyembunyian kode agar perkakas analisis gagal membacanya.
* **Definisi Packing**: *Packing* adalah teknik mengompresi dan menyembunyikan kode asli program di dalam disk.
* **Analogi Berkas ZIP**: Kode dikemas rapat di disk layaknya berkas ZIP sehingga alat analisis statis tidak dapat membaca kodenya. Saat biner dijalankan, kode tersebut mendekompresi (*unzip*) dirinya sendiri di dalam memori sebelum dieksekusi.

---

## 2. Indikator & Ciri Biner yang Di-pack
1. **Ukuran Berkas Menyusut**: Berkas biner yang terkompresi memiliki ukuran yang jauh lebih kecil daripada biner aslinya.
2. **Perkakas Analisis Statis Gagal/Bisu**: 
   * `nm` menampilkan pesan *"no symbols"*.
   * Ghidra gagal menemukan fungsi `main` dan hanya menampilkan *stub* pengurai.
   * `objdump` tidak menemukan struktur logika untuk di-disassemble.
3. **Terdeteksi Identitas Packer**: Terdapat string atau nama pengemas di dalam biner, seperti **UPX**.

---

## 3. Alur Praktis Pengemasan Berkas (Demonstrasi Kali Linux)

### Langkah Kompilasi & Pengemasan
* **Kompilasi Awal**: Program target (`vault.c`) dikompilasi menggunakan GCC menjadi biner `new_obj`.
* **Ukuran Berkas Asli**: Sebelum dikemas, ukuran biner `new_obj` adalah **17.496 byte**.
* **Eksekusi Pengemasan**: Pengemasan dilakukan dengan perintah `upx new_obj`.
* **Ukuran Berkas Setelah Di-pack**: Ukuran biner berkurang drastis menjadi **6.792 byte** (menyusut sekitar 50%).
* **Perilaku Saat Dijalankan**: Biner terkemas tetap berfungsi normal saat dieksekusi (`./new_obj`) dan dapat menerima kata sandi rahasia `SUP3R`.

---

## 4. Perbandingan Hasil Perkakas Analisis Statis

| Perkakas | Sebelum Di-pack (Biner Asli) | Sesudah Di-pack (Biner UPX) |
| :--- | :--- | :--- |
| **`nm`** | Menampilkan simbol fungsi `main` secara jelas. | Melaporkan *"no symbols"* dan tidak menemukan fungsi `main`. |
| **`objdump -d`** | Membongkar instruksi assembly fungsi `main`. | Tidak menemukan logika pembacaan kode `main`. |
| **Ghidra** | Dekompilasi menghasilkan fungsi `main` dan kode C yang dapat dibaca. | Hanya menampilkan *unpacking stub* tanpa logika pemeriksaan kata sandi. |

---

## 5. Kelemahan Packer & Metode Unpacking

### Kelemahan Mendasar
* Pengemas (*packer*) hanya bisa menyembunyikan kode di disk, tetapi **tidak bisa menyembunyikan kode secara permanen** karena biner wajib mendekompresi dirinya di dalam memori agar dapat berjalan.

### Dua Metode Unpacking
1. **Metode A (Direct/Polite Unpack)**:
   * Menggunakan opsi bawaan alat pengemas seperti perintah `upx -d new_obj`.
   * Memulihkan biner ke ukuran semula serta mengembalikan fungsi `main` pada pembacaan `nm` dan Ghidra.
2. **Metode B (In-Memory Dump)**:
   * Digunakan untuk *custom packer* atau malware nyata yang tidak memiliki fitur *unzip* langsung.
   * Analis menjalankan biner, membiarkannya mendekompresi diri di memori, lalu mengambil (*dump*) kode asli langsung dari memori.

---

## 6. Kesimpulan Utama
* **Pentingnya Unpacking**: Sebagian besar malware dunia nyata dikemas dengan *packer*. Oleh karena itu, *unpacking* adalah **langkah pertama yang paling krusial** dalam analisis malware sebelum perkakas reverse engineering seperti Ghidra dapat digunakan.

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)
