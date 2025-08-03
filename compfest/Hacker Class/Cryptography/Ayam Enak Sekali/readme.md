# **Write-up: Ayam Enak Sekali [Cryptography]**

Ini adalah panduan penyelesaian untuk soal kriptografi "Ayam Enak Sekali" dari COMPFEST CTF. Soal ini menantang peserta untuk mengeksploitasi implementasi AES-CBC yang keliru untuk mendapatkan flag.

## **Informasi Soal**

  * **Nama Soal:** Ayam Enak Sekali
  * **Kategori:** Cryptography
  * **Deskripsi:** Aku suka ayam. Ayam enak sekali. Apakah kamu setuju?

-----

## **Analisis Kode**

Server menjalankan skrip Python (`chall.py`) yang menawarkan beberapa opsi menu:

1.  **Bikin telur (Encrypt):** Mengenkripsi input pengguna menggunakan AES-CBC dengan kunci yang *fixed* selama sesi dan IV (Initialization Vector) yang acak setiap saat.
2.  **Pecahin telur (Decrypt):** Mendekripsi ciphertext dari pengguna. Namun, terdapat pengecekan penting: jika hasil dekripsi sama dengan `FLAG`, server menolak menampilkannya.
3.  **Dapat telur rahasia (Get Secret Egg):** Memberikan hasil enkripsi dari `FLAG` rahasia.

Poin-poin kunci dari kode:

  * **Enkripsi:** `AES.MODE_CBC`.
  * **Kunci:** Dibuat sekali saat server mulai dan tidak berubah.
  * **IV:** Dibuat acak untuk setiap enkripsi dan digabungkan di depan ciphertext.
  * **Fungsi `decrypt` server:** Fungsi ini melakukan dekripsi **dan juga melakukan `unpad`** sebelum mengembalikan hasilnya. Ini adalah detail penting yang kita temukan pada tahap akhir.

-----

## **Kerentanan: Manipulasi IV pada AES-CBC**

Kerentanan utama terletak pada sifat mode operasi CBC dan bagaimana kita dapat memanipulasi input ke fungsi dekripsi.

Rumus dekripsi untuk satu blok pada mode CBC adalah:
$$P_i = D_k(C_i) \oplus C_{i-1}$$

Di mana $P\_i$ adalah blok plaintext, $C\_i$ adalah blok ciphertext, dan $D\_k$ adalah fungsi dekripsi AES. Untuk blok pertama, $C\_{i-1}$ adalah **Initialization Vector (IV)**.
$$P_1 = D_k(C_1) \oplus IV$$

Serangan ini bekerja sebagai berikut:

1.  Kita mendapatkan enkripsi flag dari **Menu 3**. Hasilnya adalah `IV_flag || C_1 || C_2 || ...`.
2.  Kita tidak bisa langsung mendekripsi ini karena ada perlindungan di **Menu 2**.
3.  Namun, kita bisa mengirimkan ciphertext yang telah dimanipulasi. Kita ganti `IV_flag` dengan IV buatan kita sendiri, `IV_malicious` (misalnya, 16 byte nol), tetapi tetap menggunakan sisa ciphertext flag yang asli (`C_1 || C_2 || ...`).
4.  Saat server mendekripsi `IV_malicious || C_1 || C_2 || ...`, ia akan menghitung:
      * **Blok Pertama:** $P'*1 = D\_k(C\_1) \\oplus IV*{malicious}$. Hasilnya akan menjadi data sampah karena IV-nya salah.
      * **Blok Kedua:** $P'\_2 = D\_k(C\_2) \\oplus C\_1$. Hasilnya adalah **blok kedua plaintext flag yang benar**, karena $D\_k(C\_2)$ dan $C\_1$ keduanya berasal dari enkripsi asli.
      * **Blok Selanjutnya:** Hal yang sama berlaku untuk semua blok berikutnya.
5.  Karena hasil dekripsi `P'_1 || P'_2 || ...` tidak sama persis dengan flag asli (blok pertamanya berbeda), server akan lolos dari pemeriksaan `if decrypt(ct) == FLAG` dan memberikan hasilnya kepada kita.

Dari sini, kita mendapatkan semua bagian flag kecuali blok pertama. Untuk merekonstruksi blok pertama ($P\_1$), kita menggunakan rumus:
$$P_1 = P'_1 \oplus IV_{flag} \oplus IV_{malicious}$$

Semua variabel di sisi kanan kita ketahui, sehingga kita bisa mendapatkan blok pertama flag dan menyatukannya dengan sisa flag yang sudah kita dapatkan.

-----

## **Rencana Eksploitasi**

Langkah-langkah untuk mendapatkan flag adalah sebagai berikut:

1.  Hubungkan ke server.
2.  Pilih **Menu 3** untuk mendapatkan ciphertext flag (`IV_flag` + `C_flag`).
3.  Buat IV berbahaya (`IV_malicious`), misalnya 16 byte nol.
4.  Kirim `IV_malicious` + `C_flag` ke **Menu 2** untuk didekripsi.
5.  Terima hasilnya dari server. Hasil ini adalah `P'_1` (blok pertama yang rusak) + sisa flag yang benar.
6.  Pisahkan `P'_1` dan sisa flag.
7.  Hitung blok pertama yang asli ($P\_1$) dengan rumus $P\_1 = P'*1 \\oplus IV*{flag} \\oplus IV\_{malicious}$.
8.  Gabungkan $P\_1$ dengan sisa flag untuk membentuk flag yang utuh. Kita tidak perlu melakukan `unpad` karena server sudah melakukannya untuk kita.

