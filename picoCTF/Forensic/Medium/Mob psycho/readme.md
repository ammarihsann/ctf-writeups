# Write-up: picoCTF – Forensics – **Mob psycho**

## Deskripsi

> **Challenge**: “Mob psycho”
> **Kategori**: Forensics
> **Hint**:
>
> 1. APK bisa di-*unzip*
> 2. Setelah diekstrak, gunakan *shell tools* untuk mencari

Targetnya sederhana: temukan flag yang disembunyikan di dalam sebuah file APK Android.

---

## Lingkungan & Tools

* OS: Linux (Kali/Ubuntu/WSL juga oke)
* Tools: `unzip`, `find`, `xxd` (ada di paket `vim-common`/`xxd`)

> Semua perintah di bawah dijalankan dari direktori yang sama dengan file `mobpsycho.apk`.

---

## Langkah Singkat (TL;DR)

```bash
# 1) Ekstrak APK (APK = file ZIP)
unzip mobpsycho.apk -d mobpsycho
cd mobpsycho

# 2) Cari file bernama "flag*"
find . -type f -iname 'flag*'
# Output: ./res/color/flag.txt

# 3) Lihat isinya (ternyata heksadesimal)
cat ./res/color/flag.txt
# 7069636f4354467b6178386d433052553676655f4e5838356c346178386d436c5f61336562356163327d

# 4) Decode hex → teks
xxd -r -p ./res/color/flag.txt
# picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

**Flag:**

```
picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

---

## Penjelasan Detail

### 1) Ekstrak APK

APK pada dasarnya adalah **arsip ZIP**. Maka, kita bisa langsung mengekstraknya:

```bash
unzip mobpsycho.apk -d mobpsycho
cd mobpsycho
```

Struktur standar APK akan muncul: `AndroidManifest.xml`, `classes.dex`, folder `res/`, dll.

### 2) Gunakan `find` untuk berburu nama file mencurigakan

Karena *hint* menyarankan “shell tools”, pendekatan termudah adalah mencari **file yang namanya mengandung kata `flag`**:

```bash
find . -type f -iname 'flag*'
```

Dari sini ditemukan:

```
./res/color/flag.txt
```

Folder `res/color/` biasanya untuk definisi warna (XML), jadi `flag.txt` di sana cukup mencurigakan — kemungkinan sengaja “disamarkan”.

### 3) Buka file & identifikasi encoding

Lihat konten file:

```bash
cat ./res/color/flag.txt
```

Isinya berupa **heksadesimal**:

```
7069636f4354467b6178386d433052553676655f4e5838356c346178386d436c5f61336562356163327d
```

### 4) Decode hex → teks asli

Gunakan `xxd` untuk mengubah hex menjadi tekstual:

```bash
xxd -r -p ./res/color/flag.txt
```

Hasil:

```
picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

Selesai ✅

---

## Catatan & Tips

* `find -name 'flag*'` bersifat **case-sensitive**.
  Untuk abaikan kapital: `find -iname 'flag*'`.
* Jika flag tidak ditemukan dari nama file, lanjutkan dengan:

  * `grep/rg` pada isi file (`res/values/strings.xml`, `.smali`, `.java` hasil dekompilasi).
  * `strings` pada `classes.dex`/`resources.arsc`.
  * `apktool`/`jadx` jika flag dirangkai di **runtime**.
* Menyimpan flag sebagai **hex** di lokasi yang “tak lazim” (`res/color/`) adalah trik klasik pembuat soal.

---

## One-liner (repro super cepat)

```bash
unzip -p mobpsycho.apk res/color/flag.txt | xxd -r -p
```

**Output:**

```
picoCTF{ax8mC0RU6ve_NX85l4ax8mCl_a3eb5ac2}
```

---

## Kesimpulan

Mengikuti *hint* untuk men-*unzip* APK dan memanfaatkan *shell tools* sederhana (`find` + `xxd`), flag dapat ditemukan dengan cepat di `res/color/flag.txt` yang berisi representasi **hex** dari string flag.
