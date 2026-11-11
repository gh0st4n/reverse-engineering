# PART 6 — Reversing Patterns

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 6 di YouTube](https://youtu.be/fgWMBp4TVdU?si=CQTsNsgRN7K\_QkxZ)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

---

## 1. Ikhtisar & Konsep Utama (*Code Injection*)
- **Ringkasan**: Pembahasan ini mengulas teknik **Code Injection** dalam rekayasa balik (*reverse engineering*) dan eksploitasi biner. Berbeda dari teknik sebelumnya, penyerang tidak lagi hanya melompat ke fungsi rahasia di dalam program, melainkan memasukkan *machine code* buatan sendiri ke dalam memori dan memaksa CPU untuk mengeksekusinya.
- **Perbandingan Fase Eksploitasi**:

  | Fitur | Fase 5 (*Borrowing Code*) | Fase 6 (*Code Injection*) |
  | :--- | :--- | :--- |
  | **Sumber Kode** | Menggunakan fungsi tersembunyi/tidak terpakai yang sudah ada di dalam program (misalnya fungsi *win*). | Menulis dan menyuntikkan *machine code* sendiri (*shellcode*). |
  | **Tujuan *Return Address*** | Menunjuk ke alamat fungsi tersembunyi di dalam biner program. | Menunjuk ke alamat *buffer* di memori tempat *shellcode* disimpan. |
  | **Hasil Eksploitasi** | Mengeksekusi fungsi internal program. | Mendapatkan akses *full shell* interaktif. |

---

## 2. Anatomi *Shellcode*
- **Definisi**: **Shellcode** adalah potongan kecil *raw machine code* independen yang dirancang khusus untuk menjalankan satu tugas utama: membuka *shell* (`/bin/sh`).
- **Karakteristik Utama**:
  - Berukuran sangat ringkas (pada contoh demonstrasi berukuran **26 byte**).
  - Bukan merupakan file terkompilasi terpisah, melainkan byte instruksi langsung yang dapat dieksekusi CPU.
  - **Mekanisme Kerja**: Mengosongkan beberapa *register*, memuat teks `/bin/sh`, dan melakukan pemanggilan sistem (*system call*) untuk menggantikan eksekusi program dengan *shell*.

---

## 3. Alur Eksploitasi 3 Langkah
1. **Langkah 1: Menyuntikkan *Shellcode* ke *Buffer***  
   Memanfaatkan kerentanan *buffer overflow* untuk mengalirkan *shellcode* ke dalam *input buffer* memori.
2. **Langkah 2: Menimpa *Return Address***  
   Mengubah nilai *return address* di *stack* agar menunjuk kembali ke lokasi memori *buffer* tempat *shellcode* berada.
3. **Langkah 3: Eksekusi *Shellcode* oleh CPU**  
   Saat fungsi berakhir (*return*), CPU membaca *return address* yang telah dimodifikasi dan melompat langsung ke lokasi *shellcode* untuk mengeksekusinya.

---

## 4. Mitigasi Keamanan Sistem & Konfigurasi Laboratorium
- **Dua Proteksi Keamanan Sistem Modern**:
  1. **NX Stack (No-Execute / DEP)**: Wilayah memori *stack* secara standar tidak diizinkan untuk mengeksekusi instruksi kode.
  2. **ASLR (Address Space Layout Randomization)**: Alamat memori diacak setiap kali program dijalankan sehingga lokasi *buffer* tidak konstan.
- **Konfigurasi Pengujian (Fase 6)**:
  - Agar konsep dasar *code injection* dapat dipelajari secara jelas, kedua proteksi tersebut dimatikan pada lingkungan pengujian:
    - **ASLR Dimatikan**: Memastikan alamat memori *buffer* tetap (*static*) setiap kali program dijalankan.
    - **Kompilasi Ulang dengan Executable Stack**: Mengizinkan instruksi di *stack* untuk dieksekusi oleh CPU.

---

## 5. Analisis Memori & Teknik *NOP Sled*
- **Penentuan Alamat Memori dengan GDB**:
  - Menggunakan *debugger* GDB dengan memasang *breakpoint* tepat setelah input selesai dibaca.
  - Menampilkan alamat memori *buffer* (`print buffer address`) untuk dijadikan target lompatan *return address*.
- **Konsep *NOP Sled* (Landasan Pacu / *Airport Landing Strip*)**:
  - **Tantangan**: Sangat sulit untuk mengarahkan eksekusi CPU tepat pada byte pertama *shellcode*.
  - **Solusi**: Menyisipkan deretan instruksi **NOP** (*No-Operation* / instruksi tanpa aksi) tepat sebelum *shellcode*.
  - **Mekanisme**: Jika CPU mendarat di titik mana pun pada area *NOP sled*, CPU akan "meluncur" (*slides down*) melewati instruksi NOP berturut-turut hingga mencapai awal *shellcode* dan mengeksekusinya.

---

## 6. Struktur *Payload* & Hasil Eksploitasi
- **Susunan *Payload* Lengkap**:
  1. **Padding / Filler**: Isian byte acak untuk memenuhi *buffer* hingga batas *return address*.
  2. **Return Address Baru**: Alamat memori yang menunjuk ke dalam area *NOP Sled*.
  3. **NOP Sled**: Deretan instruksi NOP sebagai area pendaratan aman bagi CPU.
  4. **Shellcode (26 Bytes)**: Kode mesin eksekusi utama.
- **Hasil Pengujian**:
  - Setelah *payload* dikirimkan, program tetap menampilkan pesan *"access denied"*, namun dilanjutkan dengan eksekusi *shell* interaktif (*kali prompt*).

---

## 7. Pratinjau Fase 7 (Menghadapi Proteksi Sistem Riil)
- Jika pertahanan NX dan ASLR diaktifkan kembali pada target riil, teknik *shellcode injection* secara langsung akan gagal.
- **Fase 7** akan mempelajari teknik penyerangan modern untuk menembus pertahanan tersebut dengan cara memanfaatkan kembali potongan kode yang sudah ada di dalam biner (*reusing existing code / ROP*).

---

## Kamus Istilah

| Istilah | Arti sederhana |
|---|---|
| **Code Injection** | Menyuntikkan kode buatan sendiri ke dalam memori program target lalu memaksanya dijalankan |
| **Shellcode** | Potongan kecil kode mesin mentah yang biasanya dirancang untuk membuka shell (`/bin/sh`) |
| **NX Stack / DEP** | Proteksi yang membuat area stack tidak bisa dieksekusi sebagai kode |
| **ASLR** | Pengacakan alamat memori oleh OS agar sulit ditebak penyerang |
| **NOP (No Operation)** | Instruksi "tidak melakukan apa-apa"; dipakai sebagai "landasan pacu" lompatan |
| **NOP Sled** | Deretan panjang instruksi NOP sehingga CPU yang mendarat di mana pun di dalamnya akan meluncur sampai ke shellcode |
| **Return Address** | Alamat yang dituju CPU setelah sebuah fungsi selesai dijalankan |
| **Payload** | Data yang disusun (padding + alamat + NOP sled + shellcode) untuk memicu eksploitasi |
| **ROP (Return-Oriented Programming)** | Teknik memanfaatkan kembali potongan kode yang sudah ada di binary untuk melewati proteksi NX/ASLR (dibahas di Fase 7) |

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>
