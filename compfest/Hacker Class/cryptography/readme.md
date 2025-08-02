# 🛡️ CTF Write-up: Common Modulus Attack (RSA Exploitation)

**Kategori:** Cryptography
**Event:** Comfest CTF 2025
**Judul Soal:** Common Modulus
**Author:** Karev
**Tingkat Kesulitan:** Medium

---

## 📜 Deskripsi Soal

Terdapat dua ciphertext (`o1` dan `o2`) yang dienkripsi menggunakan algoritma RSA, dengan *modulus* yang sama (`n`) namun menggunakan *eksponen publik* yang berbeda (`e1` dan `e2`).

Kita diberikan:

* `n`
* `e1`, `e2`
* `o1 = pow(m, e1, n)`
* `o2 = pow(m, e2, n)`

Tugas kita adalah untuk **mengembalikan pesan asli `m` (flag)**.

---

## 🔎 Analisis

Masalah ini merupakan contoh dari **Common Modulus Attack**, yaitu eksploitasi RSA ketika:

* Dua ciphertext (`c1`, `c2`) menggunakan modulus `n` yang sama,
* Tapi eksponen publiknya berbeda (`e1 ≠ e2`), dan
* `gcd(e1, e2) = 1`

Kita bisa mengekstrak pesan asli `m` menggunakan:

```
a * e1 + b * e2 = 1    # Extended Euclidean Algorithm
```

Maka:

```
m ≡ c1^a * c2^b mod n
```

Jika `a` atau `b` negatif, gunakan modular inverse:

```
pow(mod_inverse(c1, n), -a, n)
```

---

## 🧪 Langkah Penyelesaian

### 1. Gunakan Extended Euclidean Algorithm

Untuk menemukan nilai `a` dan `b` yang memenuhi:

```math
a*e1 + b*e2 = 1
```

### 2. Tangani Eksponen Negatif

Jika `a < 0` atau `b < 0`, kita gunakan `mod_inverse(ciphertext, n)`.

### 3. Bangun kembali plaintext

```
m = (c1^a * c2^b) mod n
```

### 4. Decode hasilnya ke string (flag)

---

## 🧑‍💻 Script Solusi

```python
from Crypto.Util.number import *
from sympy import mod_inverse

# Contoh nilai dari soal (ubah ke nilai asli jika diberikan)
n = 132540234717828437127367786821953024657245901140124898587737334078412866620769512502450071102340661105481
e1 = 65537
e2 = 17
o1 = 76872061978133893223961128187352818207423612190062347635415104170331858056454405232856888723582029892239
o2 = 66616365902990160990200480374079229994723589117393841213766081871851421459819514373588940749665387516655

# Extended Euclidean Algorithm
def extended_gcd(a, b):
    if b == 0:
        return (1, 0)
    else:
        x1, y1 = extended_gcd(b, a % b)
        x, y = y1, x1 - (a // b) * y1
        return x, y

a, b = extended_gcd(e1, e2)

# Tangani nilai negatif
if a < 0:
    o1 = mod_inverse(o1, n)
    a = -a
if b < 0:
    o2 = mod_inverse(o2, n)
    b = -b

# Rekonstruksi plaintext
m = (pow(o1, a, n) * pow(o2, b, n)) % n

# Decode flag
try:
    flag = long_to_bytes(m).decode()
except:
    flag = long_to_bytes(m)

print("Flag:", flag)
```

---

## 🌟 Output Akhir

```
Flag: COMPFEST17{c0mm0n_f4ct0r_1s_n0t_s3cur3}
```

---

## 🧐 Insight & Pelajaran

* RSA **tidak aman** jika dua orang memakai *modulus (n)* yang sama.
* **GCD(e1, e2) = 1** memungkinkan kita untuk memecahkan pesan terenkripsi melalui Extended Euclidean Algorithm.
* Pastikan implementasi RSA Anda selalu menggunakan *keypair* yang benar-benar unik untuk setiap entitas.

---

## 📚 Referensi

* [Common Modulus Attack - CryptoHack](https://cryptohack.org/)
* [Extended Euclidean Algorithm](https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm)
* [RSA Attacks Summary - CryptoPals](https://cryptopals.com/)

---

## 🏷️ Tags

`#crypto` `#rsa` `#ctf` `#common_modulus` `#extended_euclidean` `#python`
