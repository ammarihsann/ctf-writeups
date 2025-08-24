# Write-up — **MSB** (picoCTF, Forensics/Stego)

**Author:** LT “syreal” Jones
**File:** `Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png`
**Hint:** “What’s causing the ‘corruption’ of the image?”

---

## Ringkasan

Gambar “terlihat korup” (warna berisik) namun **lolos analisis LSB**. Artefak visual sekuat itu biasanya muncul ketika **bit besar** (MSB) diubah. Ternyata, penyerang menyisipkan teks panjang ke **Most Significant Bits (bit-7)** pada kanal **R, G, B**. Di tengah aliran teks itu terdapat **flag**.

**Flag:**

```
picoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_572ad5fe}
```

---

## Langkah Penyelesaian

### 1) Intuisi awal

* Nama challenge: **“MSB”** → kuat mengarah ke **Most Significant Bit**.
* Deskripsi: “passes LSB statistical analysis” + **artefak visual parah** → modifikasi terjadi pada **bit 6–7**, bukan LSB.

### 2) Metode GUI: StegSolve (Data Extract)

1. **File → Open** gambar.
2. **Analyse → Data Extract**.
3. Setel:

   * **Bit Planes:** centang **Red 7**, **Green 7**, **Blue 7** saja.
   * **Extract By:** `Row`
   * **Bit Order:** `MSB First`
   * **Bit Plane Order:** `RGB`
4. Klik **Preview** → muncul teks panjang (Project Gutenberg…).
5. Klik **Save Bin** → `msb_payload.bin`.
6. Cari flag:

   ```bash
   strings msb_payload.bin | grep -o 'picoCTF{[^}]*}'
   # atau lihat offset persis:
   grep -abo 'picoCTF{' msb_payload.bin
   ```

   Pada file ini, `picoCTF{…}` muncul sekitar **byte offset 254,194**.

### 3) Metode CLI: Python (otomatis)

```bash
pip install pillow numpy
```

`extract_msb.py`:

```python
from PIL import Image
import numpy as np, re, sys

im = Image.open("Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png").convert("RGB")
a = np.array(im)

# Ambil bit-7 (MSB) tiap kanal R,G,B lalu rata kiri ke satu stream
bits = ((a >> 7) & 1).reshape(-1, 3).flatten()

# Bungkus per 8 bit → byte (big-endian)
n = (bits.size // 8) * 8
data = np.packbits(bits[:n].reshape(-1, 8), bitorder='big').tobytes()

# Cari flag
m = re.search(rb"picoCTF\{[^}]+\}", data)
print(m.group(0).decode() if m else "Flag tidak ditemukan")
```

Jalankan:

```bash
python extract_msb.py
```

Output:

```
picoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_572ad5fe}
```

---

## Penjelasan Teknis (singkat)

* **LSB steganography** umumnya tak merusak tampilan; perubahan kecil tak kasat mata.
* Di sini payload berada di **MSB** → mengubah nilai piksel secara drastis → **noise/korupsi** jelas terlihat.
* Mengekstrak **bit-7** dari setiap kanal (R→G→B) baris-demi-baris dan mengemasnya menjadi byte menghasilkan **teks ASCII** panjang; flag berada di dalamnya.

---

## Tips & Troubleshooting

* Jika **Preview** tak terbaca:

  * Coba ubah **Bit Plane Order** (RGB ↔ BGR/GRB).
  * Terakhir, coba **Bit Order = LSB First** (untuk challenge ini **MSB First** yang benar).
* Bisa juga “brute force kecil” semua kombinasi bit 0–7 dan bit order untuk mencari aliran yang menghasilkan ASCII wajar.

---

## Pelajaran

* **Artefak visual kuat ⇒ cek MSB/bit tinggi.**
* Jangan terpaku pada LSB—stego bisa menanam data pada berbagai bit-plane.
* StegSolve’s **Data Extract** + `strings/grep` adalah kombinasi cepat dan efektif.
