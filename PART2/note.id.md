# PART 2 — Assembly Language

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 2A di YouTube](https://youtu.be/HkMetTR67XU?si=4LsW-jvBFoWNCH9Y)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

## 1. Pengantar dan Konsep Dasar
- **Tujuan Pembelajaran**: Berfokus pada kemampuan **membaca baris instruksi Assembly** yang dieksekusi langsung oleh CPU.
- **Analogi Kata Benda vs Kata Kerja**:
  - **Kata Benda (*Nouns*)**: Register, memori, dan stack (objek yang dimanipulasi oleh program).
  - **Kata Kerja (*Verbs*)**: Instruksi Assembly (tindakan yang dilakukan terhadap register dan memori).
- **Definisi Assembly**: Format yang dapat dibaca oleh manusia dari kode mesin (*raw machine code bytes*). Setiap satu baris kode mewakili satu instruksi CPU.

---

## 2. Anatomi Instruksi & Format Sintaks
### Anatomi Instruksi
Sebagian besar instruksi Assembly mengikuti struktur sederhana berikut:
\\[\text{Opcode} \quad \text{Destination}, \quad \text{Source}\\]

- **Opcode**: Perintah atau operasi yang harus dijalankan (contoh: `mov`, `add`, `lea`).
- **Destination**: Lokasi tujuan tempat menyimpan hasil operasi.
- **Source**: Nilai input yang digunakan dalam operasi.

### Perbandingan Intel Syntax vs AT&T Syntax
Terdapat dua cara penulisan sintaks instruksi Assembly:

| Fitur | Intel Syntax (Standar Pembelajaran) | AT&T Syntax (Default GDB) |
| :--- | :--- | :--- |
| **Urutan Operand** | `Destination, Source` | `Source, Destination` |
| **Penulisan Nilai/Register** | Bersih tanpa simbol khusus (`rax`, `5`) | Menggunakan simbol `$` dan `%` (`$5`, `%rax`) |
| **Keterbacaan** | Lebih natural dan mudah dibaca | Cenderung membingungkan bagi pemula |

> **Konfigurasi Utama GDB**: Untuk mengubah tampilan disassembly di GDB ke sintaks Intel, jalankan perintah:  
> `set disassembly flavor intel`.

---

## 3. Jenis-Jenis Operand
Operand pada posisi *Source* maupun *Destination* dapat berupa salah satu dari 3 jenis berikut:
1. **Immediate**: Nilai konstan atau angka literal yang ditulis langsung (contoh: `5`, `0x25`).
2. **Register**: Tempat penyimpanan data berkecepatan tinggi di dalam CPU (contoh: `rax`, `rdx`, `eax`).
3. **Memory**: Lokasi di dalam memori utama yang selalu ditandai dengan **tanda kurung siku `[...]`**.
   - 💡 **Aturan Emas**: Setiap kali melihat tanda kurung siku `[...]`, bacalah sebagai **"nilai yang tersimpan pada alamat memori ini"**.

---

## 4. Instruksi Transfer Data: `MOV` vs `LEA`

### Instruksi `MOV` (Copy Data)
- Menyalin nilai dari *Source* ke *Destination* tanpa mengubah atau menghapus isi *Source*.
- Memiliki 4 pola umum:
  1. Angka (*Immediate*) ke Register (contoh: `mov rax, 5`).
  2. Register ke Register (contoh: `mov rax, rdx`).
  3. Nilai dari Memori ke Register (contoh: `mov rax, [rbp-8]`).
  4. Nilai dari Register ke Memori (contoh: `mov [rbp-8], rax`).

### Instruksi `LEA` (Load Effective Address)
- Berbeda dengan `MOV`, `LEA` **tidak membaca isi memori**, melainkan mengambil **alamat memori itu sendiri**.
- **Perbandingan Langsung**:
  - `mov rax, [rbp-8]` \\(\rightarrow\\) Mengambil **nilai** yang tersimpan di alamat `rbp-8` lalu menyimpannya ke `rax`.
  - `lea rax, [rbp-8]` \\(\rightarrow\\) Mengambil **alamat** `rbp-8` itu sendiri lalu menyimpannya ke `rax` (dibaca: *"address of"*).

---

## 5. Operasi Aritmatika & Efek Samping Register Flags

### Operasi Aritmatika Dasar
- `ADD`: Menjumlahkan nilai *Source* ke *Destination* (contoh: `add rax, rdx` \\(\rightarrow\\) `rax = rax + rdx`).
- `SUB`: Mengurangi nilai *Destination* dengan *Source*.
- `INC` / `DEC`: Menambah 1 (*increment*) atau mengurangi 1 (*decrement*).

### Register Flags (Status CPU)
Setiap operasi `ADD` atau `SUB` secara otomatis memperbarui status pada **Flags Register**:
- **Zero Flag (ZF)**: Aktif (bernilai 1) jika hasil perhitungan bernilai `0`.
- **Sign Flag (SF)**: Aktif jika hasil perhitungan bernilai negatif.
- Register flags ini berfungsi sebagai jembatan untuk operasi logika dan percabangan (*conditional jumps*).

---

## 6. Trik Optimasi: Mengosongkan Register dengan `XOR`
- Untuk mengosongkan register menjadi `0`, *compiler* umumnya tidak menggunakan `mov eax, 0`, melainkan:  
  `xor eax, eax`.
- Operasi logika XOR terhadap nilai yang sama selalu menghasilkan `0`. Instruksi ini berukuran lebih kecil dan dieksekusi lebih cepat oleh CPU.

---

## 7. Ringkasan Pembongkaran Fungsi (Disassembly)
Saat menganalisis fungsi di GDB (seperti fungsi `main` atau `add`):
1. **Prologue Fungsi**: Baris awal yang menyiapkan *stack frame* biasanya dapat diabaikan saat pertama kali membaca alur fungsi.
2. **Pengisian Variabel**: Nilai disimpan ke slot memori lokal `[rbp-...]` menggunakan instruksi `MOV`.
3. **Eksekusi Aritmatika**: Argumen dimuat ke register, dihitung (misalnya dengan `ADD`), lalu hasil akhirnya dikembalikan menggunakan instruksi `RET`.

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)
