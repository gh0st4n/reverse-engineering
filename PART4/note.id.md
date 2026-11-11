# PART 4 — Static Analysis

[English](note.md) · **Bahasa Indonesia**

> [!NOTE]
> Sumber: [PART 4 di YouTube](https://youtu.be/sbGsMc9_qvg?si=BNq7Y_BUPWNHQHOf)
>
> Catatan ini ditulis untuk pemula. Istilah teknis tetap ditulis dalam bahasa aslinya (agar kamu terbiasa dengan istilah yang dipakai di dunia nyata), tetapi selalu dijelaskan dengan bahasa sederhana. Ada **Kamus Istilah** di bagian paling bawah.

---

## 1. Evolusi Metodologi Reverse Engineering

Dalam seri *Reverse Engineering from Zero*, tahapan analisis logika biner berkembang secara bertahap:

* **Phase 0**: Membongkar logika program sederhana tanpa *assembly*, seperti ekstraksi string rahasia menggunakan utilitas `strings`.
* **Phase 1**: Memahami arsitektur internal mesin dan mekanisme kerja eksekusi tingkat rendah.
* **Phase 2**: Membaca dan mematahkan alur logika program secara manual baris demi baris menggunakan debugger GDB.

### Masalah Skalabilitas Analisis Manual
Pendekatan analisis manual baris demi baris pada Phase 2 sangat membantu pemahaman dasar, tetapi tidak realistis untuk perangkat lunak dunia nyata. Program komersial terdiri dari ribuan fungsi sehingga analisis manual baris demi baris akan memakan waktu berhari-hari. Oleh karena itu, dibutuhkan perkakas yang dapat merekonstruksi struktur program dan mengubahnya menjadi kode yang mudah dibaca secara otomatis.

---

## 2. Pengenalan Perkakas Ghidra

**Ghidra** adalah perkakas analisis kode (*reverse engineering framework*) yang bersifat gratis dan *open-source*. Dua keunggulan utamanya meliputi:
1. **Pengorganisasian Kode Assembly**: Memetakan seluruh instruksi *assembly* dari biner secara terstruktur.
2. **Mesin Dekompilasi (*Decompiler Engine*)**: Menerjemahkan instruksi *assembly* yang rumit kembali menjadi *pseudo-code* menyerupai bahasa C secara instan.

---

## 3. Instalasi, Konfigurasi Project, dan Impor Biner

### Langkah Instalasi di Kali Linux
Instalasi Ghidra pada sistem Kali Linux dilakukan melalui terminal menggunakan perintah:

```bash
sudo apt update && sudo apt install ghidra
```

Untuk menjalankan Ghidra, ketik perintah:

```bash
ghidra
```

### Pembuatan Project dan Impor File
1. Buka menu **File** > **New Project**.
2. Pilih **Non-Shared Project**, lalu klik **Next**.
3. Tentukan direktori penyimpanan (*Project Directory*) dan isi **Project Name** (misalnya `vault1`), lalu klik **Finish**.
4. Buka menu **File** > **Import File**.
5. Pilih biner target `vault`, lalu klik **OK**.

### Metadata Biner (*Import Result Summary*)
Setelah file diimpor, Ghidra menampilkan ringkasan atribut teknis biner:

| Atribut Biner | Spesifikasi |
| :--- | :--- |
| **Program Name** | `vault` |
| **Language ID** | `x86` |
| **Compiler** | `GCC` |
| **Processor** | `x86` |
| **Endianness** | `Little` |
| **Address Size** | `64-bit` |

---

## 4. Struktur Antarmuka Code Browser dan Otomatisasi Analisis

Untuk memulai analisis biner:
1. Klik kanan pada file `vault` di jendela project `vault1`, lalu pilih **Open with Code Browser**.
2. Saat muncul jendela konfirmasi analisis (*"Vault has not been analyzed. Would you like to analyze it now?"*), pilih **Yes**.
3. Gunakan konfigurasi bawaan (*default*), lalu klik **Analyze**. Ghidra akan menganalisis fungsi, string, dan relasi kode secara otomatis.

### Tiga Panel Utama Code Browser
Antarmuka Code Browser terbagi menjadi tiga panel utama:

* **Symbol Tree (Panel Kiri)**: Menampilkan hierarki simbol, daftar fungsi (seperti `main` dan `frame_dummy`), serta *Data Type Manager*.
* **Listing Vault (Panel Tengah)**: Menampilkan instruksi *assembly* lengkap beserta alamat memori dan registrator.
* **Decompiler (Panel Kanan)**: Menampilkan hasil dekomplikasi instruksi *assembly* menjadi *pseudo-code* bahasa C.

---

## 5. Analisis Kasus & Refactoring Biner "The Vault"

### Dekompilasi Fungsi `main`
Saat fungsi `main` dipilih pada panel *Symbol Tree*, panel *Decompiler* menampilkan rekonstruksi logika program:

```c
int main(void)
{
    int iVar1;
    char input;

    iVar1 = strcmp(input, "super secret");
    if (iVar1 == 0) {
        puts("access granted");
    }
    else {
        puts("access denied");
    }
    return 0;
}
```

Alur logika biner `vault`:
1. Program menerima masukan string dari pengguna.
2. Membandingkan masukan menggunakan fungsi `strcmp` terhadap string password rahasia `"super secret"`.
3. Jika masukan cocok (`iVar1 == 0`), program mencetak `access granted`.
4. Jika tidak cocok, program mencetak `access denied`.

### Refactoring Nama Variabel (*Renaming*)
Dekompilator Ghidra menggunakan nama variabel generik seperti `iVar1` atau `local_48`. Untuk meningkatkan keterbacaan:
1. Klik kanan pada variabel `iVar1` di panel *Decompiler*.
2. Pilih **Rename Variable**.
3. Ubah nama menjadi `match`, lalu klik **OK**.
4. Baris kode akan otomatis terbarui menjadi: `match = strcmp(input, "super secret");`.
5. Catatan/komentar (*comments*) juga dapat ditambahkan pada baris kode tertentu untuk kebutuhan dokumentasi.

---

## 6. Rangkuman Pembelajaran dan Tantangan Analisis Malware (Phase 4)

* **Otomatisasi Dekompilasi**: Ghidra memangkas waktu analisis biner secara signifikan dari hitungan hari menjadi hitungan detik.
* **Navigasi Kode**: Fokus analisis berpusat pada penelusuran fungsi (*Symbol Tree*), pembacaan instruksi (*Listing*), dan pembacaan *pseudo-code* (*Decompiler*).
* **Refactoring**: Nama variabel dan struktur kode dapat diperbarui secara interaktif untuk mempermudah pemahaman.

### Persiapan Analisis Malware pada Phase 4
Program "The Vault" merupakan biner sederhana tanpa mekanisme perlindungan. Pada **Phase 4**, analisis akan berlanjut ke sampel malware dunia nyata yang dilengkapi teknik perlindungan seperti:
* **Obfuscation**: Pengaburan struktur kode agar sulit dibaca dekompilator.
* **Packing**: Kompresi dan enkripsi kode muatan biner.
* **Anti-Analysis Tricks**: Teknik pendeteksian lingkungan debugger dan pencegahan analisis.

---

## Kamus Istilah

| Istilah | Arti sederhana |
|---|---|
| **Ghidra** | Framework reverse engineering gratis buatan NSA: disassembler + decompiler |
| **Disassembler** | Alat yang menerjemahkan kode mesin menjadi assembly |
| **Decompiler** | Alat yang menebak kode tingkat tinggi (mirip C) dari assembly |
| **Pseudo-code** | Kode hasil dekompilasi yang menyerupai C tapi bukan C asli |
| **Code Browser** | Jendela kerja utama Ghidra untuk menganalisis satu binary |
| **Symbol Tree** | Panel Ghidra berisi daftar fungsi dan simbol dalam binary |
| **Listing** | Panel Ghidra yang menampilkan instruksi assembly beserta alamatnya |
| **Refactoring (rename)** | Mengganti nama variabel generik (`iVar1`) menjadi nama yang jelas maknanya |
| **Obfuscation** | Teknik mengaburkan struktur kode agar sulit dibaca/dekompilasi |
| **Packing** | Mengompresi/menyembunyikan kode asli di dalam file agar tidak terbaca statis |
| **Anti-Analysis** | Teknik program mendeteksi debugger/sandbox lalu menghindari analisis |

---

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>
