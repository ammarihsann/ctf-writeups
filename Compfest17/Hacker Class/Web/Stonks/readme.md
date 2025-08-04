### **Write-up CTF Compfest: stonks [Web]**

Write-up ini akan membahas penyelesaian tantangan "stonks" dari kategori Web pada kompetisi CTF Compfest. Tantangan ini memiliki beberapa kemungkinan jalur penyelesaian, namun kita akan fokus pada jalur yang paling efisien: menemukan kredensial yang lemah.

#### **Informasi Challenge**

| Kategori      | Poin | Deskripsi                                                                                                                              |
| :------------ | :--- | :------------------------------------------------------------------------------------------------------------------------------------- |
| **Web** | 100  | *Times were wild before the email apocalypse. There were even sites giving out free money that also supported currency conversions\!* |
| **Lampiran** | `chall.zip` (berisi source code aplikasi)                                                                                                      |
| **URL** | `http://ctf.compfest.id:7303`                                                                                                                  |

-----

### **Langkah 1: Enumerasi & Pengintaian (Reconnaissance)**

Saat pertama kali mengunjungi situs, kita disambut dengan halaman utama yang menawarkan "uang gratis" dan fitur konversi mata uang. Terdapat juga tautan untuk menuju halaman `/login` dan `/register`.

Langkah pertama yang krusial adalah memahami pengguna yang ada di sistem. Cara termudah untuk melakukannya adalah melalui halaman registrasi.

1.  Buka halaman `http://ctf.compfest.id:7303/register`.
2.  Coba daftarkan akun dengan username yang umum untuk seorang administrator, yaitu `admin`, dengan password acak.
3.  Saat form disubmit, situs memberikan pesan error: **"Username already exists"**.

Ini adalah celah keamanan klasik yang disebut **User Enumeration**. Melalui pesan error yang berbeda antara username yang ada dan yang tidak ada, kita bisa mengonfirmasi bahwa akun dengan username `admin` benar-benar ada di dalam sistem.

### **Langkah 2: Eksploitasi - Menemukan Kredensial Lemah**

Setelah mengetahui bahwa user `admin` ada, langkah logis berikutnya adalah mencoba login menggunakan password yang umum atau mudah ditebak.

1.  Buka halaman `http://ctf.compfest.id:7303/login`.
2.  Masukkan kredensial berikut:
      * **Username**: `admin`
      * **Password**: `admin`
3.  Klik tombol "Login".

Secara mengejutkan, login berhasil\! Kita langsung diarahkan ke halaman utama sebagai admin, dan flag langsung ditampilkan di layar.

#### **Flag**

```
COMPFEST17{only_check_from_user_session_1s_unsafe_hshshshs}
```

-----

### **Analisis Solusi Alternatif (Intended Solution)**

Meskipun kita sudah mendapatkan flag dengan cara yang sangat sederhana, teks dari flag itu sendiri memberikan petunjuk besar mengenai jalur penyelesaian lain yang kemungkinan dimaksudkan oleh pembuat soal.

**Flag**: `COMPFEST17{only_check_from_user_session_1s_unsafe_hshshshs}`
**Artinya**: "Mengecek (sesuatu) hanya dari sesi pengguna itu tidak aman."

Ini mengisyaratkan adanya **Business Logic Flaw** di mana aplikasi terlalu mempercayai data yang tersimpan di *session cookie* pengguna.

**Hipotesis Skenario Eksploitasi yang Diinginkan:**

1.  **Mencari Mata Uang Tersembunyi**: Petunjuk "email apocalypse" di deskripsi mengisyaratkan adanya fitur/mata uang lama. Dengan menganalisis *source code* dari `chall.zip` (terutama file JavaScript), seorang pemain kemungkinan akan menemukan mata uang yang sudah tidak ditampilkan di UI (misalnya `BTC` atau mata uang fiktif lain) tetapi masih berfungsi di *backend*.
2.  **Memanfaatkan Session**: Aplikasi kemungkinan menyimpan mata uang default pengguna di dalam *session cookie*.
3.  **Celah Logika Konversi**: Saat melakukan konversi, *backend* mungkin mengambil mata uang **`From`** (sumber) dari *session*, bukan dari parameter `POST` yang dikirim pengguna. Ini membuka celah manipulasi.
4.  **Eksploitasi**:
      * Seorang pemain akan mendaftar akun biasa.
      * Mengubah mata uang default mereka di *session* menjadi mata uang tersembunyi yang menguntungkan.
      * Menggunakan trik konversi **angka negatif** (misalnya, mengonversi `-1,000,000` dari mata uang biasa ke mata uang lain) untuk melipatgandakan uang secara eksponensial. Kombinasi dari pengambilan data yang tidak aman dari *session* dan kegagalan validasi input negatif akan memungkinkan pemain untuk mendapatkan uang dalam jumlah tak terbatas dan akhirnya mendapatkan flag.

### **Kesimpulan dan Pelajaran**

Tantangan "stonks" ini adalah contoh bagus di mana ada beberapa jalan menuju solusi.

1.  **User Enumeration**: Aplikasi tidak boleh memberikan pesan error yang berbeda untuk username yang valid dan tidak valid.
2.  **Weak Credentials**: Akun dengan hak istimewa (seperti `admin`) tidak boleh menggunakan password yang lemah dan mudah ditebak. Ini seringkali menjadi "pintu belakang" yang tidak disengaja dalam sebuah sistem.
3.  **Insecure Business Logic**: Seperti yang ditunjukkan oleh flag, jangan pernah mempercayai data yang dikirim oleh klien secara membabi buta, termasuk data yang disimpan dalam *session*. Selalu lakukan validasi di sisi server untuk setiap operasi kritis.

Pada akhirnya, solusi tercepat adalah yang terbaik dalam sebuah kompetisi CTF. Menemukan kredensial `admin:admin` adalah demonstrasi dari pemikiran *out-of-the-box* dan efisien.
