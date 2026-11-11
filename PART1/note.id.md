# PART 1 — Architecture Fundamentals

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 1 di YouTube](https://www.youtube.com/watch?v=eEcXSvd8PCM&list=WL&index=3)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

---
## Pendahuluan & Konsep Dasar

Sebelum mempelajari bahasa Assembly atau melakukan analisis biner lanjutan, seorang **reverse engineer** harus memahami bagaimana arsitektur komputer menjalankan sebuah program pada tingkat perangkat keras (*hardware*). 

Reverse Engineering pada tahap biner berfokus pada melacak bagaimana CPU mengeksekusi instruksi mesin, bagaimana data disimpan di register, serta bagaimana struktur memori (khususnya **stack**) mengelola alur eksekusi program dan fungsi.

---

## Arsitektur CPU & Siklus Eksekusi (Fetch-Decode-Execute)

CPU (Central Processing Unit) beroperasi melalui siklus berulang yang konstan untuk mengeksekusi setiap instruksi mesin:

1. **Fetch (Pengambilan)**: CPU mengambil instruksi berikutnya dari memori utama (RAM / *Text Segment*) berdasarkan alamat memori yang ditunjuk oleh **Instruction Pointer** (`RIP`).
2. **Decode (Penerjemahan)**: CPU menerjemahkan kode biner (*opcode*) instruksi tersebut menjadi sinyal kontrol internal.
3. **Execute (Eksekusi)**: CPU menjalankan operasi yang diminta (misalnya operasi aritmatika, perpindahan data, atau lompatan eksekusi) dan memperbarui register terkait.

```
[ RAM / Text Segment ] ---> Fetch (via RIP) ---> [ CPU Decoder ] ---> Execute ---> Update Registers/Memory
```

---

## Register CPU (Arsitektur x86_64)

Register adalah lokasi penyimpanan data berkecepatan tinggi yang terletak langsung di dalam CPU. Pada arsitektur 64-bit (x86_64), register memiliki lebar 64-bit dan dibagi menjadi beberapa kategori utama:

### General Purpose Registers (GPR)
Register umum digunakan untuk manipulasi data, perhitungan aritmatika, dan pemanggilan fungsi:

| Register (64-bit) | Sub-register (32/16/8-bit) | Peran Peruntukan Utama |
| :--- | :--- | :--- |
| **`RAX`** | `EAX`, `AX`, `AL` | **Accumulator**: Menyimpan hasil perhitungan aritmatika & nilai kembalian (*return value*) fungsi. |
| **`RBX`** | `EBX`, `BX`, `BL` | **Base Register**: Register penyimpan data umum / pointer basis. |
| **`RCX`** | `ECX`, `CX`, `CL` | **Counter Register**: Digunakan sebagai penghitung pada *loop* dan operasi string. |
| **`RDX`** | `EDX`, `DX`, `DL` | **Data Register**: Mengolah data aritmatika tambahan (perkalian/pembagian) & I/O. |
| **`RSI`** | `ESI`, `SI`, `SIL` | **Source Index**: Pointer sumber dalam operasi penyalinan memori/string. |
| **`RDI`** | `EDI`, `DI`, `DIL` | **Destination Index**: Pointer tujuan dalam operasi penyalinan memori/string. |
| **`R8` – `R15`** | `R8D`-`R15D`, dll. | Register tambahan pada arsitektur x86_64 untuk kebutuhan komputasi umum. |

### Specialized Registers (Register Khusus Penunjuk Memori)
Register ini mengontrol alur program dan struktur memori:

- **`RIP` (Instruction Pointer)**: Register paling krusial yang menunjuk ke alamat memori dari instruksi berikutnya yang akan dieksekusi oleh CPU.
- **`RSP` (Stack Pointer)**: Menunjuk ke alamat memori teratas (*top of stack*) saat ini. Nilai `RSP` berkurang saat data di-`PUSH` dan bertambah saat di-`POP`.
- **`RBP` (Base / Frame Pointer)**: Menunjuk ke dasar (*base*) dari *stack frame* fungsi yang sedang aktif, digunakan sebagai titik acuan untuk mengakses variabel lokal dan argumen fungsi.

### Status / Flags Register (`RFLAGS`)
Register status menyimpan kondisi (*state*) hasil operasi aritmatika/logika terakhir:

- **`ZF` (Zero Flag)**: Bernilai 1 jika hasil operasi bernilai nol.
- **`SF` (Sign Flag)**: Bernilai 1 jika hasil operasi menghasilkan angka negatif.
- **`CF` (Carry Flag)**: Bernilai 1 jika terjadi *carry-out* atau *borrow* pada bilangan *unsigned*.
- **`OF` (Overflow Flag)**: Bernilai 1 jika hasil aritmatika melebihi batas penampungan bilangan *signed*.

---

## Tata Letak Memori Proses (Process Memory Layout)

Saat sebuah program dieksekusi di OS Linux (seperti Kali Linux), sistem operasi mengalokasikan ruang alamat memori virtual (*Virtual Address Space*) yang terbagi menjadi beberapa segmen:

```
+-----------------------------------+ (Alamat Memori Tinggi: 0x7FFF...)
|            Kernel Space           |
+-----------------------------------+
|               Stack               | <--- Tumbuh ke Bawah (Alamat Rendah)
|                 |                 |
|                 v                 |
|                                   |
|                 ^                 |
|                 |                 |
|               Heap                | <--- Tumbuh ke Atas (Alamat Tinggi)
+-----------------------------------+
|        BSS Segment (Uninit)       |
+-----------------------------------+
|       Data Segment (Init)         |
+-----------------------------------+
|       Text Segment (Code)         | (Read & Execute)
+-----------------------------------+ (Alamat Memori Rendah: 0x0000...)
```

### Penjelasan Segmen Memori:
1. **Text Segment (.text)**: Berisi instruksi kode biner executable. Bersifat *Read-Only* dan *Executable* untuk mencegah penyuntingan kode saat run-time.
2. **Data Segment (.data)**: Menyimpan variabel global dan statis yang **sudah terinisialisasi** dengan nilai awal.
3. **BSS Segment (.bss)**: Menyimpan variabel global dan statis yang **belum terinisialisasi** (diisi nol secara otomatis).
4. **Heap**: Digunakan untuk alokasi memori dinamis (`malloc`, `calloc`, `new`). Ukuran Heap bertambah ke arah alamat memori yang lebih tinggi.
5. **Stack**: Digunakan untuk pengelolaan alur fungsi, argumen, variabel lokal, dan return address. Ukuran Stack bertambah ke arah alamat memori yang lebih rendah.

---

## Konsep Stack & Stack Frame

### Prinsip Kerja Stack
Stack adalah struktur data **LIFO (Last-In, First-Out)**. Puncak stack ditunjuk oleh register `RSP`.

- **Operasi `PUSH <value>`**:
  1. Nilai `RSP` dikurangi 8 byte (`RSP = RSP - 8`).
  2. Nilai dipindahkan ke alamat memori `[RSP]`.
- **Operasi `POP <register>`**:
  1. Data dari alamat memori `[RSP]` dipindahkan ke register target.
  2. Nilai `RSP` ditambah 8 byte (`RSP = RSP + 8`).

### Anatomi Stack Frame & Pemanggilan Fungsi
Setiap fungsi yang dipanggil membuat sebuah **Stack Frame** terisolasi:

1. **Persiapan Argumen**: Pada x86_64 System V ABI (Linux), argumen fungsi dilewatkan melalui register (`RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9`), selebihnya dimasukkan ke stack.
2. **Instruksi `CALL <function>`**:
   - Menyimpan **Return Address** (alamat instruksi tepat setelah `CALL`) ke puncak stack (`PUSH RIP_next`).
   - Melompat ke alamat fungsi target (`RIP = function_address`).
3. **Function Prologue (Persiapan Frame Baru)**:
   ```assembly
   push rbp        ; Menyimpan RBP fungsi pemanggil (Caller's RBP)
   mov rbp, rsp    ; Menjadikan RSP saat ini sebagai RBP baru untuk fungsi ini
   sub rsp, N      ; Mengalokasikan ruang N byte untuk variabel lokal
   ```
4. **Eksekusi & Pembacaan Variabel Lokal**: Variabel lokal diakses menggunakan offset relatif dari `RBP` (misalnya `[rbp - 0x8]`).
5. **Function Epilogue & `RET`**:
   ```assembly
   mov rsp, rbp    ; Mengembalikan RSP ke RBP (membersihkan variabel lokal)
   pop rbp         ; Mengembalikan RBP lama milik Caller
   ret             ; Mengambil Return Address dari stack ke RIP (POP RIP)
   ```

---

## Praktik Hands-On Debugging Menggunakan GDB di Kali Linux

Dalam video ini, eksplorasi dilakukan secara praktis menggunakan **GDB (GNU Debugger)** untuk mengamati perubahan register dan memori secara langsung.

### 6.1. Perintah Dasar GDB untuk Reverse Engineering

```bash
# Membuka file biner di GDB
gdb ./program

# Menampilkan instruksi assembly fungsi main
(gdb) disassemble main
(gdb) disas main

# Menetapkan Breakpoint pada alamat awal fungsi main
(gdb) break *main
(gdb) b *main

# Menjalankan program
(gdb) run
(gdb) r

# Menginspeksi seluruh register CPU saat ini
(gdb) info registers
(gdb) i r

# Melangkah satu instruksi assembly (Step Instruction)
(gdb) stepi   # Masuk ke dalam panggilan fungsi
(gdb) nexti   # Melompati pemanggilan fungsi (Over)
```

### Memeriksa Memori & Stack (Examine Command `x/`)

Format perintah `x/<n><f><u> <address>` digunakan untuk menginspeksi isi memori:

- `<n>` = jumlah elemen yang ingin ditampilkan.
- `<f>` = format (`x` = hex, `d` = desimal, `s` = string, `i` = instruksi assembly).
- `<u>` = ukuran unit (`b` = byte, `h` = halfword/2B, `w` = word/4B, `g` = giant/8B).

**Contoh Perintah Inspeksi:**
```bash
# Menginspeksi 20 word (hexadecimal) dari puncak stack (RSP)
(gdb) x/20wx $rsp

# Menginspeksi 10 instruksi assembly dari lokasi RIP saat ini
(gdb) x/10i $rip

# Menginspeksi string yang ditunjuk oleh register RDI
(gdb) x/s $rdi

# Menginspeksi nilai RBP dan variabel lokal
(gdb) x/gx $rbp
(gdb) x/gx $rbp-0x8
```

### Mengamati Perubahan Real-Time dalam Debugger
Saat melangkah instruksi demi instruksi (`stepi`), perhatikan:
1. Perubahan **`RIP`** yang selalu bertambah seiring eksekusi instruksi.
2. Perubahan **`RSP`** ketika terjadi eksekusi `push` atau `pop`.
3. Perubahan isi memori stack yang menyimpan alamat kembalian (*Return Address*) saat `call` dieksekusi.

---

## Kesimpulan & Ringkasan Rangkuman

- **CPU & RIP**: CPU bekerja secara bertahap dalam siklus fetch-decode-execute, di mana `RIP` menentukan lokasi instruksi berikutnya.
- **Register**: Komponen tercepat CPU; `RAX` menampung return value, `RSP` mengelola puncak stack, dan `RBP` mengunci batas stack frame.
- **Memori**: Kode hidup di *Text Segment*, variabel dinamis di *Heap*, dan variabel fungsi/aliran program di *Stack*.
- **Stack**: Tumbuh ke alamat memori yang lebih rendah. Mengerti alur `PUSH`, `POP`, `CALL`, dan `RET` adalah fondasi utama untuk memahami kerentanan seperti *Buffer Overflow* dan eksploitasi kode.

---

## Kamus Istilah

| Istilah | Arti sederhana |
|---|---|
| **CPU** | Otak komputer; menjalankan instruksi mesin satu per satu |
| **Fetch-Decode-Execute** | Siklus kerja CPU: ambil instruksi → terjemahkan → jalankan |
| **Register** | Tempat penyimpanan super cepat di dalam CPU (mis. `RAX`, `RSP`) |
| **GPR (General Purpose Register)** | Register serbaguna untuk perhitungan dan manipulasi data |
| **RIP (Instruction Pointer)** | Register yang menunjuk instruksi berikutnya yang akan dijalankan |
| **RSP (Stack Pointer)** | Register yang menunjuk puncak *stack* saat ini |
| **RBP (Base/Frame Pointer)** | Register penanda dasar dari *stack frame* fungsi yang aktif |
| **RFLAGS** | Register status yang menyimpan hasil operasi terakhir (ZF, SF, CF, OF) |
| **Virtual Address Space** | Ruang alamat memori "virtual" yang dilihat tiap program, dikelola OS |
| **Text Segment** | Bagian memori berisi kode program (read-only, executable) |
| **Data / BSS Segment** | Bagian memori untuk variabel global: `.data` (sudah ada nilai awal), `.bss` (belum diisi) |
| **Heap** | Area memori untuk alokasi dinamis (`malloc`), tumbuh ke alamat tinggi |
| **Stack** | Area memori LIFO untuk variabel lokal & alur fungsi, tumbuh ke alamat rendah |
| **LIFO (Last-In, First-Out)** | Data terakhir masuk adalah yang pertama keluar |
| **PUSH / POP** | Menaruh / mengambil data dari puncak stack |
| **Stack Frame** | "Ruang kerja" milik satu pemanggilan fungsi di dalam stack |
| **CALL / RET** | `CALL` memanggil fungsi (menyimpan alamat kembali); `RET` kembali ke pemanggil |
| **Return Address** | Alamat instruksi yang akan dijalankan setelah fungsi selesai |
| **Prologue / Epilogue** | Instruksi pembuka/penutup standar di awal/akhir fungsi |
| **ABI (Application Binary Interface)** | Aturan baku bagaimana fungsi menerima argumen & nilai kembalian (mis. System V ABI di Linux) |
| **GDB (GNU Debugger)** | Alat untuk menjalankan & memeriksa program secara langsung (dinamis) |
| **Breakpoint** | Titik henti yang diatur di debugger agar program berhenti di sana |

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>
