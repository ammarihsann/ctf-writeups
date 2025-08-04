# Write-up CTF Comfest: The Dumb Hacker (Forensic)

Ini adalah panduan langkah demi langkah untuk menyelesaikan soal "the dumb hacker". Soal ini adalah contoh luar biasa dari tantangan forensik yang berlapis, penuh dengan petunjuk palsu (*red herring*) dan membutuhkan analisis yang teliti terhadap berbagai artefak Windows Registry untuk merekonstruksi cerita dan menemukan flag.

### Informasi Soal

  * **Nama Soal**: the dumb hacker
  * **Kategori**: Forensic
  * **Author**: ultradiyow

### Ringkasan (TL;DR)

Tantangan ini menyediakan sebuah file registri (`chall.reg`) yang berisi jejak aktivitas seorang peretas amatir. Solusinya tidak sesederhana menemukan satu string, melainkan mengumpulkan tiga fragmen flag yang terpisah dari lokasi registri yang berbeda dan menggabungkannya. Artefak kunci yang digunakan adalah `TypedURLs`, `RecentDocs`, dan `UserAssist`. Banyak jejak lain yang valid secara forensik ternyata merupakan *red herring* yang dirancang untuk mengecoh.

-----

### Langkah-langkah Eksploitasi

#### Langkah 1: Analisis Awal - Membangun Narasi dari `TypedURLs`

Analisis dimulai dengan memeriksa riwayat URL yang diketik di browser, yang tersimpan di bawah kunci `HKEY_USERS\target\Software\Microsoft\Internet Explorer\TypedURLs`. Dari sini, kita mendapatkan gambaran tentang niat dan tingkat keahlian si peretas:

1.  `"How to open a Docs Folder?"` - Niat awal terkait dokumen.
2.  `"How do i make a Document File?"` - Mencoba membuat file baru.
3.  `"Can the computer's owner do a User Activity Tracking to check what I have Accessed?"` - Menjadi panik dan khawatir terlacak.
4.  `"How to open Paint App on a computer"` - Menyerah atau teralihkan ke aktivitas yang lebih sederhana.

Narasi "peretas bodoh" ini menjadi tema utama investigasi.

#### Langkah 2: Menemukan Fragmen Flag

Karena flag tidak ditemukan dalam bentuk teks biasa, investigasi beralih ke artefak yang lebih dalam. Kunci `RecentDocs` yang melacak aktivitas file dan folder menjadi tambang emas.

Semua penemuan di bawah ini didapat dengan menganalisis output dari perintah `strings -e l chall.reg`.

**Fragmen Pertama: `h4ck3r_l3ft_4_`**
Di bawah kunci `[...RecentDocs\Folder]`, terdapat sebuah nilai (*value*) aneh yang diberi nama `"secret"`. Jika nilai heksadesimalnya diterjemahkan ke teks ASCII, kita mendapatkan bagian pertama dari flag:

```text
p4rt_1:h4ck3r_l3ft_4_
```

**Fragmen Kedua: `N0t3_sA1d_tH4t_`**
Di bawah kunci `[...RecentDocs\.txt]`, ditemukan lagi sebuah anomali: nilai yang diberi nama `"something"`. Menerjemahkan heksadesimalnya memberikan kita bagian kedua dari flag:

```text
N0t3_sA1d_tH4t_
```

**Fragmen Ketiga: `sm00thcr1m1nal_w4s_h3re`**
Bagian terakhir ini terungkap dari dua artefak yang saling menguatkan:

1.  **Nama Folder**: Di bawah `[...RecentDocs\Folder]`, tercatat akses ke sebuah folder bernama `sm00thcr1m1nal`.
2.  **Jejak UserAssist**: Setelah melacak aktivitas eksekusi program di `[...Explorer\UserAssist\{...}\Count]`, ditemukan jejak eksekusi `Microsoft.Paint`. Di dalam data heksadesimalnya, tersembunyi sebuah string: `(name of hacker)_w4s_h3re`.

Dengan menggabungkan kedua temuan ini, `(name of hacker)` diganti dengan `sm00thcr1m1nal` untuk membentuk fragmen ketiga.

-----

### \#\# Menyusun Flag Final 🏁

Dengan menggabungkan ketiga fragmen yang ditemukan secara berurutan, kita mendapatkan flag yang lengkap:

  * Dari `RecentDocs\Folder` -\> `"secret"`: `h4ck3r_l3ft_4_`
  * Dari `RecentDocs\.txt` -\> `"something"`: `N0t3_sA1d_tH4t_`
  * Dari `RecentDocs\Folder` & `UserAssist`: `sm00thcr1m1nal_w4s_h3re`

**Flag Akhir:**

```
COMPFEST17{h4ck3r_l3ft_4_N0t3_sA1d_tH4t_sm00thcr1m1nal_w4s_h3re}
```
