Baik, terima kasih banyak atas konfirmasi finalnya. Ternyata tantangan ini jauh lebih kompleks dan cerdas dari yang kita duga. Flagnya adalah gabungan dari TIGA jalur pemecahan yang berbeda.

Ini adalah *write-up* final yang paling akurat berdasarkan solusi lengkap yang Anda temukan.

-----

# \#\# Write-up CTF: Dear bf (Compfest - Forensic)

### Informasi Tantangan

  * **Nama:** Dear bf
  * **Poin:** 211
  * **Kategori:** Forensik
  * **Lampiran:** `chall.zip`

-----

### \#\# Analisis Awal

Tantangan memberikan file `chall.zip`. Judul "Dear bf" mengarahkan pada hipotesis awal tentang bahasa pemrograman **Brainfuck**. Meskipun pada akhirnya ini bukan solusi utama, beberapa elemen tantangan (seperti nama file `secretX.txt` yang berisi potongan flag) sengaja dibuat untuk menggiring ke arah sana. Tantangan ini ternyata adalah puzzle forensik multi-jalur yang membutuhkan beberapa teknik berbeda.

-----

### \#\# Langkah 1: Membuka Arsip Utama (`chall.zip`)

Arsip `chall.zip` diproteksi password. Password ini dipecahkan menggunakan `zip2john` dan `John the Ripper`.

  * **Password Ditemukan:** `icecream`

Ekstraksi menghasilkan tiga set file yang menjadi tiga jalur berbeda untuk mendapatkan flag:

1.  `secret1.zip` - `secret63.zip`
2.  `photo64.png` - `photo71.png`
3.  `photo72.jpg` - `photo80.jpg`

-----

### \#\# Langkah 2: Menyelesaikan Tiga Jalur Puzzle

#### \#\#\# Jalur A: String Panjang dari File `secret`

Ini adalah bagian terpanjang dari flag.

1.  **Analisis:** File `secretX.zip` semuanya rusak. Analisis hexdump (`hexedit` atau `xxd`) menunjukkan **byte pertama** dari setiap file adalah `0x17`, padahal seharusnya `0x50` (karakter 'P' dari signature 'PK' ZIP).
2.  **Solusi:**
      * **Perbaikan:** Header yang rusak diperbaiki untuk semua 63 file menggunakan *loop* di terminal untuk menimpa byte pertama.
        ```bash
        for i in {1..63}; do printf '\x50' | dd of="secret$i.zip" bs=1 count=1 conv=notrunc; done
        ```
      * **Ekstraksi:** Password untuk file-file ini ditemukan di metadata file gambar (`photoXX`), yaitu **`w - 1`**. Semua file yang sudah diperbaiki diekstrak menggunakan password ini.
        ```bash
        for i in {1..63}; do unzip -P "w - 1" -d "flag_part$i" "secret$i.zip"; done
        ```
3.  **Hasil Bagian A:** Menggabungkan isi dari semua file `.txt` yang diekstrak menghasilkan string:
    **`y0u_n33d_t0_scr1pting_because_this_flag_1s_too_Long_`**

#### \#\#\# Jalur B: String dari File PNG (`photo64` - `photo71`)

1.  **Analisis:** File PNG juga rusak dan tidak bisa dibuka. Header 8-byte PNG (`\x89PNG...`) telah diganti.
2.  **Solusi:** Header diperbaiki menggunakan skrip Python untuk menulis ulang 8 byte pertama pada setiap file PNG.
    ```bash
    for i in {64..71}; do python3 -c "h=b'\x89PNG\r\n\x1a\n';d=open(f'photo{i}.png','rb').read();open(f'photo{i}_fixed.png','wb').write(h+d[8:])"; done
    ```
3.  **Hasil Bagian B:** Setelah diperbaiki, gambar-gambar tersebut dapat dilihat dan menampilkan karakter `H` dan `3` secara bergantian, menghasilkan string: **`H3H3H3H3`**

#### \#\#\# Jalur C: String dari File JPG (`photo72` - `photo80`)

1.  **Analisis:** File JPG juga rusak dengan cara yang sama.
2.  **Solusi:** Header diperbaiki dengan skrip Python untuk menulis ulang 8 byte pertama dengan header JPEG/JFIF yang valid (`\xff\xd8\xff\xe0\x00\x10\x4a\x46`).
    ```bash
    for i in {72..80}; do python3 -c "h=b'\xff\xd8\xff\xe0\x00\x10\x4a\x46';d=open(f'photo{i}.jpg','rb').read();open(f'photo{i}_fixed.jpg','wb').write(h+d[8:])"; done
    ```
3.  **Hasil Bagian C:** Gambar-gambar JPG yang telah diperbaiki menampilkan karakter `X` dan `1`, menghasilkan string: **`X1X1X1X1`**

-----

### \#\# Langkah 3: Menggabungkan Flag Final

Flag final adalah gabungan dari hasil ketiga jalur tersebut.

`COMPFEST17{` + [Hasil Jalur A] + [Hasil Jalur B] + [Hasil Jalur C] + `}`

### **FLAG: `COMPFEST17{y0u_n33d_t0_scr1pting_because_this_flag_1s_too_Long_H3H3H3H3X1X1X1X1}`**
