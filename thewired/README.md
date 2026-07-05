# Crackme: the wired

**Sumber:** crackmes.one
**Tingkat kesulitan:** Easy/Medium
**Tools:** Ghidra, gdb (WSL)

## Deskripsi Singkat
Program `thewired` menerima satu argumen command line (password) dan
memvalidasinya terhadap sebuah string yang disembunyikan (obfuscated)
menggunakan operasi XOR di dalam binary.

## Proses Analisis

### 1. Analisis Statis dengan Ghidra

Membuka binary di Ghidra dan melihat fungsi utama lewat decompiler view.
Ditemukan struktur berikut:

```c
if (param_1 == 2) {
    // decode buffer local_f8 (36 byte) via XOR
    uVar2 = 0;
    do {
        local_f8[uVar2] = (&DAT_0012f758)[uVar2 % 0xc] ^ (&DAT_00104120)[uVar2];
        uVar2 = uVar2 + 1;
    } while (uVar2 != 0x24);

    iVar1 = strcmp(*(char **)(param_2 + 8), (char *)local_f8);

    if (iVar1 == 0) {
        // decode & print pesan sukses (173 byte)
    } else {
        // decode & print "[-] Incorrect Password" (23 byte)
    }
}
```

![Ghidra decompile fungsi utama thewired](screenshots/ghidra-decompile-main.png)

<br>

*Gambar 1: Fungsi utama mengecek `argc == 2`, lalu mendekode buffer 36-byte
menggunakan XOR antara dua tabel data (`DAT_0012f758` yang berulang tiap
12 byte, di-XOR dengan `DAT_00104120`), sebelum dibandingkan dengan
`strcmp` terhadap argumen yang diberikan user.*

Pola XOR ini adalah teknik obfuscation umum: string asli (password)
tidak disimpan mentah di binary, melainkan hasil XOR-nya saja. String asli
baru terbentuk saat program dijalankan (runtime), sehingga tidak langsung
terlihat kalau cuma di-`strings`.

Titik krusial ada di baris pemanggilan `strcmp`, terlihat di assembly:

![Assembly call strcmp](screenshots/ghidra-strcmp-call.png)

<br>

*Gambar 2: Instruksi `CALL <EXTERNAL>::strcmp` di alamat `001014b6`
menjadi titik target breakpoint untuk analisis dinamis, karena di titik
inilah kedua buffer (input user dan password asli hasil decode) sudah
siap dibandingkan.*

### 2. Analisis Dinamis dengan gdb

Alih-alih menelusuri manual logika XOR (yang butuh replikasi tabel data
satu per satu), dipilih pendekatan lebih cepat: **membaca buffer password
asli langsung dari memory saat runtime**, tepat sebelum `strcmp`
dieksekusi.

![gdb load binary](screenshots/gdb-load-binary.png)

<br>

*Gambar 3: Memuat binary `thewired` ke dalam gdb.*

![gdb starti dan proc mappings](screenshots/gdb-starti-mappings.png)

<br>

*Gambar 4: Program dijalankan dengan `starti` (stop di instruksi pertama),
lalu `info proc mappings` digunakan untuk mengonfirmasi base address
proses (`0x0000555555554000`), yang dibutuhkan untuk menghitung alamat
absolut breakpoint dari offset relatif di Ghidra (`0x14b6`).*

Alamat breakpoint dihitung dengan menjumlahkan base address dengan offset
dari Ghidra:
