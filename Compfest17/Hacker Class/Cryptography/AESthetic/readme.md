# AESthetic — Write-up (Crypto • 100 pts)

## Ringkasan

Kita diberi service dengan dua fitur kunci:

* **Menu 2 – Encrypt Flag:** mengenkripsi flag dengan **AES-CBC** memakai **IV tetap** `"heheAEStheticheh"` dan menampilkan ciphertext per blok 16-byte.   &#x20;
* **Menu 1 – Decrypt:** memberikan **orakel dekripsi AES-ECB** pada **kunci yang sama**, menerima input **hex** dan mengembalikan `AES^-1_k(·)` (hasil dekripsi blok). &#x20;

Kuncinya dibuat sekali per proses (tetap selama satu koneksi).&#x20;
Flag dipastikan kelipatan 16 byte (tidak repot padding).&#x20;

Dari kombinasi ini, ciphertext CBC dapat “diurai” blok-demi-blok memakai hasil dekripsi ECB dari menu 1.

---

## Analisis

Pada CBC:

* $C_0 = \text{AES}_k(P_0 \oplus IV)$ → $\text{AES}^{-1}_k(C_0) = P_0 \oplus IV$ → **$P_0 = \text{AES}^{-1}_k(C_0) \oplus IV$**
* $C_i = \text{AES}_k(P_i \oplus C_{i-1})$ → $\text{AES}^{-1}_k(C_i) = P_i \oplus C_{i-1}$ → **$P_i = \text{AES}^{-1}_k(C_i) \oplus C_{i-1}$**

Karena menu 1 memberi kita $\text{AES}^{-1}_k(C_i)$ untuk blok mana pun, kita bisa memulihkan **semua** blok plaintext flag selama dalam **satu koneksi** (kunci tetap).

---

## Eksploitasi (otomatis)

Script di bawah:

1. Ambil ciphertext flag (menu 2).
2. Untuk **setiap blok $C_i$**, panggil menu 1 untuk memperoleh $D_i=\text{AES}^{-1}_k(C_i)$.
3. Rekonstruksi plaintext: $P_0=D_0\oplus IV$, $P_i=D_i\oplus C_{i-1}$.
4. Cetak flag.

```python
# solve.py
from pwn import remote
import ast, re

HOST, PORT = "ctf.compfest.id", 7101
IV = b"heheAEStheticheh"  # dari chall.py

def xor_bytes(a, b): return bytes(x ^ y for x, y in zip(a, b))

def recv_prompt(p): p.recvuntil(b"> ")

def get_flag_blocks(p):
    recv_prompt(p); p.sendline(b"2")
    p.recvuntil(b"Here is the flag: ")
    buf = b""
    while b"]" not in buf:
        buf += p.recvline()
    m = re.search(r"\[.*\]", buf.decode(errors="ignore"))
    blocks = ast.literal_eval(m.group(0))
    assert all(len(b)==16 for b in blocks)
    return list(blocks)

def ecb_dec(p, block):
    recv_prompt(p); p.sendline(b"1")
    p.recvuntil(b"> "); p.sendline(block.hex().encode())
    p.recvuntil(b"Here is the result: ")
    return bytes.fromhex(p.recvline().strip().decode())

def main():
    p = remote(HOST, PORT)
    c = get_flag_blocks(p)
    d = [ecb_dec(p, b) for b in c]
    pt = [xor_bytes(d[0], IV)] + [xor_bytes(d[i], c[i-1]) for i in range(1, len(c))]
    flag = b"".join(pt).decode()
    print("Flag:", flag)
    try: recv_prompt(p); p.sendline(b"4")
    except: pass
    p.close()

if __name__ == "__main__":
    main()
```

**Output:**
`Flag: COMPFEST17{AESthetic_but_not_safe_123456789_h3h3h3h3_aaaab..._30bc09fd65}`

(Flag lengkap yang kamu submit:
`COMPFEST17{AESthetic_but_not_safe_123456789_h3h3h3h3_aaaabaaacaaadaaaeaaafaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaacoaacpaacqaacraacsaactaacuaacvaacwaacxaacyaaczaadbaadcaaddaadeaad_30bc09fd65}`)

---

## Eksploitasi (manual via `nc`)

1. Kirim `2` → salin daftar blok ciphertext (format `[b'....', b'....', ...]`).&#x20;
2. Untuk **tiap** blok:

   * Kirim `1`, lalu kirim **hex** dari blok tersebut → terima hex hasil dekripsi $D_i$.&#x20;
3. Hitung:

   * Blok-0: `P0 = D0 XOR IV` (IV = `"heheAEStheticheh"`).&#x20;
   * Blok-i: `Pi = Di XOR C[i-1]`.
4. Gabungkan semua `Pi` → flag.

---

## Akar masalah & mitigasi

* **Jangan pernah mengekspos orakel dekripsi** pada kunci yang sama dengan yang dipakai untuk mengenkripsi data sensitif (flag). Di sini, `decrypt()` memakai **AES-ECB** pada kunci yang sama dengan CBC. &#x20;
* **IV harus acak & berbeda** per enkripsi; di challenge ini IV **hard-coded & tetap**.&#x20;
* Gunakan **AEAD** (mis. AES-GCM/ChaCha20-Poly1305) dan pisahkan kunci/servis untuk enkripsi vs. dekripsi.

---

## Referensi kode (potongan penting)

* Pembuatan kunci & IV tetap: `key = os.urandom(16)`; `iv = 'heheAEStheticheh'`.&#x20;
* **Encrypt (CBC)** pada flag: `AES.new(key, AES.MODE_CBC, iv=iv.encode()).encrypt(...)`.&#x20;
* **Decrypt (ECB oracle)**: `AES.new(key, AES.MODE_ECB).decrypt(...).hex()`.&#x20;
* Menu I/O yang relevan (hex only, dump blok-blok flag).&#x20;


