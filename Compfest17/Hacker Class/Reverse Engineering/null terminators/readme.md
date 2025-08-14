# Write-up Singkat — “where did my null terminators Go??” (Reverse Engineering)

## Inti Soal

* **Binary:** ELF x86-64, dibuat dengan **Go**.
* **Kenapa `strings | grep COMPFEST17` gagal?** String di Go bukan C-string (tidak diakhiri `\0`) — flag tidak tersimpan utuh sebagai teks polos.
* **Cara kerja checker:** Program menyimpan **tabel 241 string** di `.rodata`. Untuk tiap indeks `i`, dia mengambil **1 byte** dari string ke-`i` pada posisi `idx = (i ^ 0x17) % len` dan membandingkannya dengan `input[i]`. Gabungan byte-byte itu adalah flag.

## Langkah Solusi (ringkas)

1. Identifikasi Go binary & jalankan untuk lihat prompt:

   ```bash
   file chall     # ELF 64-bit, Go build
   ./chall        # prompt: "flag?"
   ```
2. RE cepat (opsional): di `main.checkInput`, terlihat blok salin `(ptr,len)*(N)` dari `.rodata` dan loop per-byte.
3. **Ekstrak** tabel `(ptr,len)` dari `.rodata` lalu rakit flag dengan rumus `idx = (i ^ 0x17) % len`.

---

## Kode Ekstraktor (solve.py)

> Simpan sebagai `solve.py` di folder yang sama, lalu: `python3 solve.py`

```python
#!/usr/bin/env python3
from elftools.elf.elffile import ELFFile
import struct, io

BIN = "chall"
XOR_CONST = 0x17

def vaddr_to_off(elf, vaddr):
    for seg in elf.iter_segments():
        if seg['p_type'] == 'PT_LOAD':
            start = seg['p_vaddr']
            end   = start + seg['p_memsz']
            if start <= vaddr < end:
                return (vaddr - start) + seg['p_offset']
    raise ValueError(f"VA 0x{vaddr:x} not in any PT_LOAD segment")

with open(BIN, "rb") as f:
    data = f.read()              # baca SELURUH file dulu
    elf  = ELFFile(io.BytesIO(data))

    ro = elf.get_section_by_name(".rodata")
    ro_vaddr = ro['sh_addr']
    ro_off   = ro['sh_offset']
    ro_size  = ro['sh_size']
    ro_bytes = data[ro_off: ro_off + ro_size]

    # Scan .rodata untuk rangkaian (ptr,len)*(N) terpanjang yang valid
    best_start = None
    best_count = 0

    # coba offset step 8 biar aman terhadap alignment
    for start in range(0, ro_size - 16, 8):
        count = 0
        while True:
            off = start + count * 16
            if off + 16 > ro_size:
                break
            ptr, ln = struct.unpack_from("<QQ", ro_bytes, off)
            # valid jika panjang wajar dan pointer menunjuk ke dalam .rodata
            if ln == 0 or ln > 1 << 20:
                break
            if not (ro_vaddr <= ptr < ro_vaddr + ro_size):
                break
            count += 1
        if count > best_count:
            best_count, best_start = count, start

    if best_count < 10:
        raise RuntimeError("Gagal menemukan tabel (ptr,len) yang masuk akal di .rodata")

    table_vaddr = ro_vaddr + best_start
    table_file_off = ro_off + best_start
    print(f"[i] Tabel ditemukan @ .rodata+0x{best_start:x} (VA 0x{table_vaddr:x}) dengan {best_count} entri")

    # Rekonstruksi flag
    out = []
    for i in range(best_count):
        off = table_file_off + i*16
        ptr, ln = struct.unpack_from("<QQ", data, off)
        s_off = vaddr_to_off(elf, ptr)
        s = data[s_off: s_off + ln]
        idx = (i ^ XOR_CONST) % ln
        out.append(chr(s[idx]))
    flag = ''.join(out)
    print(flag)
```

### Penjelasan Kode (inti per bagian)

* `ELFFile(...)` + `get_section_by_name(".rodata")`: membuka ELF dan mengambil section `.rodata` (tempat pointer dan bytes string berada).
* **Pemindaian tabel `(ptr,len)`**:

  * Go string = `(ptr: 8B, len: 8B)` → tiap entri 16 byte.
  * Loop `start` melangkah **8 byte** (alignment aman) mencari deret `(ptr,len)` terpanjang yang valid:

    * `ln` wajar (`0 < ln <= 1<<20` sebagai sanity check),
    * `ptr` harus menunjuk **ke dalam `.rodata`** (flag & string sumbernya ada di sana).
  * Deret terpanjang diambil sebagai kandidat tabel.
* `vaddr_to_off(...)`: konversi **virtual address → file offset** menggunakan segmen `PT_LOAD`. Wajib agar bisa membaca bytes string dari file.
* **Rekonstruksi flag**:

  * Untuk entri ke-`i`, baca string sumber `s` melalui `(ptr,len)`,
  * Hitung `idx = (i ^ XOR_CONST) % len(s)`,
  * Ambil byte `s[idx]` dan `append` ke `out`.
  * Terakhir `out.append('}')` — **catatan:** kalau outputmu jadi `...}}`, hapus baris ini; beberapa bina­ri sudah menyumbang `}` sebagai byte terakhir tabel, jadi tidak perlu ditambah manual.
* Cetak lokasi tabel & hasil flag.

---

## Hasil & Validasi

Contoh output:

```
[i] Tabel ditemukan @ .rodata+0x48ae0 (VA 0x4cbae0) dengan 241 entri
COMPFEST17{g0_5tr1ngs_4r3_r34lly_w31rd_huh??_anyw4y_h3r3s_s0me_l0ng_h4sh_s0_1_kn0w_y0u_d3fin3tely_used_4_scr1p7_7d98b598bdf7d9a7894287dfc805b862581a86433e36b20e45f58c90008cad8aa9cf581aa1427ef8472244b6735618e634c5132947bab4a4e53e164f3c2ec08b}
```

Masukkan ke program:

```
flag? <paste flag>
yes thats the flagg!!!
```

---

## Catatan Cepat

* Batas `ln > 1 << 20` hanya **sanity check** agar tidak tersesat — aman untuk kebanyakan chall.
* Teknik ini reusable: selama ada pola penyebaran `(ptr,len)` + indeks deterministik, cukup ganti konstanta XOR jika berbeda.
