# Secret of the Polyglot — Write-up (Forensics)

## Deskripsi

> The Network Operations Center (NOC) of your local institution picked up a suspicious file, they're getting conflicting information on what type of file it is. They've brought you in as an external expert to examine the file. Can you extract all the information from this strange file?
> **Hint:** This problem can be solved by just opening the file in different ways.

---

## Analisis Awal

File yang diberikan bernama `flag.pdf`. Pertama saya cek dengan perintah:

```bash
file flag.pdf
```

Output menunjukkan bahwa file ini terdeteksi sebagai **PNG image**. Jadi jelas ini adalah **polyglot file** — sebuah file yang valid untuk lebih dari satu format.

---

## Langkah Penyelesaian

### 1. Ubah ekstensi dan buka sebagai PNG

Saya ubah namanya:

```bash
mv flag.pdf flag.png
eog flag.png
```

Hasilnya, gambar tampil tapi hanya memperlihatkan **setengah flag**:

```
picoCTF{f1u3n7_
```

---

### 2. Cek struktur PNG

Saya jalankan `pngcheck` dan `binwalk`:

```bash
pngcheck -v flag.png
binwalk flag.png
```

Hasil menunjukkan:

* Ada **additional data after IEND**.
* `binwalk` mendeteksi ada **PDF v1.4** di offset `0x392 (914 desimal)` dan **zlib stream** di offset `0x47D (1149 desimal)`.

---

### 3. Ekstrak bagian PDF

Menggunakan `dd` untuk memotong mulai offset 914:

```bash
dd if=flag.png of=hidden.pdf bs=1 skip=914
```

Kemudian dibuka:

```bash
xdg-open hidden.pdf
```

Namun `binwalk -e` hanya menghasilkan file `47D` dan `47D.zlib`. Saat saya cek dengan:

```bash
cat 47D
```

Saya menemukan stream teks PDF:

```
(1n_pn9_&_pdf_7f9bccd1})Tj
```

Operator `Tj` pada PDF berarti “show text”. Jadi isi kurung itulah potongan terakhir flag.

---

## Hasil Akhir

Gabungkan kedua potongan:

* Dari PNG:

  ```
  picoCTF{f1u3n7_
  ```
* Dari PDF:

  ```
  1n_pn9_&_pdf_7f9bccd1}
  ```

### Final Flag

```
picoCTF{f1u3n7_1n_pn9_&_pdf_7f9bccd1}
```

---

## Lessons Learned

* **Polyglot file** bisa menyimpan lebih dari satu format sekaligus.
* Data ekstra sering diletakkan **setelah IEND** di PNG.
* Gunakan kombinasi `file`, `pngcheck`, `binwalk`, dan `dd` untuk memverifikasi serta mengekstrak data tersembunyi.
* Stream PDF (`(... )Tj`) bisa langsung mengungkap teks rahasia tanpa harus render keseluruhan PDF.
