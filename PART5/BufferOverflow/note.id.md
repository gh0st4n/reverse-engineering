# PART 5.1 — Buffer Overflow

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 5 Buffer Overflow di YouTube](https://youtu.be/UVdciB72aEk?si=yjlZnevu-efP00UG)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

---

## 1. Perubahan Fase: Dari Memahami ke Mengontrol Program
* **Fase 0 hingga 4**: Berfokus pada pemahaman kode program milik orang lain (membaca instruksi *assembly*, menggunakan Ghidra, membongkar kode tersembunyi, dan mengutak-atik instruksi sederhana).
* **Fase 5 (Eksploitasi)**: Berhenti sekadar membaca dan mulai **mengambil alih kontrol program**.
* **Prinsip Utama Eksploitasi**: Jika seseorang memahami bagaimana program berpikir, di mana ia menyimpan data, dan bagaimana mengambil keputusan, ia dapat mengarahkan program untuk menjalankan perintah sesuai kehendaknya.

## 2. Analisis Target dan Kerentanan Kode
Program target memiliki **dua kerentanan utama** yang sengaja dibuat untuk pembelajaran:
1. **Fungsi Unused (`win`)**: Terdapat fungsi bernama `win()` yang mencetak pesan `"access granted"`, tetapi fungsi ini **tidak pernah dipanggil** di dalam fungsi `main()`.
2. **Penggunaan `gets()` Tanpa Batasan**: Fungsi `main()` menggunakan `gets()` untuk membaca masukan pengguna ke dalam ruang memori berukuran **64 byte** (`input`). Karena `gets()` tidak membatasi jumlah input, pengguna dapat memasukkan karakter jauh melebihi kapasitas alokasi tersebut.

## 3. Konsep Dasar Buffer Overflow dan Struktur Memori Stack
* **Analogi Buffer**: Buffer berukuran 64 byte diibaratkan seperti gelas yang menampung 64 tetes air. Jika dituangkan 200 tetes, sisa air akan meluap (*overflow*) dan menimpa area di sekitarnya.
* **Struktur Memori Stack**: Tepat di samping buffer pada memori *stack* terdapat **Return Address**—alamat memori tujuan CPU untuk melompat setelah suatu fungsi selesai dieksekusi.
* **Mekanisme Penimpaan (Overwrite)**: Input yang melebihi kapasitas buffer akan menimpa data di sebelahnya pada *stack*, termasuk *return address*. Siapa pun yang mengontrol *return address* dapat menentukan ke mana CPU melompat berikutnya.

## 4. Langkah Eksploitasi Praktis (Kali Linux & GDB)
1. **Pemicuan Crash (Segmentation Fault)**:
   * Program dikompilasi di sistem Kali Linux dan diuji dengan menginput **100 karakter `'A'`** ke dalam buffer 64 byte.
   * Program mengalami *Segmentation Fault* (crash). Crash ini terjadi karena *return address* tertimpa oleh deretan karakter `'A'` (nilai *garbage*), sehingga CPU gagal melompat ke lokasi tujuan yang sah.
2. **Pemeriksaan Register RIP di GDB**:
   * Program dijalankan di dalam debugger **GDB** dengan input 100 karakter `'A'`.
   * Perintah `info registers rip` digunakan untuk memeriksa register **RIP (Instruction Pointer)**.
   * Terlihat register RIP membaca nilai heksadesimal `0x41` (representasi karakter `'A'`), yang membuktikan bahwa input pengguna berhasil mengendalikan alur penunjuk instruksi CPU.
3. **Konstruksi Payload dan Pengalihan Alur**:
   * Alamat memori dari fungsi `win()` dicari melalui GDB.
   * Payload disusun dengan memberikan **padding** (karakter pengisi buffer hingga mencapai *return address*), lalu menimpa posisi *return address* dengan **alamat fungsi `win()`**.
   * Saat fungsi berakhir, CPU melompat langsung ke alamat fungsi `win()` dan mencetak pesan `"access granted"` tanpa memerlukan kata sandi.

## 5. Ringkasan & Gambaran Fase Berikutnya (Fase 6)
* **Signifikansi Bug**: *Buffer overflow* merupakan salah satu kelas celah keamanan paling populer dalam sejarah siber yang mendasari berbagai eksploitasi besar dan *worm*.
* **Rencana Fase 6**: Pada tahap selanjutnya, eksploitasi tidak lagi sekadar melompat ke fungsi yang sudah ada dalam program, melainkan **menginjeksikan kode sendiri (*shellcode*)** ke dalam buffer melalui teknik *overflow* yang sama untuk mendapatkan akses *shell* penuh.

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)
