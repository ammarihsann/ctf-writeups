# Blast from the Past — picoCTF (Forensics)

**Goal:** “menua-kan” foto dengan menyetel semua timestamp ke
`1970:01:01 00:00:00.001+00:00` (atau waktu ekuivalen pada zona lain).
Untuk tag tanpa timezone, gunakan GMT (`+00:00`).

## Tools

* `exiftool` (baca/tulis metadata EXIF/XMP/IPTC)
* `exiv2` (opsional)
* Hex editor (Hexwalk/xxd/010 Editor) — untuk MakerNotes Samsung

## Langkah

### 1) Inspeksi metadata

```bash
exiftool -time:all -a -G1 -s original.jpg
```

Catat tag yang dicek checker (umumnya 7 tag):
`IFD0:ModifyDate`, `ExifIFD:DateTimeOriginal`, `ExifIFD:CreateDate`, sub-second, offset timezone, XMP dates, IPTC dates. Kadang ada **Samsung\:TimeStamp** (MakerNotes).

### 2) Set mayoritas timestamp dengan exiftool

Bikin salinan kerja:

```bash
cp original.jpg aged.jpg
```

Tulis EXIF/XMP/IPTC:

```bash
exiftool -overwrite_original \
  -AllDates="1970:01:01 00:00:00" \
  -SubSecTime=001 -SubSecTimeOriginal=001 -SubSecTimeDigitized=001 \
  -OffsetTime="+00:00" -OffsetTimeOriginal="+00:00" -OffsetTimeDigitized="+00:00" \
  "-XMP:CreateDate=1970:01:01 00:00:00.001+00:00" \
  "-XMP:ModifyDate=1970:01:01 00:00:00.001+00:00" \
  "-XMP:MetadataDate=1970:01:01 00:00:00.001+00:00" \
  "-IPTC:DateCreated=1970:01:01" "-IPTC:TimeCreated=00:00:00+00:00" \
  aged.jpg
```

Verifikasi cepat:

```bash
exiftool -time:all -a -G1 -s aged.jpg | egrep -i 'date|time'
```

### 3) Masalah khusus: **Samsung\:TimeStamp** (MakerNotes)

`exiftool` bisa **baca** tapi sering **tidak bisa tulis** tag ini. Nilainya adalah
**epoch dalam milidetik**. Contoh yang terlihat di file:

```
TimeStamp = 1700513181420  ->  2023:11:20 20:46:21.420+00:00
```

Target kita: **1 ms sejak epoch** → `1970:01:01 00:00:00.001+00:00`.

#### Solusi praktis (hex edit):

1. Cari offset value dengan:

```bash
exiftool -v3 -Samsung:all aged.jpg | grep -i "TimeStamp"
# catat "ValueOffset = 0x........" dari baris TimeStamp
```

2. Buka file di hex editor (atau pakai `dd`) dan tulis **8 byte little-endian** untuk angka `1`:

```
01 00 00 00 00 00 00 00
```

Contoh dengan `dd` (ganti `0xOFFSET`):

```bash
OFF=0xOFFSET
printf '\x01\x00\x00\x00\x00\x00\x00\x00' | dd of=aged.jpg bs=1 seek=$((OFF)) conv=notrunc
```

3. Cek ulang:

```bash
exiftool -n -Samsung:TimeStamp aged.jpg     # harus: 1
exiftool    -Samsung:TimeStamp aged.jpg     # harus: 1970:01:01 00:00:00.001+00:00
```

> Catatan: `exiv2` kadang bisa menulis `Exif.Samsung2.TimeStamp`, tapi pada beberapa build mapping MakerNotes Samsung tidak konsisten, sehingga hex edit adalah cara paling pasti.

### 4) Submit ke checker

```bash
nc <host> <port> < aged.jpg
# contoh event:
# nc mimas.picoctf.net 51610 < aged.jpg
```

Jika semua 7/7 cocok, checker akan menampilkan flag.

## Flag

```
picoCTF{71m3_7r4v311ng_p1c7ur3_3e336564}
```

## Insight

* EXIF standar bisa diubah “massal” dengan `-AllDates`, lalu presisi `.001` via `SubSecTime*` dan timezone via `OffsetTime*`.
* XMP/ IPTC perlu di-set eksplisit untuk jam + offset.
* MakerNotes (vendor-specific) sering tak bisa ditulis dengan `exiftool`; memahami **tipe data** (di sini: epoch ms) mempermudah patch manual.
