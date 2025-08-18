
# Can You Hear It? — Forensics (Author: Cyrus)

**Flag**: `CBD{tr4nsm1ss10n_c4rr13r_n01se_c2m96e}`

> *A single burst in the noise, can you hear it?*  
> This challenge hides a short **AFSK 1200 (Bell 202)** burst carrying a single **AX.25/APRS** UI frame. Demodulate → NRZI decode → **bit-unstuffing** → parse AX.25 → extract the plaintext flag from the Info field.

---

## Files
- `chall.wav` — mono, **48000 Hz**, duration **0.731 s**.

Check with:
```bash
file chall.wav
```

---

## TL;DR
- Spectrogram shows a dual-tone pair around **1200 Hz** (mark) and **2200 Hz** (space) → classic **AFSK1200**.
- Demodulated at **1200 baud** (48 kHz / 1200 = **40 samples/bit**).
- Found HDLC flag bytes `0x7E` (bit pattern `01111110`), then removed bit stuffing (drop `0` after every 5 consecutive `1`).
- Parsed **AX.25**: `APRS → CBDX01`, **Control 0x03 (UI)** and **PID 0xF0** (no L3).
- Info-field contains the flag: **`CBD{tr4nsm1ss10n_c4rr13r_n01se_c2m96e}`**.

---

## Approach A — Quick (multimon-ng)
Install and run:
```bash
sudo apt-get update && sudo apt-get install -y multimon-ng ffmpeg

# Decode AFSK1200 from wav
multimon-ng -a AFSK1200 -t wav chall.wav
```

You should see an APRS/AX.25 line; the **Info** part contains cleartext with the flag.

### Optional: Spectrogram
If `sox ... spectrogram -o spec.png` fails with “no handler for PNG”, use ffmpeg:
```bash
ffmpeg -i chall.wav -lavfi showspectrumpic=s=1920x1080:legend=disabled:scale=log assets/spec.png
```
(Commit `assets/spec.png` into the repo to display it in this README.)

---

## Approach B — Programmatic (Python)

### Install deps
```bash
python3 -m venv venv
source venv/bin/activate
pip install numpy scipy
```

### Minimal decoder script
Save as `decode_afsk_ax25.py` and run `python3 decode_afsk_ax25.py chall.wav`:

```python
#!/usr/bin/env python3
import sys, math
import numpy as np
from scipy.io import wavfile

def goertzel_power(block, freq, fs):
    n = len(block)
    k = int(0.5 + (n * freq) / fs)
    w = 2 * math.pi * k / n
    coeff = 2 * math.cos(w)
    s_prev = s_prev2 = 0.0
    for v in block:
        s = v + coeff * s_prev - s_prev2
        s_prev2 = s_prev
        s_prev = s
    return s_prev2**2 + s_prev**2 - coeff * s_prev * s_prev2

def bits_to_bytes_lsb(bits):
    out = []
    for i in range(0, len(bits)//8*8, 8):
        v = 0
        for j in range(8):
            if bits[i+j]: v |= (1<<j)   # LSB-first
        out.append(v)
    return bytes(out)

def parse_ax25_addresses(frame):
    addrs = []
    i = 0
    while i + 7 <= len(frame):
        blk = frame[i:i+7]
        callsign = ''.join(chr((b >> 1) & 0x7F) for b in blk[:6]).strip()
        ssid = (blk[6] >> 1) & 0x0F
        end  = blk[6] & 0x01
        addrs.append((callsign, ssid))
        i += 7
        if end == 1: break
    return addrs, i

def main(path):
    fs, data = wavfile.read(path)
    x = data.astype(np.float32)
    if x.ndim == 2: x = x.mean(axis=1)
    x /= (np.max(np.abs(x)) + 1e-12)

    f_mark, f_space, baud = 1200.0, 2200.0, 1200.0
    spb = int(round(fs/baud))   # 48k / 1200 = 40

    # Tone decision per bit (Goertzel at 1200 & 2200 Hz)
    tones = []
    for i in range(len(x)//spb):
        b = x[i*spb:(i+1)*spb]
        tones.append(0 if goertzel_power(b, f_mark, fs) > goertzel_power(b, f_space, fs) else 1)

    # NRZI decode: transition=0, no transition=1
    bits = []
    prev = tones[0]
    for t in tones[1:]:
        bits.append(0 if t != prev else 1)
        prev = t

    # Find HDLC flags (0x7E = 01111110), choose the largest gap as the frame
    s = ''.join('1' if b else '0' for b in bits)
    fp = '01111110'
    pos = [i for i in range(len(s)-7) if s[i:i+8] == fp]
    if len(pos) < 2: raise SystemExit("No HDLC frame found")
    start, end = max(zip(pos[:-1], pos[1:]), key=lambda ab: ab[1]-ab[0])

    # Bit-unstuffing (skip 0 after five 1s)
    ub = []
    ones = 0
    skip = False
    for b in bits[start+8:end]:
        if skip:
            skip = False; ones = 0; continue
        if b == 1:
            ones += 1; ub.append(1)
            if ones == 5: skip = True
        else:
            ones = 0; ub.append(0)

    frame = bits_to_bytes_lsb(ub)
    addrs, addr_len = parse_ax25_addresses(frame)

    ctrl = frame[addr_len] if addr_len < len(frame) else None
    pid  = frame[addr_len+1] if addr_len+1 < len(frame) else None
    info = frame[addr_len+2:] if addr_len+2 <= len(frame) else b""

    print("[+] Addresses:", addrs)
    print(f"[+] Ctrl/PID: 0x{{(ctrl or 0):02X}} 0x{{(pid or 0):02X}}")
    print("[+] Info (hex head):", info[:64].hex(), "...")

    # Extract flag
    i1 = info.find(b"CBD{{")
    if i1 != -1:
        i2 = info.find(b"}}", i1+4)
        if i2 != -1:
            print("[+] FLAG:", info[i1:i2+1].decode("latin1"))
            return
    print("[-] Flag not found; dump info above.")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: {{sys.argv[0]}} chall.wav")
        sys.exit(1)
    main(sys.argv[1])
```

Expected output (trimmed):
```
[+] Addresses: [('APRS', 0), ('CBDX01', 0)]
[+] Ctrl/PID: 0x03 0xF0
[+] FLAG: CBD{tr4nsm1ss10n_c4rr13r_n01se_c2m96e}
```

---

## Protocol Notes
- **AFSK1200 (Bell 202)**: mark=1200 Hz, space=2200 Hz, 1200 baud.
- **NRZI**: bit `0` = **transition**, bit `1` = **no transition**.
- **HDLC/AX.25**: frame fenced by byte flag `0x7E` (`01111110` bits).  
  Bit-stuffing → insert `0` after five consecutive `1`; remove during decode.
- **AX.25 fields**: addresses (7-byte blocks), **Control=0x03 (UI)**, **PID=0xF0**, then **Info** (contains plaintext flag).

---

## Pitfalls
- Missing **bit-unstuffing** or wrong **LSB-first** byte assembly → corrupted payload.
- Inverted **NRZI** mapping → unreadable text.
- Sampling/bit sync: 48k / 1200 = **40 samples/bit** simplifies slicing.

---

## Result
**Flag**: `CBD{tr4nsm1ss10n_c4rr13r_n01se_c2m96e}`

---

## Credits
- Challenge by **Cyrus**.
- Tools: `multimon-ng`, `ffmpeg` (optional), Python (`numpy`, `scipy`).
