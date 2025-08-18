# LockSpill — Forensic Write-up (CBD CTF)

**Category:** Forensic  
**Difficulty:** —  
**Author’s blurb:**  
> A sudden system crash at company left behind a password vault and a memory dump captured at the exact moment of failure. Rumors say the vault holds fragments of a secret project…

---

## ✅ TL;DR

- **Flag:** `CBD{wh4t_15_Th1s_V4uLt_n1j1k4a_k3ePpass_8h17d1}`  
- **Core idea:** The system crashed while a KeePass vault was open/handled, leaving **decrypted application strings** in RAM. Among them, the real flag is **split into two Base58-encoded parts** in a Notes field. A decoy flag is also present to mislead.

---

## 🧰 Given Artifacts

- `dump.dmp` — Windows memory dump at crash time (≈ 244–250 MB).  
- `lock.kdbx` — KeePass database (~3 KB).

> We verified `lock.kdbx` as **KDBX 3.1**, AES cipher, GZip compression, **transform rounds: 6000**, inner stream **Salsa20**. (See Appendix A.)  
> **However, cracking the DB was unnecessary** because decrypted content (Notes/fields) was present in memory.

---

## 🧭 Approach Overview

1. **Triage the memory dump** for relevant strings (UTF‑16LE is common on Windows).
2. Look for **KeePass signatures and field names**: `Notes`, `Password`, `Protected`, `Title`, `Entry`, etc.
3. Notice a **decoy**: `CBD{do_you_think_this_is_flag}` followed by **“OF COURSE NOT”**.
4. Extract **two Base58 parts** labeled `Part 1 58:` and `Part 2 58:` from the same region (Notes), then **Base58-decode** them and **concatenate** to get the flag.

---

## 🔎 Detailed Steps

### 1) Memory Strings (UTF‑16LE)

Because Windows GUI apps often store strings in UTF‑16LE, use `strings -el` (little‑endian, 16-bit) first:

```bash
# UTF-16LE strings, then filter for KeePass artifacts and potential flags
strings -el dump.dmp | tee strings_u16.txt \
  | grep -n -E "CBD\\{|Notes|Password|Protected|Title|Entry|keepass|kdbx|Base58|Part\\s*58"
```

Findings of interest:

- Multiple occurrences of **`CBD{do_you_think_this_is_flag}`** (e.g., around offset `0x00D7…` region in the dump) with nearby text:
  > `CBD{do_you_think_this_is_flag} OF COURSE NOT`
- **Notes** sections that include **“Part 1 58:”** and **“Part 2 58:”** followed by character sets that match **Base58**.

Example snippets (from the UTF‑16 region around ~`0x00D7xxxx`):
```
...,CBD{do_you_think_this_is_flag}
OF COURSE NOT
...
Part 1 58: 2PaEBnQJx9Z56XrvhRCTE1tcbSX1VCLe
...
Part 2 58: B3eXWB4yupKPYUde1VhvrBDsq91jRY2tt
...
```

> We also saw KeePass‑related symbols like `KeePassLib.Security.Protected*`, which further supports the hypothesis that KeePass data structures were in memory at crash time.

### 2) Ignore the Decoy

The **“CBD{do_you_think_this_is_flag}”** string is a classic decoy:
- It’s immediately followed by **“OF COURSE NOT”** in the same memory region.
- No complementary context (e.g., split markers) surrounds it.
- The Base58 split parts **do** have explicit “Part X 58” tags, which is a strong signal.

### 3) Decode the Base58 Parts

Two parts were extracted from Notes:

- **Part 1:** `2PaEBnQJx9Z56XrvhRCTE1tcbSX1VCLe`  
  **Base58 →** `CBD{wh4t_15_Th1s_V4uLt_`

- **Part 2:** `B3eXWB4yupKPYUde1VhvrBDsq91jRY2tt`  
  **Base58 →** `n1j1k4a_k3ePpass_8h17d1}`

Concatenate:
```
CBD{wh4t_15_Th1s_V4uLt_ + n1j1k4a_k3ePpass_8h17d1} 
= CBD{wh4t_15_Th1s_V4uLt_n1j1k4a_k3ePpass_8h17d1}
```

A minimal Python decoder used:

```python
B58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"
IDX = {c: i for i, c in enumerate(B58)}

def b58decode(s):
    n = 0
    for ch in s:
        n = n * 58 + IDX[ch]
    out = bytearray()
    while n > 0:
        n, r = divmod(n, 256)
        out.append(r)
    out.reverse()
    # Preserve leading zeros for each leading '1'
    pad = len(s) - len(s.lstrip('1'))
    return b"\x00" * pad + bytes(out)

p1 = "2PaEBnQJx9Z56XrvhRCTE1tcbSX1VCLe"
p2 = "B3eXWB4yupKPYUde1VhvrBDsq91jRY2tt"

print(b58decode(p1))  # b'CBD{wh4t_15_Th1s_V4uLt_'
print(b58decode(p2))  # b'n1j1k4a_k3ePpass_8h17d1}'
```

### 4) (Optional) About the KDBX

Parsing the `lock.kdbx` header shows:
- **Format:** KDBX 3.1
- **Cipher:** AES  
- **Compression:** GZip (flag = 1)  
- **Transform rounds:** 6000  
- **Inner stream:** Salsa20 (for `Protected="True"` fields)

In a real‑world workflow we could attempt to validate master keys by simulating KeePass’s KDF (AES-ECB rounds) and checking the stream start bytes. **But here it’s unnecessary**—the dump already leaked the info we need.

---

## ✅ Final Answer

**`CBD{wh4t_15_Th1s_V4uLt_n1j1k4a_k3ePpass_8h17d1}`**

---

## 📝 Notes & Pitfalls

- Always check for **UTF‑16LE** strings on Windows dumps (`strings -el`). Plain ASCII often misses the good stuff.
- Expect **decoys** in CTFs. Look for structure/hints (e.g., “Part 1 58”) instead of jumping on the first flag‑looking string.
- KeePass often leaves rich clues: `Notes`, `Title`, `Password` (sometimes `Protected="True"` with Salsa20). Even if protected values are encrypted, surrounding metadata/notes can be enough.
- If you must attack the KeePass DB: parse header → derive key via transform rounds → AES‑CBC decrypt start → verify **StreamStartBytes**. But do it only if memory doesn’t already hand you the win.

---

## Appendix A — KDBX Header Snapshot (What we saw)

- **Magic:** KDBX 3.x  
- **Version:** `3.1`  
- **Cipher UUID:** `31c1f2e6bf714350be5805216afc5aff` (AES)  
- **Compression:** `1` (GZip)  
- **Transform rounds:** `6000`  
- **Inner stream ID:** `2` (Salsa20)  

This corroborates the KeePass traces we observed in RAM.

---

## Appendix B — One‑liners You Can Reuse

```bash
# 1) Quick UTF-16LE sweep for flags and KeePass fields
strings -el dump.dmp | grep -n -E "CBD\\{|Notes|Password|Protected|Title|Entry"

# 2) If you want ASCII too
strings dump.dmp | grep -n -E "CBD\\{|Notes|Password|Protected|Title|Entry"

# 3) Narrow down Base58 parts (look for “Part 1 58” / “Part 2 58”)
strings -el dump.dmp | grep -n -E "Part\\s*[12]\\s*58\\s*:"
```
