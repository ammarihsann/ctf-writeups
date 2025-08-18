# Stolen Data

# Ringkasan

* **Artefak:** `network-log.pcapng` (lalu lintas C2 custom via TCP).
* **Temuan kunci:** Paket berisi **Base64** yang sebenarnya adalah `IV || Ciphertext || HMAC-SHA256`.
* **Kunci bersama (hex) ditemukan di stager PowerShell pada capture:**
  `9f4c8b2e6a7f1d3b9ab2c4d5e6f70812a1b2c3d4e5f60718293a4b5c6d7e8f90`
* **Skema kripto:** AES-CBC (PKCS#7) + HMAC-SHA256(iv||ct).
* **Data yang dieksfiltrasi:** command output (listing & pembacaan file) termasuk `flag.txt`.
* **Flag:** `CBD{bz_c2_with_encrypted_traffic_d34da5}`

[Log sesi C2 yang sudah didekripsi](sandbox:/mnt/data/decrypted_c2_session.txt)

---

# Langkah Penyelesaian

## 1) Triage cepat di Wireshark

1. Buka `network-log.pcapng` di **Wireshark**.
2. Cari aliran mencurigakan (C2) dengan filter:

   * `tcp.port == 4444` (port umum untuk C2 pada challenge ini), atau
   * `tcp.len > 0 && frame contains "=="` (indikasi Base64 panjang).
3. **Follow TCP Stream** pada aliran yang ramai payload teks acak/`A-Za-z0-9+/=` — terlihat potongan panjang Base64 bolak-balik antara host internal dan IP eksternal (C2).

> Insight: Tidak ada protokol aplikasi yang jelas (bukan HTTP), sehingga besar kemungkinan **raw TCP C2** dengan **payload terenkapsulasi** (stager + tugas + hasil).

## 2) Menemukan kunci kriptografi (shared secret)

Di salah satu stream awal, nampak **stager PowerShell** yang membawa konfigurasi C2. Cara cepat:

* `File → Export Objects → TCP` lalu cari objek stream text yang berisi skrip.
* Atau **strings** langsung ke berkas pcap untuk pola 64 hex chars:

  ```bash
  strings -n 8 -t x network-log.pcapng | grep -iE '[0-9a-f]{64}'
  ```

Dari sana terlihat **shared secret** 32-byte dalam bentuk hex:

```
9f4c8b2e6a7f1d3b9ab2c4d5e6f70812a1b2c3d4e5f60718293a4b5c6d7e8f90
```

Digunakan sebagai:

* `aesKey = secret[0:16]`
* `hmacKey = secret[16:32]`

## 3) Memahami format payload

Payload Base64 ketika di-decode → **blob biner**:

```
[ 16B IV ] [ ... Ciphertext ... ] [ 32B HMAC-SHA256 tag ]
```

* **Integritas:** Tag = `HMAC_SHA256(hmacKey, IV || CT)`
* **Kerahasiaan:** `CT = AES-CBC(aesKey, IV, PKCS7(plaintext))`

## 4) Ekstraksi payload dari pcap

Metode A (tshark):

```bash
tshark -r network-log.pcapng -Y "tcp.port==4444 && tcp.len>0" \
  -T fields -e data > payload_hex.txt
# lalu konversi per-baris heksadesimal → biner, parse Base64 di layer aplikasi
```

Metode B (praktis): **scan Base64** langsung dari file pcap (karena data aplikasi tersimpan berurutan), lalu uji-decrypt setiap token yang valid HMAC.

## 5) Dekripsi & rekonstruksi sesi

Gunakan skrip Python berikut untuk:

* menemukan token Base64,
* memverifikasi HMAC,
* mendekripsi AES-CBC,
* menyusun timeline berdasarkan offset kemunculan data (proxy urutan waktu).

> Dependensi: `pycryptodome` untuk AES.

```python
#!/usr/bin/env python3
import re, base64, hmac, hashlib
from binascii import unhexlify
from Crypto.Cipher import AES

PCAP = "network-log.pcapng"
shared_hex = "9f4c8b2e6a7f1d3b9ab2c4d5e6f70812a1b2c3d4e5f60718293a4b5c6d7e8f90"

secret   = unhexlify(shared_hex)
aes_key  = secret[:16]
hmac_key = secret[16:]

def try_decrypt(b64_bytes: bytes):
    try:
        blob = base64.b64decode(b64_bytes, validate=True)
    except Exception:
        return None
    if len(blob) < 16 + 32 + 1:
        return None
    iv, tag, ct = blob[:16], blob[-32:], blob[16:-32]
    calc = hmac.new(hmac_key, iv + ct, hashlib.sha256).digest()
    if calc != tag:
        return None
    plain = AES.new(aes_key, AES.MODE_CBC, iv).decrypt(ct)
    pad = plain[-1]
    if not (1 <= pad <= 16):
        return None
    return plain[:-pad].decode('utf-8', errors='replace')

data = open(PCAP, 'rb').read()
tokens = re.finditer(rb'[A-Za-z0-9+/]{40,}={0,2}', data)

timeline = []
for m in tokens:
    dec = try_decrypt(m.group(0))
    if dec:
        timeline.append((m.start(), dec))

timeline.sort(key=lambda x: x[0])
for off, msg in timeline:
    print(f"[0x{off:x}] {msg}")
```

## 6) Hasil dekripsi (cuplikan)

Timeline menunjukkan pola perintah & balasan:

```
[...]
dir
 type notes.txt
 Nothing's there?
 cd ..\Documents
 dir
  ...
  08/17/2025  13:05            40 flag.txt
 type flag.txt
 CBD{bz_c2_with_encrypted_traffic_d34da5}
```

> **Flag akhir:** `CBD{bz_c2_with_encrypted_traffic_d34da5}`

---

# Analisis & Pembahasan

* **Kenapa AES-CBC + HMAC?**
  Banyak implant sederhana memakai CBC dengan MAC terpisah (Encrypt-then-MAC). Validasi HMAC sebelum decrypt mencegah padding oracle & memastikan integritas.

* **Kenapa scan Base64 langsung di pcap?**
  Karena aplikasi mengirim Base64 “mentah” di atas TCP (tanpa framing jelas). Cara ini cepat untuk forensic: cari token panjang `A-Za-z0-9+/` dengan `'='` akhir.

* **Validasi kunci:**
  Jika `hmac != tag`, token bukan milik sesi ini (false positive). Ini menyaring noise secara otomatis.

---

# IoC (Indicators of Compromise)

* **C2 TCP port:** `4444`
* **Skema payload:** `Base64( IV || CT || HMAC-SHA256 )`
* **Kunci (dari stager):** `9f4c8b2e6a7f1d3b9ab2c4d5e6f70812a1b2c3d4e5f60718293a4b5c6d7e8f90`
* **Artefak konten:** perintah `dir`, `type`, akses `flag.txt`

---

# Rekomendasi Mitigasi

1. **TLS interception / egress filtering:** blokir outbound arbitrary TCP ke port non-standar; allow-list domain/IP yang sah.
2. **EDR telemetry:** alert pada `powershell.exe -enc` & script block logging (Event ID 4104).
3. **DLP rules:** deteksi anomali Base64 berukuran besar pada kanal non-HTTP.
4. **Network baselining:** alarm jika host melakukan koneksi persist ke IP baru/asing via port jarang.

---

Kalau ingin, aku bisa paketkan **tool kecil** (CLI) yang:

* otomatis deteksi kunci dari stager,
* ekstrak semua stream `tcp.port==4444`,
* dan keluarkan **CSV timeline** + **log plaintext** seperti yang kulampirkan.
