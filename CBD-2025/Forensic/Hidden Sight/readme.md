Siap! Berikut write-up rapi untuk tantangan forensic **Hidden Sight** — lengkap dari nol sampai ketemu flag yang kamu susun manual.

# Ringkasan

* Diberikan: `hidden.zip` + password.
* Di dalam ZIP (AES): `btr.jpg` dan umpan `bcache24.bmc`.
* `btr.jpg` menyimpan **ZIP tersembunyi** di bagian ekor (after `FFD9`) → berisi `Cache0000.bin`.
* `Cache0000.bin` adalah **RDP bitmap cache** (ubin 64×64 BGRA).
* Menyusun potongan ubin yang berisi teks merah ⇒ terbaca kalimat flag.

**Flag:** `CBD{1_d0nT_wH4t_I_44mm_do1nGg_19Nik1j4}`

---

# Langkah Detail

## 1) Ekstrak ZIP terenkripsi (WinZip AES)

Password dari soal:
`fd9fbac804de39ba121c41173923a86f1702f1c290294f3abc2d2544bc9d93ef`

Paling gampang pakai 7-Zip:

```bash
7z x -y -pfd9fbac804de39ba121c41173923a86f1702f1c290294f3abc2d2544bc9d93ef hidden.zip
# hasil: btr.jpg dan bcache24.bmc (kosong/umpan)
```

> Catatan: `unzip` bawaan kadang belum dukung AES penuh, jadi 7-Zip lebih aman.

## 2) Cari ZIP tersembunyi di dalam `btr.jpg`

Pakai `binwalk` untuk mendeteksi struktur ZIP di trailing bytes:

```bash
binwalk btr.jpg

# contoh output relevan:
# 77878  0x13036  Zip archive data ... name: Cache0000.bin
# 2937132 0x2CD12C End of Zip archive, footer length: 22
```

Ekstraksi otomatis:

```bash
binwalk -e btr.jpg
# akan muncul _btr.jpg.extracted/13036.zip
unzip -o _btr.jpg.extracted/13036.zip -d _btr.jpg.extracted/
# hasil: _btr.jpg.extracted/Cache0000.bin
```

Atau carve manual (jika perlu angka dari binwalk):

```bash
# ukuran total = (EOCD+22) - start_offset
dd if=btr.jpg of=cache.zip iflag=skip_bytes,count_bytes skip=77878 count=$((2937132+22-77878))
unzip -o cache.zip -d cache_out
# hasil: cache_out/Cache0000.bin
```

## 3) Kenali format: **RDP Bitmap Cache**

`Cache0000.bin` adalah cache grafik sesi RDP (Remote Desktop) berisi **tile 64×64** RGBA/BGRA.
Kita bisa mengekstrak semua tile & merangkainya memakai **bmc-tools** (ANSSI).

Install & jalankan:

```bash
git clone https://github.com/ANSSI-FR/bmc-tools
cd bmc-tools
python3 -m venv venv && source venv/bin/activate
pip install pillow numpy

# ekstrak tile dan buat kolase (ubah path sumber sesuai hasilmu)
python3 bmc-tools.py \
  -s "/path/ke/Cache0000.bin" \
  -d out_tiles \
  -b -w 32
# output: out_tiles/*.bmp dan out_tiles/collage.bmp
```

## 4) Fokus ke “pita atas” (tempat teks merah)

Supaya tulisan jelas, crop 24 piksel teratas setiap tile lalu gabung jadi satu baris:

```bash
OUT_DIR="/path/ke/bmc-tools/out_tiles"
WORK="/path/ke/bmc-tools/check"
mkdir -p "$WORK" && cd "$WORK"

# crop band atas 24px
for f in $(ls -1v "$OUT_DIR"/*.bmp); do
  convert "$f" -crop 64x24+0+0 +repage "band_$(basename "$f" .bmp).png"
done

# susun 1 baris panjang biar terbaca menyambung
montage band_*.png -tile x1 -geometry +0+0 all_row.png
xdg-open all_row.png
```

Kamu juga bisa hanya menggabungkan rentang tile yang jelas berisi huruf (contoh dari screenshot: 59–71):

```bash
convert "$OUT_DIR"/Cache0000.bin_15_{59..71}.bmp +append row_59-71.png
# atau crop dulu lalu +append pada band_59..71
```

## 5) Baca dan susun flag

Dari deretan ubin teks merah (huruf agak terpotong karena per-ubin), terbaca bertahap:

* `CBD{` … `I_` … `d0nT` … `_wH4t_` … `I_44mm_` … `do1nGg_` … `19Nik1j4` `}`

Beberapa karakter terlihat “menipu” karena font & potongan:

* `1` ↔ `I` (huruf I vs angka 1)
* `0` ↔ `o`
* `mm` terlihat ganda (cache duplikat) → sesuai visual

Akhirnya didapat:

**`CBD{1_d0nT_wH4t_I_44mm_do1nGg_19Nik1j4}`**

> Kamu sudah menyusun manual (nice!). Langkah di atas menunjukkan cara otomatis/terarah untuk mencapai bacaan yang sama.

---

## Alternatif: extractor Python singkat (tanpa bmc-tools)

Kalau tidak mau clone repo:

```python
from PIL import Image
data = open("Cache0000.bin","rb").read()
T = 64*64*4
tiles = []
i = 0; n = 0
while True:
    i = data.find((T).to_bytes(4,'little'), i+1)
    if i == -1: break
    start = i+4; end = start+T
    if end > len(data): break
    raw = data[start:end]
    # heuristik: cukup banyak piksel "gelap" di alpha==FF → tile valid
    if raw[3::4].count(0xFF) > (64*64*0.3):
        Image.frombytes("RGBA",(64,64),raw,"raw","BGRA").save(f"tile_{n:05d}.png")
        n += 1
print("tiles:", n)
```

Lalu crop & gabung seperti pada langkah 4.

---

## Pitfall & Tips

* **AES ZIP**: pakai 7-Zip jika `unzip` menolak password.
* **Path salah**: gunakan path absolut untuk `-s` (sumber) & `-d` (tujuan).
* **Huruf mirip angka**: validasi dengan konteks kata (mis. `I` vs `1`, `o` vs `0`).
* **Kolase**: atur `-w` (jumlah tile per baris) agar potongan tulisan berhimpit rapi.

Selesai. Kalau mau, aku bisa rapikan **script satu-kali-jalan** yang dari `btr.jpg` langsung keluarkan `flag.txt`.
