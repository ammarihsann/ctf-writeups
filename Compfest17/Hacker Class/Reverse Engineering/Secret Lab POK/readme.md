## Write-up CTF COMPFEST: Secret Lab POK (Reverse Engineering)

Ini adalah pembahasan langkah demi langkah untuk menyelesaikan tantangan Reverse Engineering "Secret Lab POK". Tantangan ini menguji kemampuan analisis statis dan dinamis, serta ketelitian dalam proses *debugging*.

### Informasi Soal

  * **Nama Soal**: Secret Lab POK
  * **Kategori**: Reverse Engineering
  * **Poin**: 100
  * **Deskripsi**: Diberikan sebuah file `chall` yang berisi algoritma dekode tersembunyi.
  * **Format Flag**: `COMPFEST17{flag_sha256(flag)[:10]}`

-----

### Langkah 1: Analisis Awal File `chall`

File `chall` yang diberikan bukanlah sebuah program biner (ELF), melainkan sebuah file teks berisi *source code* Assembly. Namun, sintaksnya tidak bersih dan terlihat seperti hasil dari sebuah *disassembler* (seperti Ghidra/IDA Pro), bukan kode yang ditulis untuk di-compile langsung.

**Tujuan utama** adalah memahami algoritma dekode yang dibagi menjadi lima tahap (`one` hingga `five`) untuk menemukan string flag yang disembunyikan.

-----

### Langkah 2: Memperbaiki dan Meng-compile Kode

Karena file `chall` menggunakan sintaks yang tidak standar untuk *assembler* `nasm`, ia tidak bisa langsung di-compile. Terdapat tiga masalah utama:

1.  **Komentar**: Menggunakan `//` (gaya C++) bukan `;` (gaya NASM).
2.  **Pointer**: Menggunakan `BYTE PTR`, `WORD PTR`, dll., yang tidak dikenali `nasm`.
3.  **Label Jump**: Menggunakan kurung sudut seperti `<s1>`, yang merupakan notasi *disassembler*.

Kode tersebut harus dibersihkan terlebih dahulu menjadi file `.asm` yang valid. Setelah diperbaiki, proses kompilasi dilakukan secara manual dalam dua tahap:

1.  **Assembly dengan `nasm`**: Mengubah *source code* `.asm` menjadi *file objek* `.o`.
    ```bash
    nasm -f elf64 chall_bersih.asm -o chall.o
    ```
2.  **Linking dengan `ld`**: Mengubah *file objek* menjadi program biner yang bisa dieksekusi.
    ```bash
    ld chall.o -o chall_executable
    ```

Setelah langkah ini, kita memiliki `chall_executable` yang siap untuk dianalisis lebih lanjut.

-----

### Langkah 3: Analisis Dinamis dan Penemuan Flag

Analisis statis dengan membuat skrip Python pada awalnya menyesatkan. Ini karena data input (angka-angka `qword`) yang tertulis di **komentar** file `chall` ternyata adalah **petunjuk yang salah (*red herring*)**.

Langkah kuncinya adalah melakukan **analisis dinamis** dengan GDB pada file `chall_executable` yang sudah kita compile. Dengan menjalankan program hingga selesai dan berhenti tepat sebelum `exit`, kita bisa memeriksa isi memori dari buffer output.

Metode ini membuktikan bahwa program tersebut menghasilkan string yang berbeda dari yang diharapkan jika menggunakan data dari komentar. String yang benar dan terverifikasi dari GDB adalah:

`POK0kny4_P0K_1s_fun_w41t_t1ll_4vr_4sm`

*(Catatan: Proses debugging menemukan bahwa hasil GDB yang lebih awal (`...^1r...`) memiliki sedikit perbedaan dengan hasil akhir yang benar (`..._1s_...`), menunjukkan pentingnya verifikasi ulang yang teliti.)*

-----

### Langkah 4: Finalisasi Flag

Dengan string dekode yang sudah 100% terkonfirmasi, kita tinggal mengikuti aturan format dari soal.

1.  **String Dekode**: `POK0kny4_P0K_1s_fun_w41t_t1ll_4vr_4sm`
2.  **Proses Hash**:
      * `sha256("POK0kny4_P0K_1s_fun_w41t_t1ll_4vr_4sm")`
      * Hasilnya: `85cd411a36371f49615a51351185398a69841f4a44053982461a5c371f59698d`
3.  **Ambil 10 Karakter Pertama Hash**: `85cd411a36`
4.  **Gabungkan Semua Bagian**:
    `COMPFEST17{` + `POK0kny4_P0K_1s_fun_w41t_t1ll_4vr_4sm` + `_` + `85cd411a36` + `}`

### Flag Final

**`COMPFEST17{POK0kny4_P0K_1s_fun_w41t_t1ll_4vr_4sm_85cd411a36}`**

-----

### Pelajaran yang Diambil (Lessons Learned)

  * **Jangan Percayai Komentar**: Komentar dalam soal *reverse engineering* bisa jadi sengaja dibuat untuk menyesatkan.
  * **Verifikasi dengan Analisis Dinamis**: GDB adalah alat yang sangat kuat untuk memverifikasi apa yang *sebenarnya* dilakukan oleh sebuah program, dan bisa membantah asumsi dari analisis statis.
  * **Ketelitian adalah Kunci**: Perbedaan satu karakter saja pada string hasil dekode akan menghasilkan hash yang sama sekali berbeda, yang bisa menyebabkan frustrasi jika tidak diperiksa dengan teliti.
