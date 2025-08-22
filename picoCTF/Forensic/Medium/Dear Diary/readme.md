# picoCTF – Dear Diary (Forensics)

**Author:** syreal
**Hint:** “Jika mengamati data biner mentah di terminal, kamu bisa tertipu soal isi sebuah blok.”

## Ringkasan

Diberikan sebuah **disk image** Linux. Flag tidak tersimpan sebagai satu file jelas; potongannya tersebar pada file-file kecil (bahkan berukuran 0 byte) di direktori `root/secret-secrets/`. Kita gunakan **Autopsy** untuk mengekstrak & menganalisis semua partisi, lalu mencari file teks dan menyatukan potongan menjadi flag.

**Flag:** `picoCTF{1_533_n4m35_80d24b30}`

---

## Tools

* Autopsy (2.x atau 4.x)
* (Opsional) The Sleuth Kit CLI: `mmls`, `fls`, `icat`, `istat`, `blkls`

---

## Langkah (Autopsy)

1. **Buka image sebagai *Disk Image***

   * `Add Data Source → Disk Image or VM File → pilih disk.flag.img`
   * Biarkan Autopsy mendeteksi partisi (jangan hanya memilih partisi manual).
   * *Tips:* di 4.x, jalankan ingest minimal: **File Type Identification**, **Embedded File Extractor**, **Keyword Search**.

2. **Analyze seluruh image**

   * Pada panel kiri, pilih data source (volume ext).
   * **Keyword Search**: cari `.txt` untuk memfilter file teks kecil.
   * Tunggu semua sampe ke load.
   

3. **Satukan potongan**

   * Susun fragmen secara berurutan hingga membentuk flag lengkap:

     ```
     picoCTF{1_533_n4m35_80d24b30}
     ```

---

## Kenapa hint-nya penting?

Beberapa file berukuran **0 byte** tapi masih memiliki artefak di **block/fragment/slack space**. Melihat “raw” cepat di terminal sering tampak kosong, padahal string masih ada di blok terkait. Viewer **Strings/Hex** (atau traversing *Data Unit*) mengungkap isi tersebut.

---

## Catatan & Pitfall

* **Jangan hanya menambahkan partisi tunggal** saat import; tambahkan **disk image** penuh agar Autopsy mengekstrak semua partisi & artefak.
* Selalu cek **Strings/Hex** bila **Text** kosong.
* Jika pakai Autopsy 2.x, gunakan tombol **FORCE** di **Meta Data** untuk menampilkan fragmen.