-----

## **Skrip Final**

Berikut adalah skrip Python akhir yang digunakan untuk melakukan serangan.

```python
from pwn import *
from binascii import unhexlify, hexlify
import ast

# Konfigurasi koneksi
HOST = 'ctf.compfest.id'
PORT = 7103
BLOCK_SIZE = 16

def xor_bytes(a, b):
    """Fungsi helper untuk XOR dua byte string."""
    return bytes([x ^ y for x, y in zip(a, b)])

# Mulai koneksi
p = remote(HOST, PORT)

# 1. Dapatkan enkripsi dari FLAG (Menu 3)
log.info("Meminta encrypted flag dari server...")
p.sendlineafter(b'> ', b'3')

# Terima dan proses ciphertext dari flag.
encrypted_flag_hex = p.recvline().strip()
encrypted_flag = unhexlify(encrypted_flag_hex)
log.success(f"Menerima encrypted flag (hex): {encrypted_flag_hex.decode()}")

# 2. Pisahkan IV asli dan Ciphertext asli
iv_flag = encrypted_flag[:BLOCK_SIZE]
ct_flag = encrypted_flag[BLOCK_SIZE:]
log.info(f"IV Asli (hex): {hexlify(iv_flag).decode()}")

# 3. Buat IV berbahaya (malicious)
iv_malicious = b'\x00' * BLOCK_SIZE
log.info(f"Membuat IV berbahaya (hex): {hexlify(iv_malicious).decode()}")

# 4. Gabungkan IV berbahaya dengan ciphertext asli
malicious_ciphertext = iv_malicious + ct_flag
malicious_ciphertext_hex = hexlify(malicious_ciphertext)
log.info("Mempersiapkan payload berbahaya...")

# 5. Kirim payload berbahaya untuk didekripsi (Menu 2)
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b'> ', malicious_ciphertext_hex)
log.info("Mengirim payload berbahaya untuk dekripsi...")

# 6. Terima hasil dekripsi
line_from_server = p.recvline().strip()
decrypted_output = ast.literal_eval(line_from_server.decode())
log.success(f"Menerima hasil dekripsi: {decrypted_output}")

# 7. Rekonstruksi flag
p1_corrupted = decrypted_output[:BLOCK_SIZE]
flag_tail = decrypted_output[BLOCK_SIZE:]
log.info(f"Blok pertama rusak: {p1_corrupted}")
log.info(f"Sisa flag: {flag_tail}")

# 8. Hitung blok pertama yang asli
p1_original = xor_bytes(xor_bytes(p1_corrupted, iv_flag), iv_malicious)
log.info(f"Blok pertama asli direkonstruksi: {p1_original}")

# 9. Gabungkan semua bagian untuk mendapatkan flag final
# Tidak perlu unpad lagi karena server sudah melakukannya
flag = p1_original + flag_tail

log.success(f"FLAG LENGKAP: {flag.decode()}")

p.close()
```

-----

## **Mendapatkan Flag**

Menjalankan skrip di atas akan menghasilkan output berikut, yang mengungkapkan flag secara lengkap.

```bash
[x] Opening connection to ctf.compfest.id on port 7103
[x] Opening connection to ctf.compfest.id on port 7103: Trying 34.124.220.20
[+] Opening connection to ctf.compfest.id on port 7103: Done
[*] Meminta encrypted flag dari server...
[+] Menerima encrypted flag (hex): b96faf88d4eac73c3de37bed400e250e5e4a5b1a02f49876c6c4fdafea0ee7dfc16527717340fb2bd1a23fccc53305b7f4c0642daf926a8cef7a50ca1808dccefdb6bb3e20f81b83a20dbce35f15b0ba
[*] IV Asli (hex): b96faf88d4eac73c3de37bed400e250e
[*] Membuat IV berbahaya (hex): 00000000000000000000000000000000
[*] Mempersiapkan payload berbahaya...
[*] Mengirim payload berbahaya untuk dekripsi...
[+] Menerima hasil dekripsi: b'\xfa \xe2\xd8\x92\xaf\x94h\x0c\xd4\x00\x8c9oHQenak_sekali__telur_enak_sekali_juga_c8164a0e75}'
[*] Blok pertama rusak: b'\xfa \xe2\xd8\x92\xaf\x94h\x0c\xd4\x00\x8c9oHQ'
[*] Sisa flag: b'enak_sekali__telur_enak_sekali_juga_c8164a0e75}'
[*] Blok pertama asli direkonstruksi: b'COMPFEST17{ayam_'
[+] FLAG LENGKAP: COMPFEST17{ayam_enak_sekali__telur_enak_sekali_juga_c8164a0e75}
[*] Closed connection to ctf.compfest.id port 7103
```

**Flag:** `COMPFEST17{ayam_enak_sekali__telur_enak_sekali_juga_c8164a0e75}`
