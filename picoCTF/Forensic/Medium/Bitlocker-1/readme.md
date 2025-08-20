# picoCTF 2025 – Forensics: Bitlocker-1

**Author:** Venax
**Category:** Forensics
**Description:**
Jacky is not very knowledgeable about the best security passwords and used a simple password to encrypt their BitLocker drive. See if you can break through the encryption!

File: `bitlocker-1.dd`

---

## 🕵️‍♂️ Analysis

Kita diberikan sebuah disk image (`.dd`) yang terenkripsi dengan **BitLocker**.
Tujuan kita adalah membuka drive ini dan menemukan flag di dalamnya.

Umumnya langkahnya:

1. Ekstrak hash BitLocker dari image.
2. Crack password dengan wordlist.
3. Mount drive menggunakan password hasil crack.
4. Cari flag di dalam filesystem.

---

## 🔧 Tools yang Digunakan

* **bitlocker2john** → ekstrak hash BitLocker.
* **John the Ripper / Hashcat** → crack password hash.
* **dislocker** → mount BitLocker menggunakan password.
* **rockyou.txt** → wordlist umum untuk cracking.

Installasi (Kali Linux):

```bash
sudo apt update
sudo apt install -y john hashcat dislocker sleuthkit
```

---

## 🔑 Step by Step Solution

### 1. Ekstrak hash dari image

Jalankan:

```bash
bitlocker2john -i bitlocker-1.dd > hash.txt
cat hash.txt
```

Output akan menghasilkan string hash BitLocker dengan awalan `$bitlocker$...`.

---

### 2. Crack password dengan John

Gunakan wordlist `rockyou.txt`:

```bash
john --format=bitlocker --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Hasil:

```
jacqueline        (bitlocker-1.dd)
```

Password yang ditemukan adalah:

```
jacqueline
```

> Alternatif: gunakan Hashcat dengan mode `-m 22100`.

---

### 3. Buka image dengan `dislocker`

Buat direktori mount:

```bash
mkdir dislocker
```

Jalankan dislocker dengan password yang benar:

```bash
sudo dislocker -v bitlocker-1.dd -u jacqueline dislocker
```

Jika berhasil, akan muncul file:

```
dislocker/dislocker-file
```

---

### 4. Mount hasil dekripsi

Buat folder mount untuk filesystem:

```bash
mkdir mounted
sudo mount -o loop dislocker/dislocker-file mounted
```

---

### 5. Cari flag

Cek isi folder:

```bash
ls mounted
```

Terdapat file `flag.txt`. Buka isinya:

```bash
cat mounted/flag.txt
```

---

## 🎯 Flag

```
picoCTF{us3_b3tt3r_p4ssw0rd5_pl5!_3242adb1}
```

---

## 📚 Lessons Learned

* BitLocker dapat dibuka di Linux menggunakan **dislocker** jika kita mengetahui password atau recovery key.
* Hash BitLocker bisa diekstrak dengan **bitlocker2john** lalu dicrack dengan John/Hashcat.
* Challenge ini menunjukkan betapa lemahnya penggunaan password sederhana (`jacqueline`) untuk proteksi sekuat BitLocker.
* Selalu gunakan password kompleks untuk enkripsi disk.
