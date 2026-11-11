# PART 3 — Executable Formats

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 3 di YouTube](https://youtu.be/NefasNimlIk?si=iqEId\_w4sANmHAUx)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

---

## 1. Konsep Dasar Keputusan di Level Mesin (*Machine Level Decisions*)
Di tingkat kode mesin/assembly, setiap keputusan (seperti pernyataan `if-else` dalam bahasa tingkat tinggi) selalu terdiri dari **dua langkah utama**:
1. **Compare (Instruksi Perbandingan)**: Mengevaluasi dua nilai dan menyetel register penanda (*EFLAGS*, khususnya *Zero Flag / ZF*).
2. **Jump (Instruksi Lompatan)**: Membaca status flag tersebut dan menentukan apakah eksekusi akan melompat ke alamat memori lain atau berlanjut ke baris berikutnya.

---

## 2. Instruksi Perbandingan Utama

### a. `CMP` (Compare)
- **Fungsi**: Mengurangi nilai kedua dari nilai pertama (operasi pengurangan/subtraksi sederhana), tetapi **membuang hasil pengurangannya** dan hanya menyimpan hasilnya dalam penanda (*flag*).
- **Zero Flag (ZF)**:
  - Jika kedua nilai yang dibandingkan **sama**, hasil pengurangannya adalah `0` \\(\rightarrow\\) **Zero Flag aktif (`ZF = 1`)**.
  - Jika tidak sama \\(\rightarrow\\) **Zero Flag mati (`ZF = 0`)**.

### b. `TEST`
- **Fungsi**: Melakukan operasi logika bitwise `AND` antara dua nilai tanpa menyimpan hasilnya (sering digunakan dalam bentuk `TEST EAX, EAX`).
- **Tujuan**: Memeriksa apakah nilai pada register (seperti `EAX`) bernilai `0`.
- **Konteks Pemanggilan Fungsi**:
  - Fungsi perbandingan string seperti `strcmp` mengembalikan nilai `0` jika kedua string cocok (*match*).
  - Karena itu, instruksi `TEST EAX, EAX` yang ditempatkan tepat setelah pemanggilan `strcmp` bertugas mengecek: **"Apakah hasil strcmp bernilai 0 (password cocok)?"**

---

## 3. Instruksi Lompatan (*Jumps*)

| Instruksi | Nama Lengkap | Kondisi Eksekusi |
| :--- | :--- | :--- |
| **`JMP`** | Unconditional Jump | Selalu melompat tanpa syarat |
| **`JE`** | Jump if Equal | Melompat jika nilai sama (`ZF = 1`) |
| **`JNE`** | Jump if Not Equal | Melompat jika nilai tidak sama (`ZF = 0`) |
| **`JG` / `JL`** | Jump if Greater / Less | Melompat jika nilai lebih besar / lebih kecil |

> **Pola Emas**: Kombinasi **`CMP/TEST` + `JE/JNE`** adalah bentuk assembly dari struktur perabercangan `if (kondisi) { ... } else { ... }`.

---

## 4. Pemanggilan Fungsi (*Calls & Returns*)

- **`CALL`**: Menyimpan alamat instruksi berikutnya (*return address*) ke dalam *stack*, lalu melompat ke alamat fungsi yang dipanggil.
- **`RET`**: Mengambil (*pop*) alamat kembali dari *stack* dan melompat kembali ke fungsi pemanggil.

### Pembuka & Penutup Fungsi (*Bookends*)
Sebagian besar fungsi di assembly memiliki struktur pembuka (prolog) dan penutup (epilog) yang konsisten:
- **Prolog (Awal Fungsi)**:
  ```assembly
  push RBP
  mov RBP, RSP
  ```
- **Epilog (Akhir Fungsi)**:
  ```assembly
  leave
  ret
  ```

---

## 5. Analisis Alur Program "The Vault"

Alur instruksi assembly pada program pemeriksaan login:
1. **Input User**: Memanggil `printf` dan `scanf` untuk meminta input password dari pengguna.
2. **Perbandingan String**: Memanggil `strcmp(password_input, password_rahasia)`.
3. **Pengecekan Hasil**: `TEST EAX, EAX` mengecek nilai kembalian dari `strcmp`.
4. **Keputusan**:
   - Jika password salah, `strcmp` menghasilkan nilai non-zero \\(\rightarrow\\) `JNE` aktif dan melompat ke bagian **Access Denied**.
   - Jika password benar, `JNE` dilewati \\(\rightarrow\\) alur program langsung masuk ke bagian **Access Granted**.

---

## 6. Dua Cara Membobol Login (*Bypassing the Check*)

### Metode 1: Modifikasi Flag di Memori Saat Runtime (GDB)
Cara ini merubah keputusan program saat sedang berjalan di RAM tanpa mengubah file biner di disk secara permanen:

1. Buka program dengan GDB dan atur sintaks Intel:
   ```bash
   gdb ./vault
   (gdb) set disassembly-flavor intel
   ```
2. Pasang breakpoint pada baris pemeriksaan password atau setelah pemanggilan `strcmp`:
   ```bash
   (gdb) break *main+<offset_strcmp>
   (gdb) run
   ```
3. Masukkan password acak (misal: `12345`).
4. Jalankan instruksi langkah per langkah (`stepi`) hingga berada tepat di instruksi `JNE`.
5. Ubah nilai Zero Flag pada EFLAGS secara paksa agar dianggap cocok:
   ```bash
   (gdb) set $eflags |= 0x40
   (gdb) continue
   ```
6. **Hasil**: Program mengeksekusi alur **Access Granted** meskipun password salah.

---

### Metode 2: Patching Biner Permanen pada Disk (`objdump` & `dd`)
Cara ini mengubah kode biner program secara permanen sehingga password acak akan selalu diterima.

1. Cari offset dari instruksi `JNE` menggunakan `objdump`:
   ```bash
   objdump -d vault | grep jne
   ```
   *Catatan*: Opcode hex untuk `JNE` adalah `0x75`, sedangkan untuk `JE` adalah `0x74`.
2. Ganti opcode `0x75` (`JNE`) menjadi `0x74` (`JE`) pada byte offset tersebut menggunakan perintah `dd`:
   ```bash
   printf '\x74' | dd of=vault bs=1 seek=$((0xOFFSET)) conv=notrunc
   ```
3. Uji kembali biner `./vault` di terminal:
   ```bash
   ./vault
   Enter password: 12345
   # Output: Access Granted!
   ```

---

## 📊 Kesimpulan Pembelajaran
1. Memahami bagaimana instruksi `CMP` dan `TEST` mengatur penanda (*EFLAGS / Zero Flag*).
2. Memahami bagaimana `JE` dan `JNE` mengontrol arah eksekusi program.
3. Mampu melakukan bypass logika keamanan baik melalui **runtime flag manipulation (GDB)** maupun **permanent binary patching (`dd` / `objdump`)**.
4. **Langkah Berikutnya (Phase 3)**: Menggunakan alatan reverse engineering tingkat lanjut (**Ghidra**) untuk mendekompilasi seluruh program assembly kembali menjadi kode C yang mudah dibaca.

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)
