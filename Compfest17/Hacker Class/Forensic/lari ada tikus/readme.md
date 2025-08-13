## 📝 Write-up: Forensic – "lari ada tikus" (COMPFEST 17)

**Category:** Forensic
**Author:** jay
**Flag format:** `COMPFEST17{UPPERCASE_LETTERS_ONLY}`

---

### **1. Analisis awal file**

Dari soal, kita diberi sebuah file `speed.jpg`.
Langkah pertama, cek informasi file:

```bash
file speed.jpg
```

Hasilnya menunjukkan ini adalah **JPEG image**.
Lanjut cek struktur file dengan **binwalk**:

```bash
binwalk speed.jpg
```

Output:

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01
20919         0x51B7          RIFF audio data (WAV), PCM, 1 channels, 11050 sample rate
```

Di dalam gambar ada **file WAV** tersembunyi mulai offset `0x51B7`.

---

### **2. Ekstraksi file audio tersembunyi**

Karena binwalk kadang gagal ekstrak otomatis, kita ekstrak manual dengan `dd`:

```bash
dd if=speed.jpg of=hidden.wav bs=1 skip=20919
```

Sekarang kita punya file `hidden.wav`.

---

### **3. Analisis audio**

Putar audio:

```bash
aplay hidden.wav
```

Terdengar suara **beep pendek dan panjang** → indikasi **kode Morse**.

---

### **4. Decode kode Morse**

Bisa decode dengan tools seperti:

* [morsecode.world decoder](https://morsecode.world/international/decoder/audio-decoder-adaptive.html)
* `morse-audio-decoder`
* Python script custom untuk ekstraksi sinyal dan konversi.

Hasil decoding memberi teks:

```
KOKMORSEADADICTFKIRAINHANYAADADIPRAMUKA
```

---

### **5. Menentukan flag**

Awalnya dikira ini adalah password untuk membuka data `steghide`, tapi sesuai catatan soal (`all letters inside the flag uses only uppercase letters`), ternyata teks ini **langsung** adalah isi flag (dibungkus format COMPFEST17).

Sehingga flag final:

```
COMPFEST17{KOKMORSEADADICTFKIRAINHANYAADADIPRAMUKA}
```

---

### **6. Kesimpulan**

* File `speed.jpg` mengandung **WAV audio** di dalamnya.
* Audio tersebut berisi **kode Morse**.
* Decode Morse → dapatkan teks uppercase.
* Teks tersebut adalah flag.

---

**Final Flag:**

```
COMPFEST17{KOKMORSEADADICTFKIRAINHANYAADADIPRAMUKA}
```
