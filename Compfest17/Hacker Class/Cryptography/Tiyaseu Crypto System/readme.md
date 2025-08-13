# Tiyaseu Crypto System — Write-up (Crypto)

## Ringkasan

Kita memecahkan RSA karena server **membocorkan sebagian bit dari faktor prima**:

* **Info 1** = 192 bit MSB dari $p$ → $A = p \gg 192$
* **Info 2** = 288 bit LSB dari $q$ → $B = q \bmod 2^{288}$
* **Info 3** = $n = p \cdot q$
  Server juga memberi ciphertext $c$ (encrypted flag). Dari kebocoran ini kita **rekonstruksi $p$** secara eksak, faktorkan $n$, lalu dekripsi.

Hasil akhir:

```
COMPFEST17{ff2a99df85c4afdba5ec037017473eb2634db266fc18cf64c273b2edf246c383}
```

---

## Analisis

Tulis $p$ sebagai:

$$
p = A \cdot 2^{192} + x,\quad \text{dengan } 0 \le x < 2^{192}.
$$

Dengan $n = pq$ dan kebocoran LSB $q$ sebesar 288 bit ($B = q \bmod 2^{288}$), ambil kongruensi modulo $2^{288}$:

$$
n \equiv p \cdot q \equiv (A\cdot 2^{192} + x)\cdot B \pmod{2^{288}}.
$$

Karena $q$ ganjil $\Rightarrow B$ ganjil, maka **invertible** modulo $2^{288}$. Kalikan kedua sisi dengan $B^{-1}$ (mod $2^{288}$):

$$
t := n \cdot B^{-1} \equiv A\cdot 2^{192} + x \pmod{2^{288}}.
$$

Perhatikan bahwa ketika dikali $2^{192}$ lalu diambil mod $2^{288}$, dari $A$ (192 bit) yang “bertahan” hanya **96 bit terendah**:

$$
A\cdot 2^{192} \equiv (A \bmod 2^{96}) \cdot 2^{192} \pmod{2^{288}}.
$$

Sebut $A_{\text{low}} = A \bmod 2^{96}$. Maka:

$$
x \equiv t - A_{\text{low}} \cdot 2^{192} \pmod{2^{288}}.
$$

Karena domain $x$ diketahui $0 \le x < 2^{192}$, nilai $x$ menjadi **unik** (bukan sekadar kongruensi). Bila hasil $t - \text{base}$ negatif, cukup tambahkan $2^{288}$ sekali.

Akhirnya:

* $p = (A \ll 192) \;|\; x$
* $q = n / p$ (cek $n \bmod p = 0$)
* Dekripsi: $d \equiv e^{-1} \pmod{\varphi(n)}$, $m \equiv c^d \bmod n$.

---

## Eksploitasi (langkah praktis)

1. **Ambil data dari server** via `nc`/script:

   * `Info 1` $\to A$
   * `Info 2` $\to B$
   * `Info 3` $\to n$
   * `Encrypted flag` $\to c$
2. **Hitung** $B^{-1} \bmod 2^{288}$ (pasti ada karena $B$ ganjil).
3. **Dapatkan** $t = n \cdot B^{-1} \bmod 2^{288}$.
4. **Ambil** $A_{\text{low}} = A \bmod 2^{96}$, lalu `base = A_low << 192`.
5. **Pulihkan** $x = (t - base) \;(\text{mod } 2^{288})$ dan pastikan $x < 2^{192}$.
6. **Bangun** $p = (A << 192) | x$, lalu $q = n // p$.
7. **Dekripsi** RSA untuk mendapatkan flag.

---

## Kode Solver (inti logika)

```python
from Crypto.Util.number import long_to_bytes

e = 65537
S1 = 192
S2 = 288
MOD = 1 << S2

def reconstruct_and_decrypt(A, B, N, C):
    invB = pow(B, -1, MOD)             # B odd -> invertible mod 2^288
    t = (N * invB) % MOD

    A_low = A & ((1 << 96) - 1)        # keep 96 LSBs
    base = (A_low << S1)

    if t < base:                       # normalize into [0, 2^288)
        t += MOD
    x = t - base                       # must land in [0, 2^192)

    p = (A << S1) | x
    assert N % p == 0
    q = N // p

    phi = (p - 1) * (q - 1)
    d = pow(e, -1, phi)
    m = pow(C, d, N)
    return long_to_bytes(m)
```

> Di solver lengkap, bagian di atas dipadukan dengan parser output `nc` dan printing hasil.

---

## Validasi & Hasil

Menjalankan solver terhadap service `ctf.compfest.id:7102` menghasilkan:

```
COMPFEST17{ff2a99df85c4afdba5ec037017473eb2634db266fc18cf64c273b2edf246c383}
```

---

## Catatan & Pembelajaran

* **Partial key leakage** pada RSA (membocorkan sebagian MSB/LSB dari faktor) dapat cukup untuk memulihkan faktor. Di sini, gabungan **MSB $p$** dan **LSB $q$** membuat kongruensi modular yang **menentukan** sisa 192 bit $p$ secara unik.
* Kunci dari solusi adalah melihat aritmetika modulo $2^{288}$ sehingga perkalian dengan $2^{192}$ “memotong” 96 bit atas dari $A$.
* Mitigasi: **jangan pernah membocorkan bit-bit faktor prima** (bahkan sebagian), gunakan ukuran kunci memadai, dan hindari debug/log sensitif.
