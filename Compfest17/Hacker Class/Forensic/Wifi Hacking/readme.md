## 📝 Write-up CTF Forensic — “UU Tanah”

### **Deskripsi Soal**

* **Kategori:** Forensic
* **File diberikan:** `chall.pcap`
* **Hint:** *"Can you find who this handshake belongs to and crack the password? Use it to login to the service below"*
* **Service:**

  ```
  nc ctf.compfest.id 7501
  ```
* Tujuan: Temukan pemilik handshake (SSID + username) dan crack password untuk login ke service dan mendapatkan flag.

---

### **Langkah Penyelesaian**

#### **1. Analisis file `.pcap`**

Buka file `chall.pcap` di Wireshark:

```bash
wireshark chall.pcap
```

Gunakan filter:

```
eapol
```

Terlihat ada WPA/WPA2 4-way handshake antara AP (BSSID `E4:95:6E:45:90:24`) dan sebuah client.

---

#### **2. Temukan SSID**

Karena handshake WPA tidak menyimpan SSID langsung di EAPOL, cari paket Beacon atau Probe Response:

```
wlan.ssid
```

Ditemukan SSID:

```
joesheer
```

---

#### **3. Ekspor handshake minimal**

Untuk mempercepat crack, hanya simpan:

* 1 paket Beacon SSID `joesheer`
* Semua paket EAPOL

Gunakan filter gabungan:

```
(wlan.fc.type_subtype == 8 && wlan.ssid == "joesheer") || eapol
```

Pilih semua hasil filter → **File → Export Specified Packets → Selected packets**
Simpan sebagai:

```
handshake_minimal.pcap
```

---

#### **4. Crack password dengan rockyou.txt**

Jalankan `aircrack-ng` dengan wordlist `rockyou.txt` bawaan Kali Linux, simpan hasil password ke file:

```bash
aircrack-ng handshake_minimal.pcap -w /usr/share/wordlists/rockyou.txt -e "joesheer" -l found.txt -p 7
```

Hasil:

```
KEY FOUND! [ nanotechnology ]
```

Password: **`nanotechnology`**

---

#### **5. Login ke service**

Gunakan username **joesheer** dan password hasil crack:

```bash
nc ctf.compfest.id 7501
```

Input:

```
Username: joesheer
Password: nanotechnology
```

Output:

```
Login successful!
Flag: COMPFEST17{<isi_flag_asli>}
```

---

### **Flag**

```
COMPFEST17{<isi_flag_asli>}
```

---

### **Catatan**

* WPA handshake tidak mengirim password secara plaintext.
* Hanya Beacon + EAPOL yang dibutuhkan untuk crack.
* Opsi `-l` di aircrack-ng menyimpan hasil password ke file.
* `rockyou.txt` sangat efektif untuk CTF karena berisi banyak password umum.

---

