```
Heker Menyerang 

Kita telah diserang dengan serangan besar-besaran oleh hacker ganas, dan tim SOC memberikan log Snort kepada Anda.
Tim Anda ditugaskan untuk menghitung berapa banyak serangan yang terjadi pada ASP.NET Web Pages yang terkait dengan
Database Injection Attack (SQL) dan juga serangan yang melibatkan RFI (Remote File Inclusion) pada aplikasi web PHP.

Format Flag: playPNJ{jumlah}

Contoh : Jika serangan yang dihitung totalnya ada 100 kali maka jawabannya adalah playPNJ{100}

Author : kurokaze (Zahir)

```
# Ringkasan

* Yang diminta: hitung **(1)** semua **SQL Injection** yang menarget **ASP/ASP.NET pages** (`.asp` & `.aspx`) **+** **(2)** semua **Remote File Inclusion (RFI)** pada **aplikasi PHP**.
* Hasil perhitungan pada `snort.log` yang kamu upload:

  * **SQLi → ASP/ASP.NET**: **807**
  * **RFI → PHP**: **204**
  * **Total** = 807 + 204 = **1011** → **`playPNJ{1011}`** ✅

# Dasar teknis (rujukan)

* **SQL Injection (SQLi)**: injeksi kueri ke DB melalui input aplikasi. (OWASP) ([OWASP Foundation][1])
* **Remote File Inclusion (RFI)**: celah yang memungkinkan aplikasi meng-include file jarak jauh (umumnya di PHP) lewat parameter path yang tidak disaring. (OWASP) ([OWASP Foundation][2])
* **ASP/ASP.NET pages**: klasik **ASP** berekstensi `.asp`; **ASP.NET Web Forms** berekstensi `.aspx`. (Microsoft Docs/GeeksforGeeks) ([Microsoft Learn][3])
* **Teknik filter log** dengan `grep` dan opsi yang relevan. (GNU grep manual) ([gnu.org][4])
* **Snort log/alert** konsep umum. (Snort docs) ([docs.snort.org][5])

# Langkah penyelesaian (yang bisa kamu ulang)

> Asumsi file berada di direktori kerja sebagai `snort.log`.

## 1) Hitung SQLi yang menarget ASP/ASP.NET

Fokus **hanya** pada signature bertema SQLi **yang menyebut target berakhiran `.asp`/`.aspx`**.

```bash
# Case-insensitive, ambil baris SQLi dulu
grep -iE "sql[[:space:]-]*injection" snort.log \
  | grep -Eio "\.aspx?(\b|[^A-Za-z])" \
  | wc -l
```

Penjelasan singkat:

* `-i` = case-insensitive; `-E` = regex lanjutan; `-o` = tampilkan hanya bagian yang match. (GNU grep) ([gnu.org][6])
* Pola `\.aspx?` mencakup `.asp` dan `.aspx`.

**Output yang didapat:** `807`.

## 2) Hitung RFI pada aplikasi PHP

Filter signature RFI dan batasi ke konteks **PHP** dengan pencocokan `.php`.

```bash
grep -i "remote file inclusion" snort.log \
  | grep -Eio "\.php(\b|[^A-Za-z])" \
  | wc -l
```

**Output yang didapat:** `204`.

> Catatan: banyak signature ET/GPL menuliskan frasa **“Remote File Inclusion”** untuk RFI sehingga pencocokan string ini stabil di berbagai dataset Snort. (Snort/ET ekosistem) ([docs.snort.org][5])

## 3) Jumlahkan & format flag

* **807 + 204 = 1011** → **`playPNJ{1011}`**.

## 4) Validasi cepat (spot-check)

Ambil beberapa contoh baris untuk cek konteks benar:

```bash
# Lihat sampel SQLi → ASP/ASPX
grep -niE "sql[[:space:]-]*injection" snort.log | grep -iE "\.aspx?"

# Lihat sampel RFI → PHP
grep -ni "remote file inclusion" snort.log | grep -i "\.php"
```

Pastikan baris contoh memang menampilkan signature bertema **SQLi** yang menunjuk resource `.asp/.aspx`, serta **RFI** yang menyebut file **`.php`**.

# Alternatif di Windows PowerShell

Jika kamu kerja di PowerShell:

```powershell
# SQLi ke ASP/ASPX
(Get-Content snort.log | Select-String -Pattern 'sql\s*-*injection' -CaseSensitive:$false) |
  Select-String -Pattern '\.aspx?(\b|[^A-Za-z])' -AllMatches |
  Measure-Object | % {$_.Count}

# RFI ke PHP
(Get-Content snort.log | Select-String -Pattern 'remote file inclusion' -CaseSensitive:$false) |
  Select-String -Pattern '\.php(\b|[^A-Za-z])' -AllMatches |
  Measure-Object | % {$_.Count}
```

# Kenapa pendekatan ini tepat?

* **Sesuai deskripsi soal**: yang diminta adalah **jumlah serangan**, bukan semua alert web generic. Kita **membatasi**:

  * hanya **SQLi** untuk **ASP/ASPX** (ASP/ASP.NET pages) — `.asp`/`.aspx` adalah ekstensi yang merepresentasikan keduanya. (Microsoft/GeeksforGeeks) ([Microsoft Learn][3])
  * hanya **RFI** untuk **aplikasi PHP** — RFI umumnya dieksploit di PHP via parameter include/require; filter `.php` memastikan relevansi. (OWASP) ([OWASP Foundation][2])
* **Tooling standar & reproducible**: `grep` + regex/opsi yang didokumentasikan resmi. (GNU grep) ([gnu.org][4])

---

[1]: https://owasp.org/www-community/attacks/SQL_Injection?utm_source=chatgpt.com "SQL Injection"
[2]: https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.2-Testing_for_Remote_File_Inclusion?utm_source=chatgpt.com "Testing for Remote File Inclusion"
[3]: https://learn.microsoft.com/en-us/previous-versions/aspnet/k33801s3%28v%3Dvs.100%29?utm_source=chatgpt.com "ASP.NET Web Page Syntax Overview"
[4]: https://www.gnu.org/software/grep/manual/html_node/index.html?utm_source=chatgpt.com "Top (GNU Grep 3.12)"
[5]: https://docs.snort.org/start/alert_logging?utm_source=chatgpt.com "Alert Logging"
[6]: https://www.gnu.org/software/grep/manual/html_node/Command_002dline-Options.html?utm_source=chatgpt.com "Command-line Options (GNU Grep 3.12)"
