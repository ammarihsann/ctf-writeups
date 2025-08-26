### **Write-up: St3g0 (PicoCTF)**

Dokumen ini menjelaskan langkah-langkah yang diambil untuk menyelesaikan tantangan steganografi "St3g0" dari kompetisi PicoCTF.

#### **Informasi Tantangan**

  * **Nama Tantangan:** St3g0
  * **Kategori:** Forensics
  * **Platform:** PicoCTF
  * **Deskripsi:** `Download this image and find the flag.` (Unduh gambar ini dan temukan flagnya).

-----

#### **Analisis Awal**

Tantangan ini menyediakan sebuah file gambar tunggal bernama `pico.flag.png`. Petunjuk paling jelas datang dari nama tantangan itu sendiri: **"St3g0"**. Ini adalah variasi penulisan dari **"Stego"**, yang merupakan singkatan umum untuk **Steganografi**.

Steganografi adalah teknik menyembunyikan data rahasia di dalam file lain yang terlihat normal (dalam hal ini, sebuah file gambar). Berdasarkan format file `.png` yang bersifat *lossless* (tidak kehilangan kualitas data saat dikompresi), metode steganografi yang paling umum digunakan adalah **LSB (Least Significant Bit)**. Teknik ini bekerja dengan mengubah bit data yang paling tidak signifikan pada setiap piksel gambar untuk menyisipkan data rahasia tanpa mengubah penampilan gambar secara kasat mata.

#### **Proses Penyelesaian**

1.  **Pemilihan Alat (Tool Selection)**
    Untuk mendeteksi data yang disembunyikan menggunakan teknik LSB pada file PNG, salah satu alat yang paling cepat dan efektif adalah `zsteg`. Alat ini dirancang khusus untuk menganalisis file PNG dan BMP, serta secara otomatis memeriksa berbagai kombinasi bit dan metode encoding untuk menemukan teks atau file tersembunyi.

2.  **Eksekusi Perintah**
    Langkah selanjutnya adalah menjalankan `zsteg` pada file `pico.flag.png` melalui terminal. Perintahnya sangat sederhana:

    ```bash
    zsteg pico.flag.png
    ```

3.  **Penemuan Flag**
    Setelah perintah dieksekusi, `zsteg` akan menampilkan semua kemungkinan data tersembunyi yang ditemukannya. Dalam kasus tantangan ini, `zsteg` berhasil mendeteksi dan mengekstrak string teks yang memiliki format flag PicoCTF. Output dari perintah tersebut secara langsung mengungkapkan flag-nya.

    ```
    # Contoh output yang mungkin muncul dari zsteg
    ...
    b1,rgb,lsb,xy       .. text: "picoCTF{7h3r3_15_n0_5p00n_a9a181eb}"
    ...
    ```

#### **Flag**

Flag yang berhasil ditemukan dari dalam gambar adalah:

```
picoCTF{7h3r3_15_n0_5p00n_a9a181eb}
```

-----

#### **Kesimpulan**

Tantangan "St3g0" adalah pengenalan yang bagus untuk konsep steganografi LSB. Tantangan ini menunjukkan pentingnya mengenali petunjuk dari nama tantangan dan menggunakan alat yang tepat (`zsteg`) untuk menganalisis jenis file yang diberikan. Dengan satu perintah sederhana, flag dapat ditemukan dengan cepat dan efisien.
