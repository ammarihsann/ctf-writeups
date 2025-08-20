# picoCTF 2025 – Forensics

## Challenge: Event-Viewing

**Author:** Venax
**Category:** Forensics

---

## 📝 Deskripsi

Seorang karyawan melaporkan komputernya langsung shutdown setiap kali login.
Petunjuk kasus:

1. Mereka menginstal software dari internet.
2. Software dijalankan, terlihat "tidak melakukan apa-apa".
3. Setelah reboot, komputer langsung shutdown begitu login.

Kita diberikan file **Windows\_Logs.evtx**. Tugas kita adalah mencari bukti di event log dan menemukan flag (terbagi menjadi 3 bagian).

---

## 🔧 Analisis dengan Chainsaw

Untuk menganalisis log, saya menggunakan **Chainsaw**, sebuah tool cepat untuk memfilter dan mencari event log Windows.

---

### 1. Instalasi Software (Event ID 1033 – `MsiInstaller`)

Jalankan:

```bash
chainsaw search filename.evtx -e 1033
```

Output menunjukkan software bernama **Totally\_Legit\_Software** dengan string base64:

```
cGljb0NURntFdjNudF92aTN3djNyXw==
```

Decode:

```bash
echo 'cGljb0NURntFdjNudF92aTN3djNyXw==' | base64 -d
```

→ `picoCTF{Ev3nt_vi3wv3r_`

📌 **Flag Part 1:** `picoCTF{Ev3nt_vi3wv3r_`

---

### 2. Persistence via Registry (Event ID 4657 – `Security`)

Jalankan:

```bash
chainsaw search filename.evtx -e 4657
```

Output menunjukkan **Registry modification** pada key `HKLM\Software\Microsoft\Windows\CurrentVersion\Run` dengan value:

```
MXNfYV9wcjN0dHlfdXMzZnVsXw==
```

Decode:

```bash
echo 'MXNfYV9wcjN0dHlfdXMzZnVsXw==' | base64 -d
```

→ `1s_a_pr3tty_us3ful_`

📌 **Flag Part 2:** `1s_a_pr3tty_us3ful_`

---

### 3. Shutdown Trigger (Event ID 1074 – `User32`)

Jalankan:

```bash
chainsaw search filename.evtx -e 1074
```

Output menunjukkan **shutdown event** dengan argumen base64:

```
dDAwbF84MWJhM2ZlOX0=
```

Decode:

```bash
echo 'dDAwbF84MWJhM2ZlOX0=' | base64 -d
```

→ `t00l_81ba3fe9}`

📌 **Flag Part 3:** `t00l_81ba3fe9}`

---

## 🏁 Flag Lengkap

Gabungkan ketiga bagian:

```
picoCTF{Ev3nt_vi3wv3r_1s_a_pr3tty_us3ful_t00l_81ba3fe9}
```

✅ **Final Flag:**
**`picoCTF{Ev3nt_vi3wv3r_1s_a_pr3tty_us3ful_t00l_81ba3fe9}`**

---

## 📚 Lessons Learned

* **Event ID 1033 (MsiInstaller):** bukti instalasi software.
* **Event ID 4657 (Security – Registry Modification):** persistence / auto-run malware.
* **Event ID 1074 (User32):** shutdown event.
* **Chainsaw** sangat mempermudah analisis forensik karena bisa langsung filter log berdasarkan Event ID dan string tertentu.
