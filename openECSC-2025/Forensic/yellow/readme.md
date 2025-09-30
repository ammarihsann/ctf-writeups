# Writeup — “yellow” (stego, 100 pts)

## Ringkasan

Diberikan sebuah **PNG polos berwarna kuning**. Pesan disisipkan pada **byte terakhir (LSB) dari field `Length`** di setiap chunk `IDAT`. Ingat, struktur PNG adalah deretan *chunk* dengan format **`Length(4B, big-endian) | Type(4B) | Data | CRC(4B)`**, dan `IDAT` boleh muncul berkali-kali secara berurutan—datanya dianggap sebagai *concatenation* dari semua `IDAT`. Fakta ini yang dimanfaatkan pembuat soal untuk “menulis” karakter per karakter melalui nilai `Length` tiap `IDAT`. ([libpng][1]) ([libpng][2])

Hasil:
**`openECSC{W3_4l1_l1v3_1n_4_y3ll0w_subm4r1n3}`**

---

## Solusi 1 (cepat): `strings`

Observasi awal cukup dengan:

```bash
strings -n 1 yellow.png | grep 'IDAT' | sed -E 's/^(.).*/\1/' | tr -d '\n'; echo
```

`strings` menampilkan deretan karakter yang bisa dicetak dari file biner; karena byte terakhir dari `Length` tepat berada **sebelum** teks `IDAT`, baris-baris output terlihat seperti `oIDATx`, `pIDAT\``, dst. Ambil karakter pertamanya di setiap baris yang mengandung `IDAT`, gabungkan → **flag**. ([sourceware.org][3])

---

## Solusi 2 (rapi/terprogram): parse PNG chunk

Pendekatan yang lebih “resmi” adalah membaca file PNG dan mengekstrak byte terakhir dari `Length` setiap `IDAT`.

```python
import struct

flag_bytes = []

with open("yellow.png", "rb") as f:
    # Cek signature PNG (8 byte)
    assert f.read(8) == b"\x89PNG\r\n\x1a\n"
    while True:
        len_raw = f.read(4)
        if not len_raw:
            break  # selesai
        length = struct.unpack(">I", len_raw)[0]   # big-endian 4 byte
        ctype  = f.read(4)

        # Jika ini IDAT, simpan LSB dari Length sebagai karakter
        if ctype == b"IDAT":
            flag_bytes.append(length & 0xFF)

        # skip data + CRC
        f.seek(length + 4, 1)

print(bytes(flag_bytes).decode())
```

* **`struct.unpack(">I", ...)`** digunakan untuk membaca bilangan 32-bit big-endian sesuai spesifikasi PNG. ([libpng][1])
* `IDAT` boleh berulang dan berurutan; kita manfaatkan setiap kemunculannya untuk “memanen” satu karakter dari `Length`. ([libpng][2])

Output script:
`openECSC{W3_4l1_l1v3_1n_4_y3ll0w_subm4r1n3}`

---

## Validasi & catatan

* Struktur *chunk* PNG (termasuk urutan *signature → chunks*, bentuk `Length|Type|Data|CRC`, dan endian-ness) dijelaskan jelas di spesifikasi PNG. ([libpng][1])
* Aturan `IDAT` harus berurutan dan datanya digabung menjustifikasi kenapa banyak `IDAT` sah dan tidak merusak gambar. ([W3C][4])
* Perilaku `strings` yang mengekstrak *printable sequences* menjelaskan kenapa “huruf” muncul menempel di `IDAT` pada hasil *triage*. ([sourceware.org][3])

---

## Pelajaran yang bisa diambil

1. **Stego struktur file**: selain piksel/LSB, metadata dan layout format (seperti field panjang chunk) sering jadi medium penyisipan. Spesifikasi resmi sangat membantu *threat-modeling* ide stego. ([W3C][5])
2. **Triage cepat**: `strings`, `xxd/hexdump`, dan *parsing* minimal sering cukup untuk menemukan anomali awal sebelum analisis berat.
