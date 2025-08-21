# picoCTF – *endianness-v2* (Forensics)

**Author:** Junias Bonou
**Category:** Forensics
**Difficulty:** ★★☆
**Flag:** `picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_f72c4bf7}`

## TL;DR

File tipenya “terlihat rusak” karena **urutan byte diacak per 4 byte (DWORD)**. Dengan membalik **setiap blok 4 byte** di seluruh file, struktur kembali normal sebagai **JPEG** berisi teks flag.

---

## Tools

* `file`, `xxd`
* Python (one-liner)
* Opsional: hex editor (HxD/010 Editor/wxHexEditor) untuk perbaikan manual

---

## Recon & Analisis

1. **Cek signature / magic bytes**

   ```bash
   file challengefile
   xxd -g 1 -l 32 challengefile
   ```

   JPEG normal diawali `FF D8 FF E0/E1/E8` dan diakhiri `FF D9`.
   Pada file ini, byte tampak “mendekati” pola JPEG namun posisinya salah (mis. `e0 ff d8 ff ...`).

2. **Tes tampilan per 4 byte (DWORD)**
   Trik cepat untuk mendeteksi “kebalik per-4 byte”: tampilkan dump **little-endian** per 4 byte.

   ```bash
   xxd -e -g 4 -l 32 challengefile
   ```

   Jika muncul `ffd8ffe0` dan “JFIF” (`4a464946`) di sisi ASCII, itu tanda kuat bahwa:

   * Tipe file = **JPEG**
   * Data = **dibalik per 4 byte** di seluruh file

3. **Validasi hipotesis**
   Cek bagian akhir:

   ```bash
   tail -c 16 challengefile | xxd -e -g 4
   ```

   Jika terlihat `ffd9` pada tampilan per-4 byte, makin menguatkan bahwa cukup **reverse tiap 4 byte**.

---

## Perbaikan (Automatis)

**Python one-liner** untuk membalik setiap blok 4 byte:

```bash
python3 - << 'PY'
b = open("challengefile","rb").read()
out = bytearray()
for i in range(0, len(b), 4):
    chunk = b[i:i+4]
    out += chunk[::-1] if len(chunk)==4 else chunk
open("fixed.jpg","wb").write(out)
PY
```

Cek hasil:

```bash
file fixed.jpg
# -> harus terdeteksi sebagai JPEG
```

Buka `fixed.jpg` → gambar berisi teks flag.

---

## Flag

```
picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_f72c4bf7}
```

---

## Catatan

* `conv=swab` pada `dd` hanya menukar **per-2 byte (16-bit)**, **tidak cukup** untuk kasus ini.
* Jika suatu file “mirip” signature format tertentu namun tidak pas, **coba tampilkan dengan `xxd -e -g 4`**; sering ketahuan kalau masalahnya adalah endianness per-4 byte.
